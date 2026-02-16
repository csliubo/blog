---
title: "When Does Spring Actually Release Your Database Connection?"
description: "A production deadlock revealed that Spring holds database connections long after commit — and that switching from REQUIRES_NEW to REQUIRED doesn't fix the problem, it makes it worse"
date: 2026-02-10
draft: false
slug: "spring-aftercommit-deadlock"
categories:
    - Deep Dive
tags:
    - Java
    - Spring
    - Database
    - Troubleshooting
---

One morning, the admin dashboard wouldn't load. Health probes passed, the JVM was fine, but every business request hung indefinitely. Both pods behind the load balancer were stuck — no one could get through.

Our first instinct was to restart both pods. Get the service back, stop the bleeding. So we did — and that was a mistake. We should have preserved the scene first. A thread dump, a connection pool snapshot, anything. Once you restart, the evidence is gone.

We got lucky. After the restart, one pod recovered but the other went right back to the same stuck state. That gave us a second chance — a healthy pod and a stuck pod to compare side by side.

---

## TL;DR

1. The outage was connection pool starvation, not monitor lock contention: many threads were WAITING for DB connections while `BLOCKED = 0`.
2. In Spring JDBC transactions, commit happens before connection release; `AFTER_COMMIT` callbacks run while the original connection is still thread-bound.
3. `@TransactionalEventListener(AFTER_COMMIT)` + `@Transactional(REQUIRES_NEW)` can starve the pool under concurrency because each thread may need a second connection.
4. Switching to `REQUIRED` avoids the second connection but is unsafe in `afterCompletion`; Spring 6.1 blocks this misconfiguration at startup.
5. A practical fix is thread isolation (`@Async`) plus explicit transaction boundaries (`TransactionTemplate`), with non-`CallerRunsPolicy` rejection and metric-based verification after rollout.

---

## The Thread Dumps

We took jstack from both pods. The difference was immediate:

| Metric | Stuck pod | Healthy pod |
|--------|-----------|-------------|
| Total threads | 225 | 179 |
| Tomcat HTTP threads | 51 (49 stuck) | 10 (all idle) |
| MQ consumer threads | 10 (all stuck) | 10 (all idle) |
| Threads waiting for DB connection | **61** | **0** |
| BLOCKED threads | 0 | 0 |

**BLOCKED = 0** ruled out lock contention. This wasn't threads fighting over a monitor — it was resource exhaustion. 61 threads all WAITING on Druid's connection pool `notEmpty` condition, meaning every connection was checked out and nobody was returning one.

---

## The Smoking Gun

MQ consumer thread stacks showed two distinct patterns:

**Pattern A** — Waiting at the entry point (straightforward — just no connections available):
```
DruidDataSource.pollLast()
DataSourceTransactionManager.doBegin()
TaskExecutorHandler.executeTask()
```

**Pattern B** — Waiting inside an AFTER_COMMIT callback:
```
DruidDataSource.pollLast()                 <- waiting for connection #2
DataSourceTransactionManager.doBegin()
TaskAppService.onTaskCreateEvent()          <- REQUIRES_NEW
triggerAfterCompletion()                    <- AFTER_COMMIT callback
TaskExecutorHandler.executeTask()           <- still holding connection #1
```

Pattern B was the smoking gun. A single thread holding connection #1 (from `executeTask`), executing an `AFTER_COMMIT` callback that requests connection #2 via `REQUIRES_NEW` — and blocking because the pool is empty.

Multiply by `maxActive` threads, and you have connection pool starvation — a resource-level deadlock. With unbounded (or very long) `maxWait`, this behaves like a permanent hang. With finite `maxWait`, it fails by timeout instead of hanging forever. No thread is BLOCKED on a monitor (hence `BLOCKED = 0` in the thread dump), but every thread holds one connection and waits for another that nobody can release. The JVM looks healthy, but the application is frozen.

But this raised a question: **why is the original connection still held?** The transaction was already committed. Shouldn't the connection be back in the pool?

---

## Why the Connection Is Still Held

It turns out Spring doesn't release connections at `commit()`. It releases them later, in `cleanupAfterCompletion()`. Here's the actual structure of `AbstractPlatformTransactionManager.processCommit()`:

*(Scope and assumptions for this analysis: Spring Boot 3.3.5 / Spring Framework 6.1.x, `DataSourceTransactionManager` with JDBC connections, Druid pool with bounded `maxActive`, and `maxWait` configured long enough that waiters accumulate. With short `maxWait`, you'll typically see timeout errors instead of an application-wide hang. JTA transaction managers and R2DBC reactive connections have different lifecycle behavior.)*

```java
// AbstractPlatformTransactionManager.processCommit() — simplified
try {
    triggerBeforeCommit(status);
    triggerBeforeCompletion(status);
    doCommit(status);                          // JDBC commit

    try {
        triggerAfterCommit(status);            // afterCommit() callbacks
    } finally {
        triggerAfterCompletion(status,         // AFTER_COMMIT listeners
            STATUS_COMMITTED);                 // fire here
    }
} finally {
    cleanupAfterCompletion(status);            // connection released here
}
```

The lifecycle as a timeline:

```
commit() execution flow              Connection state             Extension point
----------------------               ----------------             ---------------
triggerBeforeCommit()                Active, in transaction       beforeCommit()
triggerBeforeCompletion()            Active, in transaction       beforeCompletion()
doCommit()                           JDBC commit                  --
triggerAfterCommit()                 Committed, connection        afterCommit()
                                     STILL bound to thread
triggerAfterCompletion(COMMITTED)    Committed, connection        AFTER_COMMIT
                                     STILL bound to thread        listeners
cleanupAfterCompletion()             Connection unbound,          --
                                     returned to pool
method returns                       Connection back in pool
```

`@TransactionalEventListener(phase = AFTER_COMMIT)` callbacks fire during `triggerAfterCompletion()` — internally, they're registered as `TransactionSynchronization` implementations whose `afterCompletion(STATUS_COMMITTED)` method invokes the listener. The connection is not released until `cleanupAfterCompletion()` in the outer finally block.

**The critical window**: between `doCommit()` and `cleanupAfterCompletion()`, the transaction is committed but the connection has not been returned to the pool. Any code that requests a new connection during this window competes with the unreleased one.

---

## The Code That Triggered It

The pattern below uses an order scenario to illustrate; the actual service dealt with task dispatching (as shown in the thread dumps above), but the structure is identical:

```java
@Transactional
public void createOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request)); // holds connection #1
    eventPublisher.publishEvent(new OrderCreatedEvent(order.getId()));
}
// connection #1 is NOT returned until cleanupAfterCompletion()
```

```java
// "Send confirmation only after the order is safely committed"
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void onOrderCreated(OrderCreatedEvent event) {
    notificationRepository.save(                           // needs connection #2
        new Notification(event.orderId(), "Order confirmed"));
    messageSender.send(ORDER_TOPIC, event.orderId());
}
```

The intent is reasonable: don't send a confirmation for an order that might roll back. But REQUIRES_NEW opens a second connection while the first is still held.

Under concurrency:

```
Thread A                              Connection Pool (maxActive = N)
--------                              --------------------------------
1. Begin TX, get connection C1        Pool: N-1 available
2. orderRepository.save(...)
3. publishEvent(...)
4. commit()                           C1 still NOT returned
5. AFTER_COMMIT fires
   +-- onOrderCreated()
      +-- REQUIRES_NEW                Request C2 ... but pool is empty
         +-- pollLast() blocks        <- all N threads stuck here
```

When concurrent threads >= `maxActive`, this becomes connection pool starvation. With long waits it looks like a permanent deadlock; with bounded waits it degrades into timeout failures.

This is the same mechanism described in [Spring Issue #26250](https://github.com/spring-projects/spring-framework/issues/26250), where `REQUIRES_NEW` inside a suspended transaction exhausts the connection pool. The Spring team closed it as "by design," suggesting developers increase pool size or use bulkhead patterns. Our scenario is a tighter variant: the `AFTER_COMMIT` callback means every thread that enters the commit path will attempt to acquire a second connection — so the starvation threshold is simply concurrent threads >= `maxActive` (with outcome determined by `maxWait`: hang-like behavior vs timeout failures).

### Stabilizing first, fixing later

Once we understood the deadlock mechanism, the immediate priority was to stop it from happening again. We didn't jump straight to a code fix — changing transaction behavior under pressure is how you introduce the next incident.

Instead, we increased Druid's `maxActive` as a temporary mitigation. The math was straightforward: each MQ consumer thread could hold one connection while its `AFTER_COMMIT` callback requests a second. So we counted the MQ consumers per pod, doubled that number (worst case: every consumer hits the callback simultaneously), added a buffer for Tomcat HTTP threads, and set `maxActive` accordingly.

This bought us time. The starvation requires concurrent threads >= `maxActive`, so a larger pool raises the threshold high enough to avoid it under normal load. It's not a fix — it's a band-aid that shifts the breaking point (or timeout point, depending on `maxWait`). But it stabilized the service while we dug deeper into the Spring internals.

---

## What About REQUIRED?

At this point you might think: "just switch from `REQUIRES_NEW` to `REQUIRED` — it'll join the existing transaction and reuse the connection, avoiding the deadlock."

We tested this. REQUIRED does not deadlock. But what it does is worse.

**Important caveat:** REQUIRED propagation works correctly during normal transaction lifecycle — when the outer transaction is still active, a participant joining via REQUIRED gets proper commit/rollback semantics. The problem described below is specific to the `afterCompletion` phase, where the outer transaction has already committed but Spring's transaction state hasn't been fully cleaned up.

### REQUIRED reuses the connection — no deadlock

During `afterCompletion`, `isExistingTransaction()` returns `true`. The `ConnectionHolder` still has `transactionActive = true` — that flag isn't cleared until `cleanupAfterCompletion()`. So when REQUIRED checks for an existing transaction, it finds one and "participates" through `handleExistingTransaction()` — no `doBegin()`, no new connection, no deadlock.

We verified this by subclassing `DataSourceTransactionManager` to log every `doBegin()` call. With REQUIRED, the listener reused the same connection (identical `hashCode`). With REQUIRES_NEW, it got a different one.

### The zombie transaction

The problem is what REQUIRED "joins." The transaction is already committed. The connection has `autoCommit = false` (set by the outer `doBegin()`), but there's no real transaction backing it. You're operating in a zombie state.

**Data commits through a side effect.** When `cleanupAfterCompletion()` runs, it restores `autoCommit` to `true`. Most JDBC drivers treat `setAutoCommit(true)` as an implicit commit — so any pending writes get committed as a byproduct of connection cleanup.

**Rollback doesn't work.** Spring transactions have two roles: the **initiator** (`newTransaction = true`, created by `doBegin()`) and the **participant** (`newTransaction = false`, joined via `handleExistingTransaction()`). This is the same model you see in everyday Spring code — when a `@Service` method with `@Transactional(REQUIRED)` calls another `@Transactional(REQUIRED)` method, the inner method is a participant, not an initiator. Only the initiator actually calls `doCommit()` and `doRollback()`. A participant's `commit()` is a no-op, and its `rollback()` only sets a `rollbackOnly` flag — the real rollback is delegated to the initiator.

In the `AFTER_COMMIT` phase, REQUIRED becomes a participant of a transaction whose initiator has already committed and moved on. So when the participant tries to rollback, it sets `rollbackOnly` — but nobody is listening. Then `cleanupAfterCompletion()` restores `autoCommit` → implicit commit → the data persists despite the exception.

We verified this too:

```java
// REQUIRED in afterCompletion: INSERT, then throw
TransactionTemplate tt = new TransactionTemplate(txManager);
tt.setPropagationBehavior(PROPAGATION_REQUIRED);
tt.executeWithoutResult(status -> {
    jdbcTemplate.update("INSERT INTO event_log(...)");
    throw new RuntimeException("should trigger rollback");
});
// Result: row persisted. Rollback had no effect.
```

The INSERT survives the exception. You think you have transaction protection, but you don't.

### Spring 6.1 blocks this combination

Starting with Spring Framework 6.1, this misconfiguration is rejected at startup:

```
IllegalStateException: @TransactionalEventListener method must not be annotated
with @Transactional unless when declared as REQUIRES_NEW or NOT_SUPPORTED
```

This validation was introduced in [Spring Issue #30679](https://github.com/spring-projects/spring-framework/issues/30679), raised by Oliver Drotbohm (Spring Data maintainer). The rationale: transactional event listeners execute during the cleanup phase in an **undefined transactional state**. Only `REQUIRES_NEW` creates an independent transaction with correct commit/rollback semantics. (Spring 6.1.3+ also allows `NOT_SUPPORTED`, which explicitly runs without a transaction.)

The validation lives in `RestrictedTransactionalEventListenerFactory`, registered by `AbstractTransactionManagementConfiguration`.

Our production system ran Spring Boot 3.3.5 (Spring Framework 6.1.x), so this validation was active. But it only blocks REQUIRED and other incorrect propagations — `REQUIRES_NEW` is allowed, and that's exactly what we had. Spring 6.1 prevents the data leak scenario but not the connection starvation. The starvation is a "legitimate" configuration that the framework can't prevent — it depends on runtime concurrency and pool sizing.

### Why no @Transactional is actually safer

Without `@Transactional`, DB access goes through `DataSourceUtils.getConnection()`:

```java
ConnectionHolder conHolder =
    TransactionSynchronizationManager.getResource(dataSource);
if (conHolder != null &&
    (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
    conHolder.requested();
    if (!conHolder.hasConnection()) {
        conHolder.setConnection(fetchConnection(dataSource));
    }
    return conHolder.getConnection();  // <- reuses connection #1, no deadlock
}
```

`DataSourceUtils.getConnection()` simply checks: is there a connection holder bound to this thread with an active connection? If yes, reuse it. No transaction semantics, no `doBegin()`, no new connection. It's honest about the situation — there's no transaction wrapping your writes, and it doesn't pretend otherwise.

### The full picture

| Propagation | Deadlock risk | Data commits | Exception rollback | Spring 6.1 |
|---|---|---|---|---|
| **REQUIRES_NEW** | **Yes** (new connection) | Normal commit | Normal rollback | Allowed |
| **REQUIRED** | No (reuses connection) | autoCommit side effect | **Broken — data leaks** | **Blocked at startup** |
| **No @Transactional** | No (reuses connection) | autoCommit side effect | No TX protection (each SQL auto-commits independently) | N/A |

REQUIRES_NEW is the only option that gives you real transaction semantics — but it's also the only one that deadlocks. That's why the fix needs to address the connection contention at the thread level, not the propagation level.

---

## The Fix

### Core idea: @Async to isolate threads

```java
@Configuration
@EnableAsync
public class ApplicationConfig {

    /**
     * Rejection policy MUST NOT be CallerRunsPolicy:
     * the caller is the AFTER_COMMIT thread still holding its connection.
     * Falling back to that thread reintroduces the deadlock.
     */
    @Bean("eventListenerExecutor")
    public Executor eventListenerExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("event-listener-");
        executor.setRejectedExecutionHandler((r, e) ->
            log.error("Event listener pool full, event discarded. "
                    + "Retry scanner will compensate."));
        executor.initialize();
        return executor;
    }
}
```

```java
@Service
public class OrderEventHandler {
    private final TransactionTemplate transactionTemplate;

    @Async("eventListenerExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCreated(OrderCreatedEvent event) {
        transactionTemplate.executeWithoutResult(status -> {
            notificationRepository.save(
                new Notification(event.orderId(), "Order confirmed"));
            messageSender.send(ORDER_TOPIC, event.orderId());
        });
    }
}
```

### Three design decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Async execution | `@Async` with bounded pool | Separate thread = separate connection = no competition |
| Rejection policy | Discard + alert, **NOT CallerRunsPolicy** | CallerRunsPolicy falls back to the caller's thread, reintroducing the deadlock |
| Transaction management | `TransactionTemplate` | Avoids implicit dependency on `@Async` / `@Transactional` proxy ordering |

Capacity rule of thumb: size the async pool against DB pool headroom, not CPU only. For example, keep listener max concurrency `<= maxActive - peakForegroundDbConcurrency`; otherwise async listeners can still starve foreground traffic.

### The CallerRunsPolicy trap

This deserves emphasis. `CallerRunsPolicy` is often recommended as a "safe" default for thread pool saturation. But in this specific context, the "caller" is the AFTER_COMMIT callback thread — the one still holding its connection. Running the task back on that thread is exactly the deadlock scenario we're trying to escape.

### The @Async + @Transactional proxy ordering concern

When both annotations are on the same method, `@Async`'s proxy must wrap around `@Transactional`'s proxy for correct behavior. Spring Boot 3.x gets this right by default, but it's an implicit ordering dependency. `TransactionTemplate` makes the transaction boundary explicit — no proxy ordering to worry about.

### Alternative approaches

`@Async` is not the only way to solve this. The right choice depends on your consistency requirements:

| Approach | Consistency | Connection risk | Complexity | Failure recovery |
|----------|------------|-----------------|------------|-----------------|
| Same-TX direct call | Strong for local DB writes; cross-system effects are not atomic without outbox/XA | None — single connection | Lowest | Automatic for local transaction |
| AFTER_COMMIT + REQUIRES_NEW (no @Async) | Eventual | **Starvation risk** | Low | Needs retry mechanism |
| AFTER_COMMIT + @Async + TransactionTemplate | Eventual | None — separate thread | Medium | Needs retry mechanism |
| Transactional outbox + scheduler | Eventual | None — single connection for write | Medium | Built-in (polling) |

For our case, the task table itself was already a transactional outbox — a retry scheduler scans for pending tasks. The simplest fix would have been to drop the event chain entirely and write the follow-up task in the same transaction. We chose `@Async` as the immediate fix to minimize blast radius, but the long-term plan is to simplify back to the same-TX approach.

---

## After the Fix

```
Thread A (caller)                       Thread B (from pool)
-----------------                       --------------------
1. Begin TX, get C1
2. Business logic ...
3. publishEvent(...)
4. commit()
5. AFTER_COMMIT fires
   +-- @Async -> submit to pool         onOrderCreated()
6. Return, C1 back to pool               +-- TransactionTemplate
                                          +-- Get C2 (independent)
                                          +-- DB writes + MQ
                                          +-- Commit, C2 back to pool
```

This removes the single-thread hold-C1-then-wait-C2 cycle that created circular waiting under pool saturation.

The trade-off: `@Async` introduces eventual consistency. There's a brief window where the order is committed but the follow-up hasn't executed yet. If the application crashes in that window, the notification is lost — unless you have a compensation mechanism. In our case, a retry scheduler scans for orders missing confirmations, so the gap is covered. The listener must be **idempotent**: the scheduler may fire after the async listener has already succeeded, so duplicate execution should be safe.

### What we verified after rollout

We validated the fix with runtime signals instead of just code inspection:

- `jdbc.connections.active` no longer pinned at `maxActive` for long intervals.
- MQ consumer lag recovered instead of monotonically increasing.
- Request timeout rate dropped back to baseline.

If these don't improve together, the bottleneck moved rather than disappeared.

---

## Takeaways

1. **"Restart first" destroys evidence.** Take a thread dump and connection pool snapshot before you restart. You might not get a second chance — we got lucky that one pod stayed stuck.

2. **Spring releases connections in `cleanupAfterCompletion()`**, not at `commit()`. Everything between is a danger zone.

3. **REQUIRED in AFTER_COMMIT doesn't deadlock — it does something worse.** It "joins" a committed transaction where rollback silently fails and data commits through an `autoCommit` side effect. Spring 6.1 now blocks this at startup ([#30679](https://github.com/spring-projects/spring-framework/issues/30679)).

4. **`CallerRunsPolicy` can reintroduce the exact problem you're solving.** Always ask "who is the caller?" before choosing a rejection policy.

5. **`@TransactionalEventListener(AFTER_COMMIT)` + `@Transactional(REQUIRES_NEW)` is the only combination with correct transaction semantics — but it starves the connection pool without `@Async`.** You can't fix this at the propagation level; you need thread isolation or a different architecture (outbox, same-TX direct call).

6. **Monitor `jdbc.connections.active`.** Connection pool exhaustion is silent — health probes pass, the JVM is fine, but the service is dead.

---

## Operational Checklist

1. Before restart, capture `jstack`, connection pool active/waiting metrics, and recent timeout logs.
2. Confirm symptom type: many threads WAITING on pool conditions (`notEmpty`) with near-zero BLOCKED threads.
3. Check whether `jdbc.connections.active` is pinned near `maxActive` and whether waiters/timeouts are increasing.
4. Search for `@TransactionalEventListener(AFTER_COMMIT)` + `REQUIRES_NEW` and other patterns that acquire a second connection in callbacks.
5. Verify async rejection policy is not `CallerRunsPolicy` for commit-path callbacks.
6. Apply short-term mitigation (`maxActive`/concurrency tuning), then implement a structural fix (`@Async` isolation, outbox, or same-TX simplification).
7. After rollout, validate with key signals together: active connections, timeout rate, and consumer lag.

---

## Appendix: A Note on Vibe Coding

This incident was introduced during a vibe coding refactor. Here's how the code evolved:

**V1 — The original (worked for months):**

```java
@EventListener(TaskCreateEvent.class)
@Transactional(rollbackFor = Exception.class)
public void onTaskCreateEvent(TaskCreateEvent event) {
    taskDomainService.saveTask(task);
    messageProducer.send(MessageTopic.AI_TASK, taskMessage);
}
```

`@EventListener` runs synchronously during `publishEvent()`, inside the caller's transaction. `REQUIRED` propagation joins the existing transaction, reuses the connection. No deadlock risk. The only issue: the outer transaction included a long-running LLM call, so the connection was held for minutes. A resource efficiency problem, not a correctness one.

**V2 — The vibe coding refactor (introduced the deadlock):**

```java
/**
 * Uses AFTER_COMMIT to ensure the event fires only after
 * the publishing transaction commits successfully.
 * Avoids data inconsistency where the main transaction rolls back
 * but the task notification has already been sent.
 */
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public void onTaskCreateEvent(TaskCreateEvent event) {
    taskDomainService.saveTask(task);
    messageProducer.send(MessageTopic.AI_TASK, taskMessage);
}
```

The refactor replaced `@EventListener` + `@Transactional(REQUIRED)` with `@TransactionalEventListener(AFTER_COMMIT)` + `@Transactional(REQUIRES_NEW)`. The code was clean, the comment was thorough, and the reasoning was sound on the surface. Two problems hid beneath:

1. **The AI-generated comment explained the reasoning perfectly** — "ensure the event fires only after the publishing transaction commits" — but the implementation was a high-risk `AFTER_COMMIT` + `REQUIRES_NEW` pattern under bounded pool and concurrency. Single-threaded tests don't deadlock; production does.
2. **Nobody questioned the premise.** Our task system already had a retry scheduler that scans for pending/failed tasks. The task table itself was effectively a transactional outbox. The entire event-driven approach was unnecessary — a simple `try { messageProducer.send(...) } catch { log.warn("scheduler will retry") }` inside the same transaction would have been sufficient.

Each step in the AI's reasoning was locally correct: events for decoupling → `AFTER_COMMIT` to avoid inconsistency → `REQUIRES_NEW` because "we need a new transaction." The code compiled, the tests passed (single-threaded tests don't deadlock), and it read like a textbook Spring pattern. But the underlying assumptions were never examined: *Do we actually need event decoupling here? What's the connection state when AFTER_COMMIT fires? What does "new transaction" mean at the JDBC level?*

This is the real lesson: vibe coding can get you to "it works" fast, but "it works" and "it works under concurrency with a bounded connection pool" are very different things. The transaction boundary was chosen by vibes, not by thinking through the actual requirements.

---

*References:*
- [Spring Issue #30679 - Detect invalid transaction configuration for transactional event listeners](https://github.com/spring-projects/spring-framework/issues/30679) (Spring 6.1, blocks REQUIRED on @TransactionalEventListener)
- [Spring Issue #26250 - REQUIRES_NEW connection pool deadlock](https://github.com/spring-projects/spring-framework/issues/26250) (closed as "by design")
- [Spring Framework - AbstractPlatformTransactionManager](https://github.com/spring-projects/spring-framework/blob/main/spring-tx/src/main/java/org/springframework/transaction/support/AbstractPlatformTransactionManager.java)
- [Spring Framework - DataSourceTransactionManager](https://github.com/spring-projects/spring-framework/blob/main/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/DataSourceTransactionManager.java)
- [Spring Framework - DataSourceUtils](https://github.com/spring-projects/spring-framework/blob/main/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/DataSourceUtils.java)
- [Spring Docs - Transaction-bound Events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)
- [Beware of slow transaction callbacks in Spring](https://www.javacodegeeks.com/2017/03/beware-slow-transaction-callbacks-spring.html)
