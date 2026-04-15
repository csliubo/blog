---
title: "Three Commits, Seventeen Months Apart, One Bug: A MyBatis L1 Cache Story"
description: "A deposit record merged into a list silently polluted the MyBatis L1 cache, but only when three independently reasonable commits aligned"
date: 2026-04-15
draft: false
slug: "mybatis-l1-cache-pollution"
categories:
    - Deep Dive
tags:
    - Java
    - MyBatis
    - MyBatis-Plus
    - Troubleshooting
---

## The Symptom

An API for editing order details started throwing a business error: `ORDER_ITEM_CAN_NOT_DELETE — only pending or cancelled items can be deleted`. It happened whenever the order contained deposit (prepaid) entries. The older version of the same endpoint, calling the same downstream logic, worked fine.

The developer's initial instinct: "MyBatis-Plus caching issue; the first and second query return different results." Directionally correct, but the mechanism was not what anyone expected. The root cause turned out to be a single `.add()` call written seventeen months before the bug appeared.

---

## The Clue: Same Object, Different Size

The call chain was: `updateOrderDetail` calls `getOrderDetail` (which queries order items), then calls `saveOrderDetail` (which queries order items again before diffing). Two queries to the same mapper, in the same `@Transactional` method.

I added identity probes at both query sites:

```java
// Probe at query site #1 (in OrderQueryService.getOrderDetail)
List<OrderItem> items = orderItemService.listByOrderId(orderId);
System.out.println("query#1 identity=" + System.identityHashCode(items) + " size=" + items.size());

// Probe at query site #2 (in OrderUpdateService.saveOrUpdate — different method, same mapper call)
List<OrderItem> oldItems = orderItemService.listByOrderId(orderId);
System.out.println("query#2 identity=" + System.identityHashCode(oldItems) + " size=" + oldItems.size());
```

Results with the failing endpoint:

```
query#1 identity=1697248320 size=4
query#2 identity=1697248320 size=6
```

Same identity hash. Same object reference. But `size` grew from 4 to 6 between the two calls.

So where did the extra 2 come from? The database had 4 rows:

```sql
SELECT id, product_type, status FROM order_item
WHERE order_id = ? AND tenant_id = ? AND is_del = 0;
-- Returns 4 rows, all product_type = PRODUCT
```

And 2 deposit entries in a separate table:

```sql
SELECT id FROM customer_deposit
WHERE order_id = ? AND type = 0;
-- Returns 2 rows
```

4 + 2 = 6. A search through the call chain between the two query sites found it: `getOrderDetail` was `.add()`-ing deposit entries into the list after query #1. And somehow query #2 was seeing that mutation instead of hitting the database for a fresh result. The save logic diffed old items against new, saw 2 deposit entries that did not exist in the incoming list, tried to delete them, and failed the status check.

---

## The Bug: Three Commits That Conspired

Three changes, each independently reasonable, combined to create this bug.

### Commit 1: The Mutation (November 2024)

A developer added deposit entries to the order item list for display purposes:

```java
// OrderQueryService.getOrderDetail()
List<OrderItem> items = orderItemService.listByOrderId(orderId);
// ... fetch deposit entries from a separate table, convert to OrderItem VO ...
items.add(convertedDeposit);  // mutates the list returned by the mapper
```

This made the API response include deposits alongside regular items — a reasonable UI requirement.

The problem: `listByOrderId` is a MyBatis mapper call. MyBatis's first-level (L1) cache, managed by `BaseExecutor`, stores query results in a `PerpetualCache`, which is just a `HashMap<Object, Object>`. When a subsequent query hits the cache, `localCache.getObject(key)` returns the raw reference from the map:

```java
// BaseExecutor.query() — simplified
list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
if (list != null) {
    handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
} else {
    list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

No defensive copy. No `Collections.unmodifiableList()`. The exact same `List` object is returned. So when `getOrderDetail` called `items.add(convertedDeposit)`, it was mutating the list that the L1 cache held a reference to.

At this point, the mutation was harmless: `getOrderDetail` was always the terminal consumer. Nobody queried the same mapper method again in the same transaction.

### Commit 2: The Second Query (April 2026, Week 1)

A new feature added operation logging: "show what changed between the old and new order." This required fetching the current state before applying updates:

```java
// OrderAppService.updateOrderDetail()
OrderDetail current = orderQueryService.getOrderDetail(orderId);  // query #1
// ... diff logic for operation log ...
saveOrderDetail(orderId, newItems);  // eventually calls query #2
```

Now two calls to the same mapper existed in the same call chain. But this alone did not trigger the bug. Without `@Transactional`, Spring's `SqlSessionTemplate` opens and closes a new `SqlSession` for each mapper call. Two separate sessions, two separate caches, no pollution.

### Commit 3: The Transaction (April 2026, Week 2)

The developer added DML operations to `updateOrderDetail` and correctly wrapped the method in `@Transactional`:

```java
@Transactional
public void updateOrderDetail(Long orderId, List<OrderItemDTO> newItems) {
    OrderDetail current = orderQueryService.getOrderDetail(orderId);  // query #1
    saveOrderDetail(orderId, newItems);  // contains query #2
}
```

`@Transactional` binds a single `SqlSession` to the current thread for the duration of the method. Now query #1 and query #2 share the same executor, the same `localCache`.

MyBatis builds a `CacheKey` from the statement ID, the RowBounds offset and limit, the SQL string, and each parameter value. Both queries had identical inputs, so both produced the same `CacheKey`. Query #2 hit the cache and got back the exact same `List` object that query #1 returned. The one that `.add()` had already mutated with 2 extra deposit entries.

### The Timeline

```
Nov 2024     items.add(deposit)           — pollution source (dormant)
Apr 2026 W1  added getOrderDetail() call  — two queries in same call chain
Apr 2026 W2  added @Transactional         — shared SqlSession, cache hit, bug fires
```

Seventeen months between the first commit and the bug surfacing. Each commit passed code review. Each was correct in isolation.

---

## Why the Old Endpoint Worked

The older endpoint called the same `getOrderDetail` (query #1) and the same `saveOrderDetail` (query #2). It also had `@Transactional`. It should have had the same bug.

It didn't, by accident. Between query #1 and query #2, the old endpoint ran six DML operations:

```java
@Transactional
public void editWithDetail(Long orderId, ...) {
    OrderDetail current = orderQueryService.getOrderDetail(orderId);  // query #1, cache populated + polluted

    // Six DML operations:
    updateCustomerInfo(orderId);             // UPDATE customer_extra
    updateOrder(orderId, ...);               // UPDATE order
    updateProducts(orderId, ...);            // UPDATE/INSERT order_item
    saveFormData(orderId, ...);              // INSERT/UPDATE form_data
    updateActivityLog(orderId, ...);         // UPDATE activity_log
    updateAttributionSource(orderId, ...);   // UPDATE customer_detail

    saveOrderDetail(orderId, newItems);     // query #2 — cache MISS, hits DB, gets clean data
}
```

The first DML wiped the cache. `BaseExecutor.update()` calls `clearLocalCache()` before every DML:

```java
// BaseExecutor.update()
public int update(MappedStatement ms, Object parameter) throws SQLException {
    if (closed) throw new ExecutorException("Executor was closed.");
    clearLocalCache();          // wipes the ENTIRE L1 cache — all tables, all queries
    return doUpdate(ms, parameter);
}
```

The clearing is all-or-nothing: any INSERT, UPDATE, or DELETE in the current session wipes every cached query result, regardless of which table was involved. MyBatis does not track table dependencies.

So the first `updateCustomerInfo` triggered `clearLocalCache()`, the polluted list was gone, and query #2 missed the cache, hit the database, and got the correct 4 rows.

**The old endpoint was protected by luck, not by design.** If someone refactored away those intermediate DMLs, or reordered them to run after query #2, the same bug would appear.

I verified this with a controlled experiment: commenting out the six DML operations in the old endpoint produced identical behavior to the new one. Same identity hash, size 6, same business error.

---

## The saveBatch Trap

The bug above happened because `.add()` polluted a cached list. But there is a second way L1 can silently serve stale data, and it is easier to walk into. You do not need to mutate anything; just using the wrong save method is enough.

MyBatis-Plus's batch operations (`saveBatch`, `saveOrUpdateBatch`, `removeBatchByIds`) open a **separate** `SqlSession` with `ExecutorType.BATCH`:

```java
// Inside SqlHelper.executeBatch()
SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
try {
    consumer.accept(sqlSession);
    sqlSession.commit(!transaction);  // commits only if not in a Spring-managed transaction
} finally {
    sqlSession.close();
}
```

Each `SqlSession` has its own `BaseExecutor`, its own `localCache`. The batch session's DML clears the batch session's cache, not the outer transaction's. So:

```java
@Transactional
public void process() {
    List<Item> before = itemMapper.selectList(condition);  // cached in outer L1
    itemService.saveBatch(newItems);                        // runs in separate SqlSession
    List<Item> after = itemMapper.selectList(condition);    // L1 cache hit — stale!
    // 'after' does NOT include the newly saved items
}
```

The two sessions share the same JDBC connection (same database transaction), so the data is consistent at the DB level. But the L1 cache in the outer session has no idea that rows were inserted by the batch session.

---

## The Fix

One line:

```java
// Before
List<OrderItem> items = orderItemService.listByOrderId(orderId);

// After
List<OrderItem> items = new ArrayList<>(orderItemService.listByOrderId(orderId));
```

This breaks the reference link between the caller's list and the L1 cache. Subsequent `.add()` calls mutate the copy, not the cached original.

A more thorough fix would be to wrap the return value at the domain service layer, protecting all callers:

```java
public List<OrderItem> listByOrderId(Long orderId) {
    return new ArrayList<>(orderItemMapper.selectList(buildWrapper(orderId)));
}
```

This is safer long-term but changes behavior for every caller at once. Higher confidence, higher blast radius. For the immediate fix, wrapping at the call site is lower risk.

---

## Detecting L1 Cache Pollution

When an API returns a business error but the database looks correct, three signals confirm L1 cache pollution:

1. Enable MyBatis SQL logging (`log-impl: STDOUT_LOGGING`). If the same `SELECT` appears only once in the log but your code calls it twice, the second call hit L1.
2. Print `System.identityHashCode(list)` at both query sites. Same hash = same object = cache hit.
3. Compare `list.size()` at both sites against the actual `SELECT COUNT(*)` from the database. A mismatch means the list was mutated after caching.

All three signals together are diagnostic. Any one alone could have other explanations.

---

## Takeaways

1. Treat any `List` returned by a MyBatis mapper as immutable. If you need to add or remove elements, copy it first with `new ArrayList<>(source)`. The L1 cache holds a direct reference to the same object. Your mutation is the cache's mutation.

2. L1 cache scope = `SqlSession` scope. Under Spring `@Transactional`, that means the entire method. Two identical queries in the same transaction return the same cached object, and any DML between them wipes the cache entirely (not per-table; all of it). No DML between the two queries means the second one silently returns the cached result, mutations included.

3. `saveBatch` and `saveOrUpdateBatch` run in a separate `SqlSession`. Their DML does not clear the outer transaction's L1 cache. A `selectList` after `saveBatch` in the same `@Transactional` method returns stale data unless you manually clear the cache or run a non-batch DML first.

4. When commits months apart each look correct in isolation, the bug they create together is invisible to single-diff code review. Enforce mapper return immutability as a convention. It is simple enough to check in review without tracing the full call graph.

---

*References:*
- [MyBatis BaseExecutor.java](https://github.com/mybatis/mybatis-3/blob/master/src/main/java/org/apache/ibatis/executor/BaseExecutor.java)
- [MyBatis PerpetualCache.java](https://github.com/mybatis/mybatis-3/blob/master/src/main/java/org/apache/ibatis/cache/impl/PerpetualCache.java)
- [MyBatis-Plus SqlHelper.java](https://github.com/baomidou/mybatis-plus/blob/master/mybatis-plus-extension/src/main/java/com/baomidou/mybatisplus/extension/toolkit/SqlHelper.java)
