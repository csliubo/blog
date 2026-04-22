---
title: "Why Is My Flink TaskManager Eating 8 GB on a 10K-Row UPDATE?"
description: "A Flink CDC OOM traced to two hidden source-side queues, BEFORE+AFTER duplication, and UTF-16 string bloat, plus the three config lines that tame it."
date: 2026-04-22
draft: false
slug: "flink-cdc-source-queue-oom"
categories:
    - Deep Dive
tags:
    - Flink
    - Flink CDC
    - Debezium
    - MySQL
    - OOM
    - JVM
    - Troubleshooting
---

> Measured: 2 GiB TaskManager survives a 10K-row UPDATE under a specific config, on a fat-row-rich slice where 13% of rows are 800 KiB+. Extrapolated via the Flink memory-model formula to a 12 GiB production TM. Every claim below is tagged measured vs extrapolated.

## TL;DR

A stock-config Flink CDC pipeline OOMs when one MySQL transaction is large and the UPDATE hits an id range clustered with fat rows. The root cause is two **independent** source-side queues: Debezium's `ChangeEventQueue` (default cap: 8192 events) and Flink's `FutureCompletingBlockingQueue` (default cap: 2 elements, each holding up to `max.batch.size` events). Together they can hold gigabytes of in-flight events. Each fat event also sits on heap at roughly `MySQL byte size × 2 (BEFORE+AFTER) × 2 (UTF-16 compact-string fallback)` — about 3.1 MiB for an 800 KiB JSON row.

**Fix** (three config lines, no code change):

```text
debezium.max.queue.size=100
debezium.max.batch.size=50
taskmanager.memory.managed.fraction=0.1
```

Measured at 2 GiB TM only; production numbers are formula-extrapolated. Uniform-distribution UPDATEs would be far lighter than the fat-rich slice used here.

## 0. The Setup

Our MySQL-to-Doris CDC pipeline (Flink CDC + flink-doris-connector) had a recurring problem: after a large UPDATE on one particular table, the downstream Flink TaskManager would OOM.

The table belongs to a **rich-text clinical record system**. Historical content had been stored as **inline base64-encoded images** inside a JSON column. A new tool had shipped to rewrite those blobs into object-storage links, but un-migrated document templates still produced the old inline form. A backfill UPDATE to convert legacy records to the new layout is what triggered this OOM.

**Fact check (confirmed post-mortem)**: production had **zero Debezium / Flink CDC tuning**. `max.batch.size=2048`, `max.queue.size=8192`, `managed.fraction=0.4` were all defaults. TM 12 GiB, JM 24 GiB.

**Production's emergency path** (context only; this investigation didn't drive it):

1. Immediately after the OOM, ops **removed the affected table from `include-tables`** so the rest of the tables kept syncing.
2. Then they retried the backfill in smaller batches:
   - Month-sized batches: still OOM or not draining.
   - 100 rows per UPDATE (app-side loops issuing `UPDATE ... WHERE id IN (...)`): still failed.
   - **50 rows per UPDATE: succeeded.**

Note that 50 and 100 are **rows per SQL statement at the application layer**, not Debezium's `max.batch.size`. Production's CDC config was never changed.

**Questions this investigation set out to answer**:

1. With stock defaults, why does a single large transaction OOM the TaskManager? Where is the root cause?
2. Is there a durable config change, **not requiring application-side batch splitting**, that lets the pipeline absorb large transactions on its own?

Spoiler: yes. See §5. But with capacity caveats.

## 1. Terminology: three layers of event size

Every "bytes per event" number in the rest of the post falls into one of these three tiers. I won't mix them.

| Layer | Symbol | Typical value | How measured | Meaning |
|---|---|---|---|---|
| MySQL binlog raw bytes | `R_binlog` | **~135 KiB/event** (non-fat slice); **~230 KiB/event** (fat-rich slice, §3.6) | §3.1 probe: 1000-row UPDATE → binlog POS +135 MiB | UTF-8 bytes inside the binlog file |
| Debezium `DataChangeEvent` on heap | `R_dbz` | **~230 KiB/event** (single heap-dump observation) | §3.4 MAT: Debezium `ChangeEventQueue` retained 23 MiB / 100 events | Target-table events only (`table.include.list` applied; §3.5) |
| Flink `SourceRecord` on heap | `R_flink` | **~970 KiB/event** (batch-avg on fat slice); **~14 MiB** for a single max-fat row | §3.4 MAT: `MySqlRecords` retained 97 MiB / 100 events | Target-table only, past the Debezium queue |

A small naming hazard worth knowing upfront: **`SourceRecord` (singular, Kafka Connect) is the event object**; **`SourceRecords` (plural, Flink CDC) is a batch container** wrapping many `SourceRecord`s. The MAT suspect ranking in §3.2 mixes these names, so keep the singular/plural distinction in mind when reading.

About the 4× gap between `R_dbz` (230 KiB) and `R_flink` (970 KiB): the two layers hold the same `SourceRecord` objects (Debezium's `DataChangeEvent` has just one `SourceRecord record` field, no other state), and both queues only contain target-table events. The 4× gap is most likely a single-snapshot timing artifact; §3.11.2 shows the supporting experiment. This detail doesn't affect the §5.2 capacity math, where the worst-case is computed against max-fat rows (14 MiB per event on both layers).

**Unit convention**: base-2 throughout (KiB / MiB / GiB). `information_schema.data_length` is also treated as base-2 per MySQL docs.

## 2. Environment and reproduction

### 2.1 Cluster and target table

- k8s + Flink Operator + FlinkDeployment CR. `namespace=cdc-dev`, standalone deployment named `cdc-oom-repro`.
- Target: `app_db.clinical_records`.
  - 254,832 rows (`SELECT COUNT(*)`, with a +17-row drift during our queries; normal dev-db churn).
  - `data_length = 16.2 GiB` (from `information_schema.tables`: clustered-index columns + InnoDB page overhead + fragmentation; excludes secondary indexes and off-page BLOBs).
  - `avg_row_length = 83 KiB` (same scope, all columns).
  - Summing only the five "large text columns" (`content_json` + four others) gives a weighted mean of **65 KiB/row**. The 18 KiB delta is the other ~22 columns + row overhead. Plausible.

### 2.2 Nacos config: scope down to one table

Three changes to the pipeline's Nacos `[mysql-1]` section:

```diff
- include-tables=.*
+ include-tables=clinical_records
- table-name=^app_db\.(?!undo_log|tmp).*$
+ table-name=^app_db\.clinical_records$
+ scan.startup.mode=latest-offset
```

### 2.3 FlinkDeployment diagnostic flags

```yaml
flinkConfiguration:
  kubernetes.operator.job.restart.failed: "false"   # keep OOM state; operator won't rebuild the pod
  env.java.opts.taskmanager: >-
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:HeapDumpPath=/tmp/dumps
    -XX:+ExitOnOutOfMemoryError
    -Xlog:gc*:file=/tmp/dumps/gc-%p.log:time,uptime,level,tags:filecount=5,filesize=20M

jobManager:           # fixed throughout; not varied
  resource:
    memory: "4g"
    cpu: 1

taskManager:
  resource:
    memory: "2g"    # the only knob I varied across runs
    cpu: 2
  podTemplate:
    spec:
      containers:
        - name: flink-main-container
          volumeMounts:
            - { name: dump-volume, mountPath: /tmp/dumps }
      volumes:
        - name: dump-volume
          persistentVolumeClaim: { claimName: oom-dump-pvc }   # 40Gi NFS-SSD
```

`kubernetes.operator.job.restart.failed: "false"` prevents operator-level pod rebuilds on failure. The PVC ensures the heap dump survives pod termination so I can pull it out later with a dump-reader pod plus `kubectl cp`.

### 2.4 Fixed workload (and a caveat about representativeness)

Every run: a 10,000-row UPDATE, `WHERE id BETWEEN 1870319408066945025 AND 1913089719906832385`, `SET modify_time = NOW()`.

The row-size distribution (§3.6) shows 4.1% of the whole table falls in the 500 KiB to 1 MiB "fat" bucket, but this particular id range is **13.3% fat**, 3× the table-wide concentration. **Conclusions here do not represent an arbitrary 10k UPDATE**. They hold only when the workload has this kind of fat-row clustering, which is plausible in production but not the common case.

### 2.5 A structural dev-vs-prod gap I can't close

My experiments run in a dev cluster with `include-tables` narrowed to the single target table. Debezium's `table.include.list` contains just that one table, and its queue is nearly empty before each burst.

Production is different:

- `include-tables=.*`. Debezium listens to tens to hundreds of tables.
- Real users continuously generate binlog events.
- Debezium's queue is never empty. At any moment it holds dozens to hundreds of events from other tables.
- A large transaction's commit burst layers on top of that existing queue, not onto empty.
- Average event size is smaller in multi-table mode (most tables are narrow), but a fat-table burst produces a heap mixture of many small events plus some fat events.

This gap is structurally unreproducible in dev: even if I set `include-tables=.*` in dev Nacos, the dev MySQL has no real users and negligible background traffic.

Implications:

- The upper-bound formulas below (`queue.size × R_per_event`, summed across layers) still apply. `queue=100` is a hard cap in any workload; the heap ceiling is bounded regardless of event mix.
- The exact "50 rows succeeds, 100 rows fails" threshold observed in production is not reproducible in dev. I don't have a data-level explanation. My guess is that background-queue depth plus fat-event arrival patterns interact, but this isn't verified.
- The proposed fix (`queue=100 + fraction=0.1`) holds by design for prod. Why "50 vs 100" specifically is the production threshold is out of scope here.

## 3. Investigation

### 3.1 Ruling out "partial flush during commit"

Early on I carried an assumption: "for a very large transaction, MySQL flushes binlog incrementally during execution." Reviewing my own reasoning I noticed this contradicts 2PC semantics, so I probed it:

```text
-- Poll SHOW MASTER STATUS once per second while a 1000-row UPDATE runs
14:31:57   POS=511,024,257    +2.8 MiB/s background
14:31:58   POS=513,860,063
14:31:59   POS=515,768,179
=== FIRE UPDATE @14:32:00.026 ===
14:32:00.616   POS=651,155,487   ← +135 MiB in 0.6s
=== UPDATE RETURNED @14:32:01.595 ===
14:32:02.056   POS=652,415,594   +1.2 MiB (back to background rate)
```

Binlog is flushed **at once during the 2PC commit phase**, before the client receives its ACK. Consistent with durability-before-ack semantics. "Incremental flush" was the wrong mental model.

(This probe yields an initial `R_binlog = 135 KiB/event`, but those 1000 rows fell in a non-fat id range. The 10k fat-rich slice gets recomputed in §3.6.)

Side observation: `SHOW BINARY LOGS` shows `mysql-bin.001440` reached 2.7 GiB, confirming that a big transaction can push a binlog file past `max_binlog_size` (MySQL 8.0 default 1 GiB). That 2.7 GiB is our 10k UPDATE transaction plus concurrent background writes — roughly 2.2 GiB for our transaction (slice big-field sum 1.1 GiB × 2 for BEFORE+AFTER, see §3.6) plus background traffic.

### 3.2 First reproduction: 2g + stock defaults

- 14:17:16 fire 10k UPDATE. InnoDB grinds through PK-ordered pages for ~5m45s (the latency is non-linear vs 1k UPDATE, see Appendix A).
- 14:23:02 MySQL commit.
- Immediately after commit, TM OOMs. Flink internally restarts it 8 times (taskmanager-1-1 through 1-8) in a replay death spiral until I manually suspend the job.

First heap dump: **453 MiB** (triggered by `-XX:+HeapDumpOnOutOfMemoryError`; the specific `OutOfMemoryError` subclass wasn't captured in logs). MAT Leak Suspects:

| Rank | Share | Object | Thread |
|---|---|---|---|
| 1 | **29%** / 126 MiB | `FutureCompletingBlockingQueue` | `Source Data Fetcher`, blocked at `FutureCompletingBlockingQueue.java:203 waitOnPut` |
| 2 | 18% / 75 MiB | `MySqlRecords` | same fetcher, mid-`put` |
| 3 | 17% / 73 MiB | `SourceRecords` | `Source: mysql-1` main thread, `SourceReaderBase.pollNext:160` |
| 4 | 11% / 46 MiB | `MySqlRecords` | main thread, `SourceReaderBase.pollNext:173` |

**Σ = 75%**. MAT's "related via common path" hint notes 1+2 and 3+4 share dominator paths, and MAT doesn't strictly deduplicate retained sizes, so the naive sum is qualitative only.

> "Flink self-restarts" above means Flink's internal `restart-strategy` (default `FixedDelayRestartBackoffTimeStrategy`, `maxAttempts=2,147,483,647`) respawning tasks — a different layer from the `kubernetes.operator.job.restart.failed: "false"` in §2.3. The operator flag prevents **pod/deployment-level** rebuilds; Flink's own scheduler can still restart tasks inside a living TM. This is also why the 4g run below "looked fine" while actually OOMing mid-drain.

The decisive finding: **75% of retention is on the Flink CDC source side**. Doris-sink objects (`DorisBatchStreamLoad`, `BatchRecordBuffer`) don't even appear in the top 10.

### 3.3 Heap ladder (2/4/8g measured; 12/16g formula-extrapolated)

`task.heap` is computed via the Flink 1.18 memory model: `flink.memory.process.size` minus metaspace/overhead/managed/network/framework. At the 2g tier the formula matches the TM startup log exactly (validated in §3.10).

```text
flink.size  = process.size - metaspace(256 MiB) - overhead(clamp(10% × process.size, 192, 1024))
managed     = fraction × flink.size
network     = max(10% × flink.size, 64)   # no upper cap in Flink 1.18 by default
task.heap   = flink.size - framework.heap(128) - framework.off(128) - managed - network
```

Network note: `taskmanager.memory.network.max` defaults to `MemorySize.MAX_VALUE` (no cap) in Flink 1.18.1. Only JVM-overhead is capped at 1 GiB. Earlier drafts of this post assumed network was capped at 1 GiB too, which was wrong.

| TM process | managed.fraction | managed | network | **task.heap** | 10k UPDATE result |
|---|---|---|---|---|---|
| 2 GiB | 0.4 (default) | 635 | 159 | **537 MiB** | ✗ OOM at commit (measured; formula matches TM log) |
| 4 GiB | 0.4 | 1,372 | 343 | **1,459 MiB** | ✗ silent OOM during drain (hprof 1.2 GiB), Flink self-restart masked it |
| 8 GiB | 0.4 | 2,847 | 712 | **3,302 MiB** | ✓ no OOM, drain 3m33s, container RSS peak 4,680 MiB (includes managed off-heap) |
| 12 GiB (prod) | 0.4 | 4,403 | 1,101 | **5,248 MiB ≈ 5.1 GiB** | not measured at 12g; formula-extrapolated |
| 16 GiB | 0.4 | 6,042 | 1,510 | **7,296 MiB ≈ 7.1 GiB** | not measured at 16g |

Visually:

```text
TM process    task.heap  (fraction=0.4 default)
  2 GiB         537 MiB  ▓░░░░░░░░░░░░░░░░░░░░░░  OOM at commit           ✗
  4 GiB       1,459 MiB  ▓▓▓▓░░░░░░░░░░░░░░░░░░░  silent OOM during drain ✗
  8 GiB       3,302 MiB  ▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░  no OOM                  ✓
 12 GiB       5,248 MiB  ▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░  formula-extrapolated
 16 GiB       7,296 MiB  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░  formula-extrapolated
```

The 4g tier is a trap. Monitoring showed the job "finished," but the PVC had a 1.2 GiB hprof sitting in it. The TM actually OOMed mid-drain and Flink's auto-restart masked the OOM. Always check the dump directory, not just the "is the job running" signal.

### 3.4 Hypothesis one: cap batch and queue to 100

From MAT Suspect #2: the default `MySqlRecords` batch averages 75 MiB. Intuition: "shrink the batch, problem solved."

I changed both knobs at the same time in a single Nacos update (an early draft of this post mentioned only `batch`, which was a slip):

```text
debezium.max.batch.size=100
debezium.max.queue.size=100
```

Still OOM (2g, hprof 446 MiB). MAT Top Consumers:

```text
MySqlRecords @0xda4da748        retained 97 MiB   ← one batch, 100 events
ChangeEventQueue @0xd977d4a0    retained 23 MiB   ← Debezium layer
```

So `R_flink = 97 MiB / 100 events ≈ 970 KiB/event retained`.

`ChangeEventQueue` dropped from its theoretical `8192 × R_dbz ≈ 1.8 GiB` upper bound down to 23 MiB, so `queue.size=100` did take effect (otherwise it'd still be climbing). But the Flink queue holds 2 × 97 MiB ≈ 194 MiB, plus the main thread holds another batch. Together, still overflowing the 537 MiB `task.heap`.

### 3.5 The key insight: two independent queues, two different capacity units

A critical question surfaced here, and only the code could answer it: **is `FutureCompletingBlockingQueue` sized by `max.batch.size`, or is it independent?**

From Apache Flink CDC `release-3.2.1`:

`MySqlSource.java:167` — the Flink-side queue is constructed with the no-arg form:

```java
FutureCompletingBlockingQueue<RecordsWithSplitIds<SourceRecords>> elementsQueue =
        new FutureCompletingBlockingQueue<>();   // no-arg → default capacity
```

Flink 1.18.1 `FutureCompletingBlockingQueue.java:109`:

```java
public FutureCompletingBlockingQueue() {
    this(SourceReaderOptions.ELEMENT_QUEUE_CAPACITY.defaultValue());
}
```

`SourceReaderOptions.java:36-40`: `ELEMENT_QUEUE_CAPACITY` defaults to **2 elements** (config key `source.reader.element.queue.capacity`).

`FutureCompletingBlockingQueue.java:193` — `put()` bounds by element count, not bytes:

```java
while (queue.size() >= capacity) {   // ★ counts elements, not bytes
    ...
    waitOnPut(threadIndex);
}
```

`BinlogSplitReader.java:147-162` — one important detail:

```java
public Iterator<SourceRecords> pollSplitRecords() {
    final List<SourceRecord> sourceRecords = new ArrayList<>();
    if (currentTaskRunning) {
        List<DataChangeEvent> batch = queue.poll();   // ← Debezium ChangeEventQueue.poll()
        for (DataChangeEvent event : batch) {
            if (shouldEmit(event.getRecord())) {      // ★ filter happens AFTER poll
                sourceRecords.add(event.getRecord());
            }
        }
        ...
    }
}
```

`StatefulTaskContext.java:139-151` — Debezium's queue is built with:

```java
new ChangeEventQueue.Builder<DataChangeEvent>()
        .maxBatchSize(connectorConfig.getMaxBatchSize())
        .maxQueueSize(queueSize)
        .maxQueueSizeInBytes(connectorConfig.getMaxQueueSizeInBytes())  // default 0 = no byte cap
        .build();
```

Debezium `ChangeEventQueue.poll()` (v1.9.8.Final, simplified — actual impl also honors `maxQueueSizeInBytes` and a `pollInterval` timeout):

```java
public List<T> poll() {
    List<T> records = new ArrayList<>();
    ...
    while (!queue.isEmpty() && records.size() < maxBatchSize) {
        records.add(queue.poll());
    }
    return records;
}
```

`poll()` returns up to `maxBatchSize` events; it stops when the queue is empty. With `queue.size=100` and `batch.size=2048`, `poll()` takes at most 100 (capped by current queue depth); no exception.

**Risk note**: Debezium's `CommonConnectorConfig.java:344` has a `validateMaxQueueSize` assertion:

```java
if (maxQueueSize <= maxBatchSize) {
    problems.accept(field, maxQueueSize, "Must be larger than the maximum batch size");
}
```

My `queue=100 + batch=2048` (default) configuration **violates** this invariant. It ran fine in experiment 5 (§4) without complaint. I haven't chased down why (whether Flink CDC's init path skips this validator, or whether it's non-fatal in this version). But Debezium's docs and code both expect `queue > batch`. A safer production config sets `batch=50` so the invariant holds; see §5.

#### Two independent queues, two independent caps

Both queues hold only target-table events. `table.include.list` is threaded from Nacos `table-name` through `MysqlDatabaseSync.buildCdcSource()` into Debezium (`MySqlSourceConfigFactory.java:340-341`).

The structure:

```text
┌─ Flink TaskManager (task.heap) ────────────────────────────────────────┐
│                                                                        │
│  Debezium reader thread                                                │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Debezium ChangeEventQueue                                        │  │
│  │   capacity = max.queue.size EVENTS (default 8192)                │  │
│  │   holds DataChangeEvent (thin wrapper around SourceRecord)       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                    │                                                   │
│                    │ poll() returns up to max.batch.size events        │
│                    │          (default 2048)                           │
│                    ▼                                                   │
│  BinlogSplitReader.pollSplitRecords():                                 │
│     for each event: if shouldEmit(e) add to sourceRecords              │
│     wrap as one MySqlRecords                                           │
│                    │                                                   │
│                    │ put(), blocks when queue.size() >= capacity       │
│                    ▼                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Flink FutureCompletingBlockingQueue                              │  │
│  │   capacity = 2 ELEMENTS (source.reader.element.queue.capacity)   │  │
│  │   each element = one MySqlRecords (up to max.batch.size events)  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                    │                                                   │
│                    │ SourceReaderBase.pollNext()                       │
│                    ▼                                                   │
│  Main source thread: holds 1 in-flight batch                           │
│                    │                                                   │
│                    │ filter, serialize, forward                        │
│                    ▼                                                   │
│                ... sink writer                                         │
│                                                                        │
│  ▲ SIMULTANEOUSLY ALIVE in heap, at worst case:                        │
│      Debezium queue             :  max.queue.size × R_per_event        │
│   +  Flink queue (capacity = 2) :  2 × max.batch.size × R_per_event    │
│   +  Main-thread in-flight batch:      max.batch.size × R_per_event    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

Numerically, at 12g prod with fat-rich workload:

| Position | Holds | Upper-bound formula | Default-config estimate |
|---|---|---|---|
| Debezium `ChangeEventQueue` | `DataChangeEvent` wrapping `SourceRecord` (target table only) | `max.queue.size × R_per_event` | batch-avg: 8192 × 230 KiB ≈ **1.8 GiB**. Pathological: 8192 × 14 MiB ≈ 110 GiB. |
| Flink `FutureCompletingBlockingQueue` (cap=2) | `MySqlRecords` wrapping `SourceRecords` (target table only) | `2 × max.batch.size × R_per_event` | batch-avg: 2 × 2048 × 970 KiB ≈ **4 GiB**. Pathological: 2 × 2048 × 14 MiB ≈ 56 GiB. |
| Main-thread in-flight batch | same | `max.batch.size × R_per_event` | batch-avg: 2048 × 970 KiB ≈ **2 GiB**. Pathological: 2048 × 14 MiB ≈ 28 GiB. |

"batch-avg" is the fat-rich-slice MAT observation from §3.4; "pathological" is the §5.2 assumption that every event is a max-fat row. Both columns are target-table only; the difference is the event-size assumption.

All three theoretically stack on the same `task.heap`, but in steady state they sit well below their caps. Runtime event flow is throttled by backpressure; the main thread keeps consuming so the Flink queue usually holds just 1 batch; the Debezium queue stays far below 8192 on average. All three simultaneously near their caps is the extreme instant of "main thread can't drain while Debezium is still filling." Not steady state.

`max.batch.size` alone is insufficient. Shrinking only it trims the lower two rows; the upstream Debezium queue at default `queue.size=8192` can still accumulate ~1.8 GiB (batch-avg, fat-rich workload). Production never changed `max.batch.size`, but the duality matters: even if someone tried "just turn the batch down" as a fix in the future, it would still fail because the upstream queue remains unbounded in event count.

### 3.6 Row-size long tail and slice selectivity

Whole-table bucket distribution (`SELECT ... GROUP BY bucket`, column = sum of five large text columns = `content_json + 4 others`):

```text
| bucket        | rows    | avg_KiB | cumulative_KiB |
| a <10 KiB     | 57,597  |   5.3   |       305,264  |
| b 10-50 KiB   | 180,973 |  38.8   |     7,021,752  |   ← 71%, the main mode
| c 50-100 KiB  |   5,318 |  56.8   |       302,062  |
| d 100-500 KiB |     337 | 219.3   |        73,926  |
| e 500 KiB-1 MiB| 10,426 | 798.4   |     8,324,118  |   ← 4.1% but 51% of total bytes
| f 1-5 MiB     |     198 |1,672.9  |       331,234  |   ← 0.08%, max = 3.47 MiB
| total         | 254,849 |         |    16,358,356  |   ≈ 15.97 GiB
```

Weighted mean: **65 KiB/row** across the five big columns. Compared to `avg_row_length = 83 KiB` (all columns + page overhead), the 18 KiB delta is reasonable.

**Bimodal distribution**: 71% is normal 10-50 KiB business rows; 4% is 500 KiB+ fat rows. The 83 KiB average is a mean driven by outliers. Don't let it fool you.

Slice-specific distribution (`id BETWEEN 1870319408066945025 AND 1913089719906832385`, a snowflake-ID range corresponding to ~4 months of business data):

```text
| a <10 KiB        |  5,170 |   5.3   |
| b 10-50 KiB      |  3,468 |  13.5   |
| c 50-100 KiB     |     29 |  65.9   |
| d 100-500 KiB    |      0 |    -    |
| e 500 KiB-1 MiB  |  1,332 | 801.9   |   ← 13.3%, 3× the table-wide concentration
| f 1-5 MiB        |      1 |1,518.6  |
| total 10,000 rows, 1,145,780 KiB ≈ 1.1 GiB big-field bytes |
```

Fat ratio in this slice is 3× the whole-table rate. Id range and fatness are correlated here. Why the correlation exists at the business layer isn't verified; it could be that a certain category of records was predominantly created during that time window, or it could be a snowflake-id-to-business coincidence. **Every OOM conclusion below assumes production will encounter similarly fat-rich UPDATEs**. Uniformly distributed UPDATEs would be significantly lighter.

Weighted binlog bytes per row for this slice: `1.1 GiB × 2 (BEFORE+AFTER, binlog_row_image=FULL) / 10000 rows ≈ 230 KiB/row`. (Small columns and binlog event header overhead are <10%, ignored here.) This matches the 2.7 GiB binlog file observation from §3.1 (2.2 GiB our transaction + 0.5 GiB background).

### 3.7 Hypothesis two: where the heap bloat comes from

Pausing to review the numbers, an instinct flagged something: Java object-header overhead shouldn't be on the order of 7×. The header is tens of bytes, and a hundred-KiB string shouldn't balloon that much. That question pushed me back to pin down the actual source of the bloat. Spoiler: it's not headers; it's two independent 2× factors stacking.

Arthas measurements (`com.taobao.arthas 4.1.8`, injected via `jattach` into a JRE-only container):

Verify runtime config is actually applied:

```text
> vmtool --action getInstances --className io.debezium.connector.base.ChangeEventQueue \
    --limit 2 --express 'instances[0].maxBatchSize + "|" + instances[0].maxQueueSize'
@String[100|100]   ✓
```

Verify schema is shared (rule out "each record carries its own Schema"):

```text
> vmtool ... ConnectSchema --limit 1 --express 'instances.length'
@String[ConnectSchema_count=1]   ✓ one global instance
```

Drill into a fat row:

```text
> UPDATE ... WHERE id = 1886996929695858689   (single-column content_json = 801 KiB UTF-8)

> vmtool ... SourceRecord ... --express 'instances[0].value.get("before").get("content_json").length()'
@String[before.content_json.length=817292]   ← 817,292 chars

> vmtool ... SourceRecord ... --express 'instances[0].value.get("after").get("content_json").length()'
@String[after.content_json.length=817292]
```

Note: the §3.6 "e" bucket average of 798 KiB is a sum across five columns (`content_json` + four others). The 801 KiB here is `content_json` alone. They're close on this row only because the other four columns are nearly empty on fat rows; `content_json` dominates. The "800 KiB" used in the breakdown below refers to the measured `content_json` value.

Analysis:

- MySQL `content_json` is 801 KiB (UTF-8 bytes).
- Java `String.length()` = 817,292 chars.
- The JSON contains non-Latin-1 characters (Chinese clinical text embedded in the values), so JDK's compact-string optimization falls back to UTF-16 encoding. The `char[]` is 2 bytes × 817,292 chars ≈ 1.6 MiB on heap.

Bloat chain (this behavior requires English keys plus some non-Latin-1 values; pure-ASCII wouldn't trigger UTF-16 fallback, and pure-Chinese text actually shrinks by ~0.67× going UTF-8 → UTF-16):

```text
   MySQL column `content_json`            801 KiB (UTF-8 bytes)
                     │
                     │ Debezium parses the binlog row into Java
                     ▼
   Java String (coder=1, UTF-16)        ≈ 1.6 MiB
                     │                    (char[] × 2 bytes;
                     │                     triggered by any non-Latin-1 char)
                     │
                     │ binlog_row_image=FULL → Envelope.before + Envelope.after
                     ▼
        BEFORE String    ≈ 1.6 MiB
     +  AFTER  String    ≈ 1.6 MiB
     ──────────────────────────────
     One fat event on heap ≈ 3.1 MiB
```

Step-by-step evidence:

| Step | Number | Evidence |
|---|---|---|
| MySQL `content_json` UTF-8 bytes | 801 KiB | Direct: `SELECT LENGTH(content_json)` |
| Corresponding Java `String.length()` | 813,965 to 817,292 chars (varies per row) | Direct: arthas `instances[0].value.get("before").get("content_json").length()` |
| Java `String.coder` | `coder=1` (UTF-16) | Direct (added for this writeup): arthas `instances[0].value.get("before").get("content_json").coder`. BEFORE and AFTER both return 1. |
| Java String internal `byte[]` length | 1,626,476 bytes ≈ 1.55 MiB | Direct: arthas `value.length`. Matches `chars × 2` to a few bytes (alignment/padding). |
| BEFORE + AFTER are two independent Strings | × 2 ≈ 3.1 MiB | Code fact: `binlog_row_image=FULL` carries both images; Debezium's `Envelope` stores `before` and `after` as independent Struct fields. |
| Struct / Schema / HashMap fixed overhead | negligible (~KB) | Arthas direct: ConnectSchema global `instance_count=1`; Struct internals are `Object[]`. |
| **Single fat event on heap, total estimate** | **≈ 3.1 MiB** | Sum of the above. Every key step is directly measured (`coder=1`, `byte[].length`). |

Cross-check against §3.4's MAT observation (`97 MiB / 100 events = 970 KiB/event`): with 30% fat + 70% normal composition (§3.8 derives this), the per-event mean comes to `0.30 × 3.1 MiB + 0.70 × 0.05 MiB = 0.965 MiB ≈ 988 KiB`. Off from 970 KiB by 18 KiB, about 1.9%. The bloat chain closes.

The "Java-headers-shouldn't-be-7×" instinct was right. The real bloat sources are BEFORE+AFTER and UTF-16, two independent 2× factors stacked, not object-header overhead. (Both factors are table-shape-dependent.)

One more thing: BEFORE lives on the heap through the entire source → queue → main-thread pipeline. The sink-side default `ignoreUpdateBefore=true` (`DorisExecutionOptions.java:291`) does **not** help upstream heap occupancy. It only controls whether `JsonDebeziumDataChange.extractUpdate` writes BEFORE into the Doris stream-load body. **`ignoreUpdateBefore` cannot rescue a source-side OOM.**

### 3.8 Back-solving the 97 MiB `MySqlRecords` batch (Flink layer)

The fat concentration inside one batch can easily exceed the slice's table-wide 13%. Binlog writes events in PK order, and fat rows cluster on adjacent ids, so a fat cluster's worth of them can land in a single batch. Back-solving from the MAT retained:

```text
R_flink = 97 MiB / 100 events = 970 KiB/event (batch-avg, retained)
```

Estimate with fat event = 3.1 MiB (from §3.7's direct measurement) and normal event ≈ 0.05 MiB (slice bucket-b average = 13.5 KiB × 4× BEFORE+AFTER+UTF-16 bloat ≈ 54 KiB ≈ 0.05 MiB):

```text
p × 3.1 MiB + (1 - p) × 0.05 MiB = 0.97 MiB
→ p ≈ 30%
```

About 30% of the batch is fat rows (about 30 out of 100), vs the slice's overall 13%. A plausible-but-unverified explanation: InnoDB executes UPDATEs in PK order, and binlog writes in PK order too, so if fat rows cluster in id space, individual batches can have higher fat concentration than the slice average. I didn't directly measure event id-vs-size correlation; this is a consistency argument, not proof.

### 3.9 Live-queue backpressure (arthas)

With `2g + queue=100 + batch=100`, I re-ran 10k UPDATE and polled both queues every 3 to 6 seconds (sampling cadence misses some instantaneous peaks, but 5 consecutive `FlinkQ=2/2` observations are sufficient evidence of saturation):

```text
15:52:30 DebeziumQ=0/100   FlinkQ=0/2      pre-commit
15:52:35 DebeziumQ=47/100  FlinkQ=2/2 (full) burst arrives
15:52:40 DebeziumQ=1/100   FlinkQ=2/2      fetcher blocked on put
15:52:52 DebeziumQ=11/100  FlinkQ=2/2
15:53:02 DebeziumQ=26/100  FlinkQ=2/2
15:53:18 DebeziumQ=77/100  FlinkQ=2/2
15:53:24 arthas SIGKILL (exit 137)  ← TM OOM
15:53:28 pod NotFound
```

Frame before OOM: `2×100 (Flink, SourceRecords inside MySqlRecords) + 100 (main thread, SourceRecords) + 77 (Debezium, DataChangeEvent) ≈ 377 event objects alive simultaneously` (300 SourceRecord + 77 DataChangeEvent; all point to SourceRecord objects, since DataChangeEvent is a thin wrapper). Backpressure is working exactly as designed. **The container heap is simply too small.** (§3.10 shows `managed.fraction=0.1` as a way to reclaim heap without adding RAM.)

### 3.10 Final validation: `managed.fraction=0.1` (with caveats on single-diff coverage)

Flink 1.18 defaults `taskmanager.memory.managed.fraction=0.4`, reserving 40% of `flink.size` for RocksDB. The pipeline uses very little state (binlog offset + small sink state, well under 1 GiB). That 40% is recoverable.

At 2g, plug into the formula:

```text
flink.size      = 2048 - 256(metaspace) - 205(overhead)        = 1587 MiB
fraction=0.4: managed=635, network=159, task.heap = 1587 - 128 - 128 - 635 - 159 = 537 MiB
fraction=0.1: managed=159, network=159, task.heap = 1587 - 128 - 128 - 159 - 159 = 1014 MiB
```

TM startup log confirms:

```text
taskmanager.memory.managed.size=166429984b    ≈ 159 MiB   ✓
taskmanager.memory.task.heap.size=1063004400b ≈ 1014 MiB  ✓
-Xmx1197222128                                ≈ 1142 MiB = task.heap + framework.heap
```

`task.heap` goes from 537 MiB to 1014 MiB, a **+89%** gain.

Experiment 5 config: `2g TM + debezium.max.queue.size=100 + taskmanager.memory.managed.fraction=0.1 + batch.size=2048 (default, capped by queue=100 at runtime)`.

Result:

```text
16:35:54 MySQL commit (10k rows)
16:36:18 TM mem 1263 → 1670 MiB (+407),  CPU 1046m → 1579m
16:36:34 CPU 2000m (2 cores saturated),  mem 1659 MiB
~3 min period CPU stays at ~2000m,  mem stable 1660-1704 MiB
16:39:36 CPU drops to 1020m,  mem 1683 MiB   ← processing complete, 3m42s
TM stays Running for the following 12 min
```

Doris-side consistency: `SELECT COUNT(*) WHERE id IN range AND modify_time >= '2026-04-21 16:35:00'` returns **10,000/10,000**.

Methodological caveat (caught in post-hoc review): the transition from experiment 4 to experiment 5 changed two variables (`batch=100 → 2048` + `fraction=0.4 → 0.1`). No single-variable control was run. Reasoning about each alone:

- `fraction=0.1` alone (leave `queue.size=8192` default): `task.heap` becomes 1014 MiB (formula-matched with TM log). But Debezium queue's theoretical upper bound is `max.queue.size × R_per_event = 8192 × 230 KiB ≈ 1.8 GiB`, already exceeding 1014 MiB `task.heap`. Whether it actually OOMs depends on whether runtime backpressure lets the queue near that bound. Not measured. "Theoretical cap > available heap" is not a safe plan.
- `queue=100` alone (leave `fraction=0.4`): experiment 4 (`2g + queue=100 + batch=100 + fraction=0.4`) measured OOM. `task.heap` is only 537 MiB, and MAT data (§3.4) shows Flink queue's two batches + main thread's one batch already eat ~260 MiB + JVM baseline.

So the provable conclusion: at the 2g tier, experiment 5's combination passed, experiment 4 (queue=100 alone) failed. "Both diffs are necessary" is an inference from the experiment 5 vs 4 differential. "Is `fraction=0.1` alone enough?" was not directly tested. The 12g/16g extrapolations are formula + assumption, not measured.

### 3.11 Follow-up validations

Two follow-up experiments run after the main story ended. Both are confidence-builders, not new conclusions.

#### 3.11.1 Control: non-fat slice (10k UPDATE on bucket 4)

To confirm the fat-rich slice is a necessary condition for OOM, I ran the same config (`2g + queue=100 + fraction=0.1`) against a non-fat slice.

First, an id-range distribution scan (table split into 10 id buckets):

```text
| bucket | row_cnt  | fat_pct | min_id               |
| 0,1    | ~1.7k    | 0.0%    | 1838... 1879...      |
| 2      |   3,144  | 24.1%   |                      |  ← densest fat region
| 3      |  10,807  |  9.3%   |                      |  ← §2.4's experiment slice is in here
| 4      | 162,929  |  0.9%   | 1921... 1942...      | ← table body, almost no fat
| 5      |  13,200  | 13.7%   |                      |
| 6-9    |  ~63k    | 7-12%   |                      |
```

Workload: bucket 4, `id BETWEEN 1921717482026164226 AND 1942249500063379458 LIMIT 10000` (hit 10,000 exactly).

| Metric | §3.10 fat-rich slice (13.3%) | **Non-fat bucket 4 (0.9%)** |
|---|---|---|
| UPDATE InnoDB runtime | 5m45s | **43 seconds** (small rows, fast page updates) |
| Drain CPU saturation duration | ~3 min (2000m) | ~2 min (2000m) |
| TM container memory delta | +407 MiB (1263 → 1670) | **+20 MiB** (1726 → 1746) |
| Drain completion | commit + ~3m42s | commit + ~2 min |

Delta ratio is ~20×, same order of magnitude as per-event heap occupancy (fat 3.1 MiB / normal ~0.08 MiB ≈ 40×). The gap vs 40× is because Flink queue cap = 100 events also throttles in-flight count; drain rate and sink throughput are basically the same, only the event size differs.

Baseline note: the fat-slice run started at 1263 MiB; the bucket-4 run started at 1726 MiB. TM wasn't restarted between the two, so old gen had accumulated retained objects. The two baselines aren't fully independent. The 20× delta conclusion still holds: the accumulated retained is a shared baseline drift, not something that distorts the delta comparison. Strict apples-to-apples would require a TM restart between runs; not done here.

Implication: §5.2's "5.6 GiB worst-case upper bound" is a pure theoretical ceiling. Real production heap usage is dictated by whether the UPDATE hits a fat-rich id range. Hitting fat pushes into GB territory; hitting normal rows is nearly a no-op.

#### 3.11.2 Simultaneous two-queue snapshot

To stress-test §1's "timing-sampling-bias" explanation of the 4× `R_dbz`-vs-`R_flink` gap, I ran a 2000-row UPDATE on bucket 2 (24% fat), with 8 rapid arthas snapshots during drain:

```text
probe  SR_total  DebeziumQ   FlinkQ   MySqlRecords  DataChangeEvent
1      172       1/100       2/2      3             3
2       48       1/100       2/2      4             4
3       57       1/100       2/2      4             19
4       88      21/100       2/2      4             36
5      124      16/100       2/2      4             18
6      168       0/100       2/2      3             50
7      186      13/100       2/2      4             16
8      224      51/100       2/2      4             53
```

Observations:

- FlinkQ is persistently full at 2/2 (backpressure is stable).
- DebeziumQ oscillates between 0 and 51 (fetcher tops it up, main thread drains it).
- SR_total oscillates between 48 and 224, reflecting two-queue + main-thread batch dynamics.
- The two layers, at any given instant, hold different events. There's no apples-to-apples comparison of per-event retained across them.

These probes measured queue **depths** (event counts) only, not per-event size or per-queue fat ratio. The "composition is out of sync" language below is an interpretation of depth oscillation, not a direct measurement.

Revisiting §1's claim: the original OOM hprof's 230 vs 970 KiB gap is most consistently explained by timing-sampling bias. The two queues' depths oscillate independently during drain, so it's reasonable to infer their event **composition** also drifts (otherwise the pipeline would ebb and flow as one unit). No mechanical evidence of a per-layer difference (code confirms identical object shape).

Cross-scenario caveat: §3.11.2 was on bucket 2 (24% fat, 2000 rows); the original OOM hprof was on bucket 3 (13% fat, 10k rows). I didn't redo multi-snapshot probes on the original OOM scenario. The conclusion is extrapolation assuming two-queue fluctuation mechanics don't depend on fat density or event count.

## 4. Experiments recap

| # | TM | fraction | batch | queue | Result | Heap peak (task.heap limit) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 2g | 0.4 | 2048 | 8192 | ✗ OOM | peak ~1,156 MiB; hprof 453 MiB | Flink self-restarted 8× |
| 2 | 4g | 0.4 | 2048 | 8192 | ✗ mid-drain OOM + self-restart | hprof 1.2 GiB | "looks like it finished," but PVC has a dump |
| 3 | 8g | 0.4 | 2048 | 8192 | ✓ no OOM | 4,680 MiB | drain 3m33s |
| 4 | 2g | 0.4 | **100** | **100** | ✗ OOM | hprof 446 MiB; Debezium q 23 MiB; MySqlRecords 97 MiB | both queue/batch confirmed active |
| **5** | **2g** | **0.1** | 2048 (default) | **100** | **✓ no OOM** | **1,670 MiB** stable | drain 3m42s; the recommended config |
| 6 | 12g | not measured | not measured | not measured | not measured | task.heap 5.1 GiB (default) / 8.3 GiB (fraction=0.1) | formula-extrapolated only; see §5 |
| 7 | 16g | not measured | not measured | not measured | not measured | task.heap 7.1 / 11.5 GiB | formula-extrapolated only |

"N/A heap peak" means the TM OOMed before that time and the displayed value is the last monitoring sample, not "wasn't sampled."

## 5. Production recommendations

### Quick pick

| If your current TM is | Apply | Expected behavior |
|---|---|---|
| **16 GiB** (e.g. already expanded during an incident) | the 3 config lines from the TL;DR | zero additional hardware, full safety margin against max-fat pathological |
| **12 GiB** (original prod spec) | the 3 config lines from the TL;DR | safe for fat-average workload; tight against max-fat pathological (see §5.1.1 for full-margin option) |
| <12 GiB | run an 8g-scale regression first; this post's numbers don't guarantee safety below 12 GiB | see §3.3 heap ladder |

Before rolling to prod, run the full combo in dev against the same fat-row UPDATE workload and verify no OOM plus Doris consistency.

The rest of §5 explains *why these knobs*, *which scenarios they cover*, and *where the safety margin breaks* — skip to §5.2 if you care about worst-case capacity math.

### 5.0 Scope of applicability

All estimates below only hold for fat-rich UPDATE workloads (per §2.4 + §3.6: 13% fat in the slice vs 4% table-wide). Uniformly distributed UPDATEs have far lower heap occupancy. Other burst shapes outside the "fat-rich failure mode" (such as high-concurrency DDL, snapshot phase) have different mechanisms and aren't covered here.

### 5.1 Minimum diff (handles fat-average; tight on max-fat pathological)

Keep TM at 12 GiB (no new hardware); change only two configs:

**Nacos `[mysql-1]` section**:

```text
debezium.max.queue.size=100
debezium.max.batch.size=50     ← satisfies the queue > batch invariant
```

**FlinkDeployment YAML `flinkConfiguration`**:

```yaml
taskmanager.memory.managed.fraction: "0.1"
```

**What this covers**: fat-average workloads (the batch-avg 970 KiB/event level observed at MAT in §3.4). Per §5.2's math, `task.heap = 8.3 GiB`, fat-avg in-flight steady state ≈ 310 MiB, nominal utilization ~3.7%. Comfortable.

**Where it's tight**: max-fat pathological (every in-flight event is a 3.47 MiB max row × 2 × 2 = 14 MiB). §5.2 gives in-flight ≈ 5.6 GiB / 8.3 GiB task.heap = 67% nominal. But with a G1 empirical-working-set cap of ~70%, effective capacity is ~5.8 GiB, utilization hits ~97%, safety margin just 3%. Probabilistically unlikely to hit (max row is 0.08% of the table; 400 in a row is astronomical), but very little error budget if it did.

About `batch=50`:

- Purpose: satisfy Debezium `validateMaxQueueSize`'s `queue > batch` invariant (§3.5).
- Throughput: `ChangeEventQueue.poll()` only waits `poll.interval.ms` (default 500 ms) when the queue is empty. During burst the queue is non-empty, so `poll()` returns immediately; batch size has negligible effect on steady-state throughput. The main overhead is that each `MySqlRecords` holds fewer events, so the main thread's `pollNext()` is invoked roughly 2× as often; minor overhead.
- `batch=50` itself wasn't directly measured in dev. Experiment 5 ran with `queue=100 + batch=2048(default) + fraction=0.1`. If touching `batch` feels risky, an alternative is `batch=100 + queue=200` (also satisfies the invariant); also not measured.
- Before rolling to prod, do a full-combo regression in dev: run the full `queue=100 + batch=50 + fraction=0.1` combo on the same fat-rich 10k UPDATE and verify no OOM + Doris consistency. Don't verify `batch=50` alone; verify the whole target config.

### 5.1.1 Complete recommendation (covers max-fat pathological too)

If max-fat clustering is a real concern (extreme but not negligible), on top of the minimum diff also bump TM to 16 GiB:

```text
TM: 16 GiB   ← original prod was 12g; if 16g already from an emergency expansion, no change needed
plus the two config diffs from §5.1
```

Cost depends on current prod state:

- If prod is still at the original 12 GiB TM (per §0): +4 GiB = +33% TM memory.
- If prod was already bumped to 16 GiB during the emergency (one version of the FlinkDeployment YAML we saw had `memory: "16g"`): zero additional hardware, just two config lines.

At 16 GiB + `fraction=0.1`, `task.heap ≈ 11.5 GiB` (§3.3 formula; not measured at 16g):

- Pathological 5.6 GiB / 11.5 GiB ≈ 49% nominal utilization.
- G1 70%-working-set view: 5.6 / (11.5 × 0.7) = 5.6 / 8.05 ≈ 70%. Comfortable vs 12g's ~97%.

Trade-off:

- **Minimum diff (12g + 2 config lines)**: zero new hardware, two lines, safe for fat-avg, tight for max-fat. Good when you're willing to accept "max-fat pathological is improbable" and want to return TM to its original size.
- **Complete recommendation (16g + 2 config lines)**: +4 GiB (+33%) vs 12g, or zero additional if already expanded during the incident. Covers the max-fat envelope with G1 room to spare. Good for conservative deployments or "since we already expanded, let's keep it."

Experiments only went up to `2g + 2 config lines` (fat-avg passed, max-fat not validated); 12g and 16g are formula-extrapolated. Before production, run a full-config regression in dev or staging either way.

### 5.2 12g prod capacity extrapolation (formula-extrapolated, not measured)

At 12g + `fraction=0.1`, `task.heap ≈ 8.3 GiB` (§3.3 formula: flink.size 11008 − framework 256 − managed 1101 − network 1101 ≈ 8550 MiB).

```text
Debezium queue heap upper bound = 100 × R_per_event(230 KiB)    ≈  23 MiB   ← MAT batch-avg
Flink queue heap upper bound    = 2 × 100 × R_per_event(970 KiB) ≈ 190 MiB  ← MAT batch-avg
Main-thread batch upper bound   =     100 × R_per_event(970 KiB) ≈  97 MiB
──────────────────────────────────────────────────────────────
In-flight steady state (fat-rich slice, §3.4 MAT)            ≈  310 MiB
```

Normal-distribution UPDATE steady state (not measured; extrapolated from §3.6 whole-table avg): using bucket-b `avg_KiB=38.8` × 4× bloat (BEFORE+AFTER × UTF-16) ≈ 155 KiB/event, `400 in-flight events × 155 KiB ≈ 60 MiB`. About 5× lighter than fat-rich steady state.

(Note on "normal" baseline drift across sections: §3.8 uses 0.05 MiB/event (slice bucket-b, 13.5 KiB × 4); §3.11.1 uses ~0.08 MiB/event (bucket-4 estimate); §5.2 here uses 0.15 MiB/event (whole-table bucket-b, 38.8 KiB × 4). Each is internally consistent for its own slice, but the numbers aren't interchangeable across sections.)

In-flight pathological worst case (assume all 400 in-flight events are max rows):

```text
  4 × 100 × 14 MiB = 5.6 GiB
  ↑ 14 MiB/event comes from §3.7's decomposition:
    3.47 MiB = SQL max for this table; ×2 BEFORE+AFTER is a binlog_row_image=FULL code fact;
    ×2 UTF-16 is arthas-confirmed via String.coder=1 + byte[].length directly.
  "400 consecutive max rows" is a probabilistically-impossible envelope assumption
  (max row = 0.08% of the table). Treat this number as an upper-bound envelope, not an expectation.
```

At 12g + `fraction=0.1`, `task.heap ≈ 8.3 GiB`:

- Fat steady state 310 MiB / 8.3 GiB ≈ 3.6% nominal.
- Pathological worst 5.6 GiB / 8.3 GiB ≈ 67% nominal.

**What "nominal" means**: `task.heap` isn't entirely available for in-flight events. The leftover `task.heap - in-flight` also has to serve:

- G1 young + survivor + old-gen working set (Flink framework resident objects, RocksDB in-heap part, Flink Pekko actor, serde intermediate objects, and so on).
- Main thread's per-batch temporaries (JSON serialization, Doris HTTP stream-load body construction).
- GC headroom (G1 starts concurrent mark around 70–80% heap utilization; beyond that you get mixed GC, then Full GC, and eventually promotion failures).

The ~70% effective-working-set target for G1 is empirical (common rule-of-thumb among Java backend teams, not something I measured here). Folding that in:

- Effective capacity ≈ 8.3 GiB × 0.7 ≈ 5.8 GiB.
- 5.6 GiB / 5.8 GiB ≈ 97% utilization, 3% margin. Tight.

Or, using a looser 80% working-set assumption (8.3 × 0.8 = 6.64 GiB), 5.6 / 6.64 ≈ 84%, 16% margin.

So: the "35% safety margin" I quoted in early drafts was nominal utilization of whole `task.heap`, which is optimistic vs G1's real working-set envelope. A genuine "70% utilization, 30% margin" promise to production requires the 16g upgrade in §5.1.1. The 12g config only makes sense if you're betting max-fat pathological won't happen.

### 5.3 Keep JobManager reasonable (empirical, not load-tested)

Production's JM at 24 GiB is over-provisioned:

- The job has 1 MySqlSource + 1 filter + 1 sink writer + 1 committer. The ExecutionGraph is tiny.
- Throughout all experiments JM was fixed at 4 GiB (§2.3). Via `kubectl top pod` I observed JM peak RSS floated around 800 to 900 MiB (estimated heap usage ~200 to 300 MiB of the `-Xmx ≈ 3.2 GiB`; Pekko, Netty, metrics reporters account for the rest of RSS). The observation uses `kubectl top`, not proper JMX instrumentation, so precision is limited.
- 4 GiB is plenty.

I didn't specifically burst-test JM memory. If the job later picks up more tables or larger checkpoint state, re-evaluate.

### 5.4 Untested alternative: `binlog_row_image=MINIMAL`

In principle, switching MySQL's `binlog_row_image` from FULL to MINIMAL would make BEFORE carry only PK + changed columns, roughly halving heap occupancy. But:

- It's a server-level global variable affecting all binlog consumers.
- On a managed cloud RDS it's risky to change without testing.
- I haven't verified that the pipeline's semantics stay correct under MINIMAL. Not changing it for this iteration.

Worth validating if a future need arises and MySQL is yours to tune.

## 6. Source references (pinned to tags)

Apache Flink CDC `release-3.2.1`:

- `flink-cdc-connect/flink-cdc-source-connectors/flink-connector-mysql-cdc/src/main/java/org/apache/flink/cdc/connectors/mysql/source/MySqlSource.java:167`
- `.../cdc/connectors/mysql/source/reader/MySqlSplitReader.java:105`
- `.../cdc/connectors/mysql/debezium/reader/BinlogSplitReader.java:147`
- `.../cdc/connectors/mysql/debezium/task/context/StatefulTaskContext.java:139`
- `flink-cdc-connect/flink-cdc-source-connectors/flink-connector-debezium/src/main/java/org/apache/flink/cdc/debezium/table/DebeziumOptions.java:25`

Apache Flink `release-1.18.1`:

- `flink-connectors/flink-connector-base/src/main/java/org/apache/flink/connector/base/source/reader/synchronization/FutureCompletingBlockingQueue.java:109`
- `.../source/reader/SourceReaderOptions.java:36`

Debezium `v1.9.8.Final`:

- `debezium-core/src/main/java/io/debezium/config/CommonConnectorConfig.java:302` (`DEFAULT_MAX_QUEUE_SIZE = 8192`, `DEFAULT_MAX_BATCH_SIZE = 2048`)
- `debezium-core/src/main/java/io/debezium/config/CommonConnectorConfig.java:344` (`validateMaxQueueSize`)
- `debezium-core/src/main/java/io/debezium/connector/base/ChangeEventQueue.java` (`poll()` impl)

Our fork of doris-flink-connector (based on apache/doris-flink-connector):

- `flink-doris-connector/src/main/java/org/apache/doris/flink/tools/cdc/mysql/MysqlDatabaseSync.java:215-231` (`debezium.*` passthrough)
- `flink-doris-connector/src/main/java/org/apache/doris/flink/cfg/DorisExecutionOptions.java:291` (`ignoreUpdateBefore = true`)
- `flink-doris-connector/src/main/java/org/apache/doris/flink/sink/writer/serializer/jsondebezium/JsonDebeziumDataChange.java:89-131` (op dispatch)

## 7. Tooling used

| Situation | Tool | Usage |
|---|---|---|
| Heap snapshot | JVM `-XX:+HeapDumpOnOutOfMemoryError` | Write to a PVC-mounted `/tmp/dumps`; survives pod termination |
| Heap analysis (offline) | Eclipse MAT 1.15.0 (`20231206-linux.gtk.x86_64`) | `ParseHeapDump.sh heap.hprof org.eclipse.mat.api:suspects` |
| Top components | MAT | `org.eclipse.mat.api:top_components` |
| Live instance count | arthas 4.1.8 `vmtool --action getInstances` | `--express 'instances.length'` |
| Live instance fields | arthas `vmtool ... --express` (OGNL) | `instances[0].value.get("before").get("col").length()` |
| Shaded class path | arthas `sc *Keyword` | |
| JRE-only container attach | `jattach` (apangin/jattach v2.2) | Doesn't need JDK tools.jar |
| Exfiltrate large heap dump from k8s | PVC + dump-reader pod + `kubectl cp` | `emptyDir` dies with the pod |
| MySQL binlog observation | dedicated CDC user + `SHOW MASTER STATUS` polling | Verify commit timing |

## 8. Takeaways

Six general debugging habits worth keeping:

1. Build a controllable reproduction first. PVC-backed dumps, operator auto-restart off. But remember that Flink's internal restart-strategy can still mask mid-drain OOM, so check the dump directory directly.
2. MAT Leak Suspects is the starting point. The retained ranking tells you "who owns the memory" directly; no need to guess first.
3. For multi-layer pipelines (CDC / Debezium), line up MAT thread names and stack frames with source. Faster than class-name search.
4. Arthas live-object checks are the fastest way to verify configs and counts at runtime.
5. Reserved-subsystem memory is often tunable. Flink's `managed.fraction=0.4` is wasted on low-state jobs.
6. Skim your own reasoning for assumptions that contradict known semantics (§3.1's "partial flush during commit" error was exactly that kind of self-caught mistake; cheap to verify, embarrassing to miss).

Debezium / Flink CDC specifically:

- Two independent queues, both must be sized. Debezium `max.queue.size` (upstream, holds DataChangeEvent) and Flink `source.reader.element.queue.capacity` (downstream, holds SourceRecord). Defaults: 8192 and 2 respectively. The latter is capped by element count, not bytes.
- Heap size is not binlog bytes. Distinguish `R_binlog` (on-disk), `R_dbz` (Debezium queue), `R_flink` (Flink queue).
- `avg_row_length` is misleading for bimodal distributions. Bucket the table and look at the long tail.
- `ignoreUpdateBefore=true` does not help source-side OOM. It only controls whether the sink writes BEFORE; upstream memory is unchanged.
- Respect the `queue > batch` invariant (Debezium `validateMaxQueueSize`) even if nothing visibly complains when it's violated.

---

*All measurements were collected on 2026-04-21 in a dev cluster. The 12g/16g `task.heap` numbers come from the Flink 1.18 memory-model formula; those tiers were not measured for OOM. The experimental slice is fat-rich (13% vs 4% table-wide), so extrapolating conclusions to other id ranges requires care.*

## Appendix A: UPDATE latency non-linearity (side observation)

- §3.1's 1000-row UPDATE took 1.6 seconds.
- §3.2's 10k-row UPDATE took 5 minutes 45 seconds.

10× rows, 216× latency. Not fully explained. Possible factors:

1. Slice difference. The 1k UPDATE used ids `1913… → 1914…` (narrow range, non-fat-rich), while the 10k used `1870… → 1913…` (wider range, fat-rich).
2. Buffer-pool state (the two runs were hours apart).
3. Fat rows live on larger InnoDB pages (off-page BLOB storage), so writes take more I/O.

I don't have enough data to decompose these. Tagged as "measured anomaly, partial explanation."

## Appendix B: How this post was written

The investigation was run by me in a dev cluster, paired with Claude Code on the tooling and analysis loop. No teammate reviewed or ran the experiments. The `kubectl top`/`arthas`/MAT commands and their outputs are my own; the conclusions are mine. The production hotfix (app-side batch splitting down to 50 rows per SQL) was a separate, parallel track by the ops team, not part of what's written up here. Mentions of "we" in the post mean me + the tooling I was driving.
