---
title: "When Long-Stable Code Suddenly Starts Failing"
description: "A RocketMQ consumer started failing intermittently after a sharding rollout. Three reviewers — two human, one AI — missed the ThreadLocal leak."
date: 2026-05-12
draft: false
slug: "latent-bug-threadlocal-pollution"
categories:
    - Deep Dive
tags:
    - Java
    - Concurrency
    - Spring
    - RocketMQ
    - Troubleshooting
---

A colleague pulled me aside at 5pm on a Friday with two screenshots open in his IDE. Internal operations users had been reporting that some asynchronous workloads were occasionally failing. The consumer side of the RocketMQ message couldn't find the row the producer had just committed. Manual retry always succeeded. The code in question was something I'd originally written years back. Other engineers had touched it since, but the last meaningful change was more than six months earlier, well before the failures started.

We read the producer. We read the consumer. The code looked correct. He had already asked Claude Code to review the full path. Claude's review also concluded the code was correct.

Three reviewers, two human and one AI, all said: no bug here. But the bug was real. The RocketMQ DLQ was filling with messages that retried successfully on demand.

This post is about what we eventually found. But the bug itself isn't the interesting part. The *class* of bug is. The code was locally correct. It had been correct for years. It passed both human and AI review on the day it broke. It broke anyway, because something external to the code had changed.

I'll call this kind of defect a **latent bug**: a flaw whose harmlessness depends on invariants that nobody wrote down, that nobody knew were load-bearing, until somebody unknowingly broke them.

---

## TL;DR

1. A long-standing RocketMQ consumer started intermittently failing to find data the producer had just committed. Manual retry always recovered.
2. Both human and AI code review found nothing. The code, read in isolation, is correct.
3. Audit logs (enabled months earlier for an unrelated project) revealed the smoking gun: the producer wrote to `shard_main`; the failing consumer's SELECT was routed to `shard_alt_2`.
4. Root cause: `ThreadLocal` pollution. An AOP advice wrote the message's target shard into a per-thread `ShardContextHolder` but never cleared it. When a thread was reused for a message with no explicit shard, the residue from the previous alt-shard message routed the query wrong.
5. The bug had existed since day one. It was harmless until six months earlier, when tenant sharding shipped — producing the first messages that ever wrote to the holder, and breaking an unwritten invariant the missing `finally` had implicitly relied on.
6. Fix: `try/finally` with unconditional `clear()`. Defend at the entry-point boundary, not at each call site.

---

## The Failing Path

The application is a multi-tenant SaaS. As tenant data grew, a handful of high-volume tenants were migrated to dedicated database shards (`shard_alt_1`, `shard_alt_2`, and so on). The vast majority of tenants still live on `shard_main`. A routing library reads `ShardContextHolder.getCurrentShard()`, a `ThreadLocal<String>`, and uses it to pick the data source per query. When the holder is empty, queries route to `shard_main` by default.

A tenant's shard assignment is static: a tenant is either on `shard_main` or has been migrated to one of the `shard_alt_*` shards. The relevant convention for the bug story: *most messages don't carry an explicit shard, because most tenants are on the default shard*. Only messages for alt-shard tenants set the shard in their metadata.

The flow, end to end:

1. An internal admin endpoint hands the request to `JobIntakeService.submit(request)`.
2. Inside a `@Transactional` block, the service INSERTs a row into `ingest_jobs`. The producer's transaction is already bound to the tenant's correct shard via a separate routing context, so the row lands on the right shard.
3. Before returning, the service registers an `afterCommit` callback via `TransactionSynchronizationManager`.
4. On commit, the callback sends a RocketMQ message containing the row's ID, plus an optional `shard` field set only when the tenant is on an alt shard.
5. `JobWorker.onMessage` receives the message and calls `findById(jobId)` to fetch the row and start the actual work.

The failure mode: `findById` returns `null`, the consumer throws `"job not found"`, the message lands in DLQ. Manual replay always succeeds. The row is, in fact, present in the database, on the shard where the producer correctly stored it.

The producer code (simplified):

```java
@Transactional
public void submit(JobIntakeRequest request) {
    IngestJob job = new IngestJob();
    job.setTenantId(request.getTenantId());
    // ... populate other fields ...
    ingestJobRepo.save(job);

    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            JobNotification notification = new JobNotification(job.getId());
            notification.setShard(shardConfig.shardFor(request.getTenantId()));  // null for main-shard tenants
            mq.send(Topics.JOB_INTAKE, notification);
        }
    });
}
```

The consumer:

```java
@RocketMQMessageListener(topic = Topics.JOB_INTAKE, ...)
public class JobWorker implements RocketMQListener<JobNotification> {
    @Override
    public void onMessage(JobNotification notification) {
        // MyBatis-Plus repo: findById returns T or null, not Optional<T>
        IngestJob job = ingestJobRepo.findById(notification.getId());
        if (job == null) {
            log.error("job not found, id = {}", notification.getId());
            throw new RuntimeException("job not found");
        }
        // ... process ...
    }
}
```

There's no bug visible in either snippet. The bug isn't in this code.

---

## What Three Reviewers Missed

We worked through the obvious hypotheses. Each was easy to rule out, and every dismissal added to the mystery.

**Replication lag?** The natural first guess for "consumer can't find what producer just wrote." But the system uses a single MySQL primary per shard, no read replicas, no read/write splitting middleware. Ruled out.

**Snapshot isolation?** If the consumer was running inside a long-lived transaction, its `REPEATABLE READ` snapshot might predate the producer's commit. We checked: `findById` runs in autocommit mode, no surrounding `@Transactional`. Ruled out.

**Cache inconsistency?** No L2 cache on this entity, and MyBatis-Plus L1 cache is session-scoped, fresh session every call. Ruled out.

**RocketMQ delivery weirdness?** Async dispatch happens after the JDBC commit completes; the data is durable by the time the producer's MQ client invokes send. Ruled out.

**Nested transactions in the producer?** Was the `afterCommit` somehow firing before the actual commit? We traced the call graph end-to-end. No nesting, no `REQUIRES_NEW` shenanigans. The `afterCommit` fires exactly when Spring's `AbstractPlatformTransactionManager.processCommit()` says it should: after `doCommit()`, while the connection is still thread-bound. *(I'd written about this exact sequence in a previous post on `afterCommit` deadlocks.)*

So: the producer's `INSERT` is durable on its shard before the message ever leaves the producer. The consumer reads from the same shards the producer writes to. There is no caching, no replica, no snapshot.

And yet, occasionally, `SELECT * FROM ingest_jobs WHERE id = ?` returns zero rows for an ID the producer had just INSERTed seconds earlier.

By the time we'd exhausted the conventional explanations, we'd burned an hour. The code still looked correct. Claude's review still looked correct. The bug was somewhere we weren't looking.

---

## The Breakthrough: An Audit Log Enabled for an Unrelated Project

A few months earlier, for a completely separate effort (workload governance for our database tier, where I needed to aggregate SQL patterns across services), I had enabled the cloud provider's MySQL audit log feature. It captures every query against our production databases with timestamps, originating source, target shard, and the SQL text.

The audit log had nothing to do with this bug, conceptually. But it was running. I had access to it. So I went looking.

I pulled one failing case from the application logs and queried the audit log for any statement touching that row's job ID. Three matches came back:

| Timestamp     | Source          | Target shard      | SQL (abbreviated)                                | Rows |
|---------------|-----------------|-------------------|--------------------------------------------------|------|
| 16:49:34.000  | producer-pod    | `shard_main`      | `INSERT INTO ingest_jobs (id, ...) VALUES (...)` | 1    |
| 16:49:34.558  | consumer-pod-A  | **`shard_alt_2`** | `SELECT * FROM ingest_jobs WHERE id = ?`         | **0** |
| 16:52:57.467  | consumer-pod-B  | `shard_main`      | `SELECT * FROM ingest_jobs WHERE id = ?`         | 1    |

Three things jumped out:

1. The producer's INSERT correctly landed on `shard_main`, the tenant's actual shard.
2. The consumer's failing SELECT, 558 milliseconds later, was routed to `shard_alt_2`, a completely different shard used by a different set of tenants.
3. The successful manual retry, three minutes later, was routed to `shard_main`, the correct shard.

The application's source code didn't write `USE shard_alt_2` anywhere. Shard selection happened inside the routing library, which reads from the `ShardContextHolder` `ThreadLocal`. So the audit log was telling me, in five words: *the ThreadLocal had the wrong value*.

This wasn't a database visibility problem. It wasn't replication. It wasn't isolation. The application was sending the query to the wrong shard entirely.

The moment I saw the `shard_alt_2` row, I knew where to look. I'd hit a structurally identical bug years ago on a Tomcat thread pool: `ThreadLocal` state leaking across requests on a reused worker thread, and the shape of the failure was unmistakable.

---

## The Mechanism: Pollution in Reused Threads

The relevant `ThreadLocal` lives in a thin wrapper class:

```java
public class ShardContextHolder {
    private static final ThreadLocal<String> CURRENT_SHARD = new ThreadLocal<>();

    public static void setCurrentShard(String shard) { CURRENT_SHARD.set(shard); }
    public static String getCurrentShard()           { return CURRENT_SHARD.get(); }
    public static void clear()                       { CURRENT_SHARD.remove(); }
}
```

The routing library reads this on every query:

```java
public class ShardAwareDataSource implements DataSource {
    @Override
    public Connection getConnection() throws SQLException {
        String shard = ShardContextHolder.getCurrentShard();
        if (shard == null) {
            shard = "shard_main";  // default routing
        }
        return shardRegistry.get(shard).getConnection();
    }
    // ...
}
```

On the consumer side, all `@RocketMQMessageListener` methods are wrapped by an AOP advice that decides the shard context based on the incoming message's metadata:

```java
@Around("pointCut()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    JobNotification msg = (JobNotification) pjp.getArgs()[0];
    String shard = msg.getShard();

    if (shard != null) {
        ShardContextHolder.setCurrentShard(shard);
    }

    return pjp.proceed();
    // ❌ no try/finally. nothing clears.
}
```

Read that last comment again. There is no `finally`. The advice writes into the `ThreadLocal` and never removes what it wrote.

The mechanism doesn't require concurrency. A single-threaded executor processing the same mixed message stream sequentially would break in exactly the same way: alt-shard message A writes the holder; main-shard message B with `meta.shard == null` reads the residue. The bug's true precondition is **thread reuse across messages with different shard contexts**. Concurrency just raises the rate at which heterogeneous messages share a thread.

Spring's `@RocketMQMessageListener` runs on `DefaultRocketMQListenerContainer`'s `ConsumeMessageConcurrentlyService`, a `ThreadPoolExecutor` whose threads live for the lifetime of the application. By default (via the `rocketmq-spring` wrapper) the pool has 20 to 64 threads named `ConsumeMessageThread_<consumerGroup>_<idx>`. Each thread processes thousands of messages over its lifetime. The `ThreadLocal` lives with the thread, not the message.

The failure plays out like this:

1. **Message A arrives**, for a tenant on `shard_alt_2`. The advice sees `msg.shard = "shard_alt_2"`, calls `setCurrentShard("shard_alt_2")`. The consumer processes the message successfully. The advice returns. **Nothing clears.** The thread's `ShardContextHolder` still holds `"shard_alt_2"`.
2. **Message B arrives** on the same thread, for a tenant on `shard_main`. The producer leaves `shard` null for main-shard tenants (main is implicit; alt shards are explicit), so `msg.shard == null`. The advice sees `shard == null` and *doesn't write anything*. `proceed()` runs. The consumer calls `findById(jobId)`. The routing library reads `ShardContextHolder.getCurrentShard()`, sees `"shard_alt_2"` (the residue from message A), and sends the query to `shard_alt_2`. The row exists on `shard_main`. Zero results.

The query returns zero rows. The consumer throws "job not found." The message lands in DLQ.

When the operations user hits "retry," the replayed message lands on a different thread whose `ShardContextHolder` happens to be empty (or, less commonly, has been overwritten by an earlier main-shard-bound code path). The query routes correctly, the message succeeds. The failure looks transient. The data looks like it has a visibility delay. It doesn't. The first query asked the wrong shard.

---

## Why It Was Harmless, Why It Broke

The system's sharding model is **default plus override**: every tenant lives on `shard_main` unless explicitly migrated. The producer's metadata reflects this. Only alt-shard tenants set `notification.shard`. Main-shard tenants leave it null and rely on the routing library's default.

In isolation this convention is fine. `ShardAwareDataSource` correctly defaults to `shard_main` when the holder is empty. The bug isn't in the convention. It's in the implicit assumption that the holder *will* be empty when a main-shard message arrives. On a brand-new thread, or after an explicit clear, yes. On a long-lived worker that's been processing messages for hours, only if every previous message either cleared the holder (which the AOP didn't) or never wrote to it (which was true for years, before sharding shipped).

That last clause is the load-bearing one. Here's the unwritten invariant the original code depended on:

> *No message carries an explicit shard, so the `ThreadLocal` is never written, so the missing `finally` doesn't matter.*

The invariant had held since the AOP advice was written, years earlier. The `if (shard != null)` check was defensive code, written speculatively to support a sharding scheme that didn't yet exist. The condition never fired. Pollution was impossible because there was nothing to leak.

Six months ago, the team rolled out tenant sharding to handle data growth. The first migration was a single tenant. Over the following months, more migrated, one or two at a time as the data team identified candidates. Each migration meant some messages started carrying `meta.shard != null`. The dormant `if` branch finally executed, the `ThreadLocal` started getting non-null writes, and the missing `finally` became actively dangerous. The mismatch rate grew with the share of alt-shard traffic. When adoption was tiny the leak was invisible; when it crossed the threshold of observable failures, the alerts started.

Neither `JobWorker` nor the AOP advice changed in this window. What changed was the *shape of the message stream feeding them*, and that was enough to wake a bug that had been asleep for years.

(A parallel pollution problem exists on the producer side too: the producer's Tomcat threads accumulate stale `ShardContextHolder` state from previous requests. The INSERT happens to land on the right shard because the producer's transaction binds to a shard through a different code path. Anywhere `ThreadLocal` is the source of truth for routing, the same trap is waiting.)

The specifics here are tenant sharding. But the pattern recurs in any system that reads per-thread context to make decisions: logging MDC traceIds, Spring's `SecurityContextHolder`, outbound HTTP client auth headers, distributed tracing span context, A/B test bucketing, feature-flag scoping. If a `ThreadLocal` informs *any* downstream behavior, the same trap is waiting.

---

## A Minimal Reproduction

To convince myself the mechanism was real, I reproduced the bug in about 40 lines of pure JDK, no Spring, no RocketMQ:

```java
import java.util.concurrent.*;

public class ShardLeakDemo {
    static final ThreadLocal<String> CURRENT_SHARD = new ThreadLocal<>();

    record Message(long jobId, String metaShard, String actualShard) {}

    /** Models the routing library: returns the shard the next query will hit. */
    static String resolveTargetShard() {
        String current = CURRENT_SHARD.get();
        return current != null ? current : "shard_main";
    }

    static void handle(Message msg) {
        // Mimics the AOP advice's conditional set-without-finally pattern.
        if (msg.metaShard != null) {
            CURRENT_SHARD.set(msg.metaShard);
        }
        // No finally. Nothing clears.

        String resolved = resolveTargetShard();
        boolean leaked = msg.metaShard == null && !resolved.equals(msg.actualShard);
        System.out.printf("[%s] jobId=%d actualShard=%-12s resolved=%-12s%s%n",
            Thread.currentThread().getName(), msg.jobId,
            msg.actualShard, resolved,
            leaked ? "  <-- LEAK (resolved shard inherited from a previous message)" : "");
    }

    public static void main(String[] args) throws InterruptedException {
        ExecutorService pool = Executors.newSingleThreadExecutor();
        Message[] msgs = {
            new Message(1001, "shard_alt_1", "shard_alt_1"),  // alt-shard tenant, explicit
            new Message(1002, "shard_alt_2", "shard_alt_2"),  // alt-shard tenant, explicit
            new Message(1003, "shard_alt_1", "shard_alt_1"),  // alt-shard tenant, explicit
            new Message(2001, null,          "shard_main"),   // main-shard tenant, implicit
            new Message(2002, null,          "shard_main"),   // main-shard tenant, implicit
            new Message(2003, null,          "shard_main"),   // main-shard tenant, implicit
        };
        for (Message m : msgs) pool.submit(() -> handle(m));
        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

Run it. The first three messages each write `CURRENT_SHARD` to an alt shard. The last three, meant to operate on `shard_main`, write nothing, and the resolved shard inherits whichever alt shard the previous message on the same thread last wrote. Every main-shard message logs `LEAK`. The same mechanism manifests identically in any long-lived thread pool: bare `ThreadPoolExecutor`, Tomcat's `http-nio-*-exec-*`, RocketMQ's `ConsumeMessageThread_*`, Kafka consumer pools, `@Async` and `@Scheduled` pools. The framework is incidental.

---

## The Fix

```java
@Around("pointCut()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    JobNotification msg = (JobNotification) pjp.getArgs()[0];
    String shard = msg.getShard();

    if (shard != null) {
        ShardContextHolder.setCurrentShard(shard);
    }

    try {
        return pjp.proceed();
    } finally {
        ShardContextHolder.clear();   // ← the missing line
    }
}
```

Two questions worth answering about this fix.

**Why `clear()` unconditionally, even when the message had no explicit shard and we therefore wrote nothing?** Because we cannot trust that the body of `proceed()` didn't write the `ThreadLocal` for its own purposes (a downstream `@DS`-style annotation, a manual `setCurrentShard` inside the consumer body, a library that uses the same holder). Regardless of how the value got onto the thread, `clear()` returns the thread to a known-clean state on the way out. The next message starts from zero, just like a brand-new thread would. Anything weaker leaks state across messages, and we've just spent a hard incident learning what that costs.

**What about restoring the previous value, the way Spring's `RequestContextHolder` does with a save/restore?** Restoration is the right pattern when you have a sensible "outer" context to restore to: a child operation temporarily overriding its parent's context, for example. Consumer threads don't have an outer context. Their natural state between messages is *no context*, and `clear()` is what enforces that. If you find yourself reaching for save/restore here, you're probably modeling consumer messages as nested operations of some outer scope they don't actually have.

We made matching fixes for the producer-side leak (the HTTP filter now clears `ShardContextHolder` in `afterCompletion`, instead of relying on whatever happens to come next), and we audited every other thread-pool entry point in the codebase: `@Scheduled`, `@Async`, custom `ExecutorService`s, Feign async callbacks. Several had the same pattern.

---

## Why Static Review Missed It

When my colleague brought this bug to me, he, Claude, and I all read the same code and all concluded it was correct. We weren't being careless. We were doing exactly what code review asks of us: reading a function, tracing its logic, checking its edge cases.

The thing is, every single read of every single function was *correct*. The producer correctly INSERTs the row on the right shard. The consumer correctly queries by ID. The AOP advice correctly sets the shard for messages that carry one. The routing library correctly defaults to `shard_main` when the holder is empty. The `afterCommit` callback correctly fires after the commit. Read in isolation, every component does what it's supposed to do.

The defect lives in the space *between* components, in an unwritten contract about thread state that gets violated when two pieces of code that each look correct execute in sequence on the same long-lived thread. The bug isn't in the worker, the AOP advice, or the routing library. It's in the relationship: the worker depends on a `ThreadLocal` value that *some prior caller, on the same thread, possibly minutes earlier* was supposed to have left in a particular state.

This is why static review fails for this class. Review reads code. The bug isn't in any single piece of code. It's in the temporal contract across pieces of code, conditioned on the runtime behavior of a thread pool nobody is reading.

The audit log cracked the case because it captured *runtime* behavior, a specific query routed to a specific shard at a specific moment. Once you can see a query land on `shard_alt_2` when it should have hit `shard_main`, the rest of the inference is mechanical. No amount of staring at source code would have produced that observation. It had to come from the running system.

If a bug depends on cross-call thread state and an invariant that isn't written down, observability, not review, is your primary defense. Code review remains necessary, but it can't be the only line.

---

## Latent Bugs as a Class

The specific mechanism here is incidental. The underlying pattern recurs across many forms:

- A cache key collision that doesn't matter until two tenants are sharded onto the same node.
- A non-idempotent retry that works fine until a load balancer starts double-delivering.
- A foreign key constraint that holds until a backfill job runs out of order.
- A "this can never be null" assumption that holds until someone adds a new entry point that bypasses validation.

In every case, the code is locally correct. It becomes wrong because the environment shifts under it: the invariant it implicitly relied on stops holding. Tests don't catch this; the test environment doesn't reproduce the new condition. Review doesn't catch it; review reads code that is, in fact, correct under the *old* invariants.

Three things help, in increasing order of leverage:

1. **Write down the load-bearing invariants when you write the code.** A comment like `// assumes: no message carries an explicit shard yet, revisit if sharding ships` would have given whoever later rolled out sharding a chance to ask the right question.
2. **Defend at boundaries, not at each call site.** One `try/finally` in the AOP entry point neutralizes every downstream leak. Reviewing each caller for `ThreadLocal` hygiene doesn't scale; making the boundary trustworthy makes the call sites irrelevant.
3. **Invest in observability before you need it.** The audit logs I used to crack this case were running because of an unrelated project. That wasn't planning. Observability you have *before* the incident is worth orders of magnitude more than observability you scramble to add *during* one.

---

## Operational Checklist

If you operate any Java service that combines a long-lived thread pool with `ThreadLocal`-based context (shard routing, request-scoped auth, MDC for logging, distributed tracing span context, ...), here's the audit I would run before next Monday:

1. `grep -r "ThreadLocal" src/main/java`. List every `ThreadLocal` field and every wrapper holder (`*ContextHolder`, `MDC`, `TtlContext`, etc.).
2. For every `set` / `push` / `put`, find the matching `remove` / `clear`. Is it in the same lexical scope? Is it inside a `finally`? If either answer is no, that's a leak candidate.
3. List every long-lived thread-pool entry point: HTTP filters, `@RocketMQMessageListener`, Kafka listeners, `@Async`, `@Scheduled`, custom `ThreadPoolExecutor` workers, Feign async callbacks. Each must, at the entry-point boundary, *clear* all per-thread state on exit.
4. Prefer unconditional `clear()` at entry-point boundaries over "restore previous". Restoration is for nested contexts; thread-pool workers don't have an outer context, so the right reset is to nothing.
5. Add a guard at the top of each entry-point advice: if the `ThreadLocal` you're about to populate is *not* empty on entry, log a warning. This turns leak state into a noisy, debuggable signal long before it causes a production incident.
6. Audit any feature that introduces a new *category* of inbound traffic, or starts exercising a previously-dormant branch. Ask explicitly: *which invariants did the previous traffic pattern satisfy, and does my new category break any of them?* Write the invariants down in the same commit.

The code wasn't wrong. The contract it depended on had stopped holding. That contract, not the source, is what you have to keep current.

---

*References:*
- [Spring Framework Javadoc — `TransactionSynchronizationManager`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronizationManager.html)
- [Apache ShardingSphere](https://shardingsphere.apache.org/) — one representative tenant-aware sharding library. The mechanism described in this post applies to any router whose decision is read from per-thread state.
- [RocketMQ Spring — `DefaultRocketMQListenerContainer`](https://github.com/apache/rocketmq-spring)
- Companion post: [*When Does Spring Actually Release Your Database Connection?*](/p/spring-aftercommit-deadlock/) — the other half of "the bug is in the runtime, not the code" pair.

---

*A note on this post's code samples: identifiers, table and shard names, and some structural details have been abstracted from the actual incident for desensitization. The production code differs in its specific routing setup. The underlying mechanism (`ThreadLocal` pollution at a thread-pool entry-point boundary, woken by a feature that introduced a previously-dormant code path) and the timeline (the bug had been latent for years; the triggering feature shipped six months earlier; the failure rate climbed as adoption grew) are accurate to what happened in production.*
