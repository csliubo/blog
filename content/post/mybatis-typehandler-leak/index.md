---
title: "Why Is Fastjson Parsing My VARCHAR? A MyBatis-Plus TypeHandler Mystery"
description: "A MyBatis-Plus TypeHandler misconfiguration silently hijacked every VARCHAR column — here's how I traced and fixed it"
date: 2026-02-26
draft: false
slug: "mybatis-typehandler-leak"
categories:
    - Deep Dive
tags:
    - Java
    - MyBatis
    - MyBatis-Plus
    - Troubleshooting
---

## The Symptom

After deploying our internal AI Console — a two-node setup behind a load balancer — I noticed some dashboard queries were intermittently failing. These were simple aggregation queries: success counts, failure counts, token usage.

The error:

```text
com.alibaba.fastjson.JSONException: syntax error, pos 7, line 1, column 8SUCCESS
```

A JSON parser choking on the string `SUCCESS` — a plain status field, not JSON.

Two things stood out. First, the failure was intermittent — one node would fail while the other succeeded, with no code difference between them. Second, the library name: **Fastjson**. I don't use Fastjson. I actively ban it from my projects. So why is it parsing my query results?

---

## Tracing the Phantom Dependency

The failing queries looked like this:

```xml
<select id="selectStatusCounts" resultType="java.util.Map">
    SELECT status, COUNT(*) as cnt
    FROM llm_call_log
    GROUP BY status
</select>
```

Straightforward — returns a `Map<String, Object>` where `status` is a plain VARCHAR column with values like `SUCCESS`, `FAILED`, etc.

I searched the codebase for Fastjson imports. Nothing in our code. The Fastjson library itself was present as a transitive dependency from another module, but nobody was using it directly. Then I found that MyBatis-Plus ships `FastjsonTypeHandler` in its extension package — and if Fastjson is on the classpath, that handler becomes functional.

The question shifted from "why is Fastjson here?" to "why is it being *used*?"

---

## The Configuration That Started It All

Found it in `application.yml`:

```yaml
mybatis-plus:
  type-handlers-package: com.baomidou.mybatisplus.extension.handlers,com.hmc.common.ext.mybatis
```

The second package is ours — custom TypeHandlers for JSON fields. But the first package, `com.baomidou.mybatisplus.extension.handlers`, is MyBatis-Plus's built-in handler package. Someone had added it early in the project to make JSON TypeHandlers work globally. That package contains `FastjsonTypeHandler`:

```java
@MappedTypes({Object.class})
@MappedJdbcTypes(JdbcType.VARCHAR)
public class FastjsonTypeHandler extends AbstractJsonTypeHandler<Object>
```

`@MappedTypes({Object.class})` — this handler registers itself as the fallback for any type without a more specific handler, including `Object` as used in `Map<String, Object>`. When registered globally via `type-handlers-package`, it becomes the handler for `(Object.class, VARCHAR)`.

Now, what's the value type of `Map<String, Object>`? It's `Object`. So every VARCHAR column in a `resultType="java.util.Map"` query gets routed through `FastjsonTypeHandler`. A plain string like `SUCCESS` gets fed to a JSON parser, which naturally chokes on it.

---

## The Obvious Fix (That Broke Everything)

The obvious next step: remove `com.baomidou.mybatisplus.extension.handlers` from `type-handlers-package` and keep only our custom package.

```yaml
# Before
type-handlers-package: com.baomidou.mybatisplus.extension.handlers,com.hmc.common.ext.mybatis

# After
type-handlers-package: com.hmc.common.ext.mybatis
```

I applied the change and restarted.

**The application failed to start.**

```text
Caused by: java.lang.IllegalStateException: No typehandler found for property hotWords
```

Some of our XML mapper files had ResultMaps that relied on the globally registered TypeHandlers — without explicitly declaring them. Remove the global registration, and MyBatis can't figure out how to map those fields anymore.

This meant I couldn't just rip out the configuration. I needed to understand how the TypeHandler system works.

---

## Two Independent Systems

MyBatis-Plus has **two separate paths** for applying TypeHandlers, and they don't know about each other.

**Path 1** — MP reads `@TableField(typeHandler=...)` annotations at startup, generates a ResultMap with the TypeHandler baked in, and registers it. This ResultMap is used by built-in methods like `selectById`.

**Path 2** — XML ResultMaps are parsed by MyBatis core. They don't read `@TableField` annotations. If you don't explicitly declare `typeHandler` in the XML, it falls back to the `TypeHandlerRegistry` — the global lookup table.

I wrote a test to verify this:

```java
@Test // Path 1: MP selectById — uses auto-generated ResultMap with TypeHandler
public void path1_mpSelectById() {
    TestEntity entity = testMapper.selectById(1L);
    assertInstanceOf(Map.class, entity.getJsonData());      // Parsed as Map
}

@Test // Path 2: XML ResultMap without typeHandler — gets raw String
public void path2_xmlResultMap_noTypeHandler() {
    TestEntityRaw raw = testMapper.selectByIdWithXmlMap(1L);
    assertInstanceOf(String.class, raw.getJsonDataRaw());   // Raw JSON string
}

@Test // XML ResultMap with explicit typeHandler — works
public void path3_xmlResultMap_withTypeHandler() {
    TestEntity entity = testMapper.selectByIdWithXmlMapAndHandler(1L);
    assertInstanceOf(Map.class, entity.getJsonData());      // Parsed as Map
}
```

All three pass. The entity annotation and the XML ResultMap are **independent systems**. The annotation doesn't leak into XML, and XML doesn't read annotations.

If you define an XML ResultMap that maps a `Map<String, Object>` field without specifying a TypeHandler, and there's no global registration — **the application won't start**:

```text
Caused by: java.lang.IllegalStateException: No typehandler found for property jsonData
```

This confirms that the XML path has zero awareness of `@TableField(typeHandler=...)`. The `hotWords` startup failure from the "obvious fix" was exactly this — XML ResultMaps that had been silently depending on global registration.

---

## How the Bug Actually Works

With both systems understood, here's how MyBatis decides which TypeHandler to use for each column:

```text
Query executes, ResultSet comes back
  |
  +-- YES: Statement has a ResultMap
  |   |
  |   +-- For each ResultMapping:
  |   |     TypeHandler is already bound (resolved at startup)
  |   |     -> Use it directly
  |   |
  |   +-- Unmapped columns -> Auto-mapping:
  |         -> Look up TypeHandlerRegistry by (javaType, jdbcType)
  |         -> Found? Use it. Not found? Skip (or fail for complex types)
  |
  +-- NO: resultType specified
        -> Create empty ResultMap, ALL columns go through auto-mapping
        -> Every column: TypeHandlerRegistry.getTypeHandler(javaType, jdbcType)

  (*) This is where the global registration matters:
      type-handlers-package registers handlers into TypeHandlerRegistry.
      Any query using auto-mapping (resultType, or unmapped columns) hits this registry.
```

Our `resultType="java.util.Map"` query hits the `NO` branch — all columns go through auto-mapping, which consults the `TypeHandlerRegistry`. And what's registered there?

```text
type-handlers-package scans com.baomidou.mybatisplus.extension.handlers
  -> Finds FastjsonTypeHandler with @MappedTypes({Object.class})
  -> Registers as (Object.class, VARCHAR) handler
  -> Also finds Fastjson2TypeHandler, GsonTypeHandler, JacksonTypeHandler
     with the same annotations
  -> All register to the same key -- last one wins
  -> Scan order is not guaranteed by the spec
```

This also explains the intermittent behavior. The failure wasn't random — it was **deterministic per JVM launch, but non-deterministic across launches**. If `FastjsonTypeHandler` happened to register last, it would parse `SUCCESS` as JSON and throw. If `GsonTypeHandler` registered last, it would silently return the string as-is (Gson is lenient with non-JSON strings). Same code, same config, different behavior depending on which class happened to be scanned last.

Two nodes behind a load balancer, two independent JVM launches, two different scan orders — one fails, one doesn't.

---

## The Fix

The solution has two parts:

**1. Remove the problematic scan path:**

```yaml
mybatis-plus:
  # Only scan our own TypeHandler package
  type-handlers-package: com.hmc.common.ext.mybatis
```

**2. Fix the implicit dependencies:**

For every XML ResultMap that relied on globally registered TypeHandlers, explicitly declare them:

```xml
<!-- Before: relied on global registration -->
<result column="hot_words" property="hotWords"/>

<!-- After: explicit TypeHandler -->
<result column="hot_words" property="hotWords"
        typeHandler="com.hmc.common.ext.mybatis.JsonListStringHandler"/>
```

For entities using `@TableField(typeHandler=...)`, ensure `@TableName(autoResultMap = true)` is present — this tells MP to generate a ResultMap that carries the TypeHandler into queries, not just inserts and updates. (See [Going Deeper](#going-deeper-why-autoresultmap-defaults-to-false) for why this isn't the default.)

---

## Takeaways

1. **`type-handlers-package` should only contain your own TypeHandlers.** Never include `com.baomidou.mybatisplus.extension.handlers` — those are designed for per-field use via `@TableField`, not global registration.

2. **`@TableField(typeHandler=...)` and XML ResultMap are independent systems.** Entity annotations don't carry over to XML. If you use both, declare TypeHandlers in both places.

3. **`@TableField(typeHandler=...)` requires `autoResultMap = true` for queries.** Without it, only inserts and updates use the TypeHandler. This is a known footgun in MyBatis-Plus — [reported since 2018](https://github.com/baomidou/mybatis-plus/issues/357), still the default behavior. The next section traces this default back to an upstream MyBatis change.

4. **Audit your `type-handlers-package` now.** Grep for it in your `application.yml`. If it includes any third-party package, check what `@MappedTypes` those handlers declare — any handler with `@MappedTypes({Object.class})` will silently hijack all auto-mapped columns of that type.

---

## Going Deeper: Why `autoResultMap` Defaults to `false`

If you use `@TableField(typeHandler=...)`, you've probably seen the advice to add `@TableName(autoResultMap = true)`. Here's why it's not the default.

When `autoResultMap` is disabled, MP generates SQL with column aliases for auto-mapping:

```sql
SELECT user_name AS userName FROM user
-- Auto-mapping matches "userName" to the property
```

When `autoResultMap` is enabled, MP generates a ResultMap that maps `column="user_name"` to `property="userName"`. But if the SQL *also* adds an `AS userName` alias, the ResultSet column name becomes `userName`, while the ResultMap expects `user_name`. They don't match.

This mismatch became fatal after [MyBatis PR #895](https://github.com/mybatis/mybatis-3/pull/895) (included in MyBatis 3.4.3), which tightened auto-mapping behavior. Previously, if a ResultMap's explicit column mapping failed to match, MyBatis would fall back to auto-mapping. After the fix, properties already declared in a ResultMap are **skipped** by auto-mapping — no fallback.

The original PR fixed a real bug: when a table has columns `phone` and `phone_number`, and the ResultMap maps `phone_number` to property `phone`, auto-mapping would also map column `phone` to property `phone` — silently overwriting the explicit mapping with the wrong value. The fix was correct: explicit mappings should not be overridden by auto-mapping.

But as a side effect, MP had to choose: either keep aliases (and break ResultMap column matching) or drop aliases (and rely on ResultMap for all column-to-property mapping). They chose the latter — when `autoResultMap` is enabled, [aliases are omitted](https://github.com/baomidou/mybatis-plus/blob/3.0/mybatis-plus-core/src/main/java/com/baomidou/mybatisplus/core/metadata/TableFieldInfo.java) from the generated SQL.

Since this changes SQL generation behavior, enabling it by default could break existing applications. So `autoResultMap` stays `false` by default, and users discover the hard way that their `@TableField(typeHandler=...)` only works for inserts and updates — not queries. For new projects, enabling it at the entity level is safe — just be aware it changes how MP generates SELECT statements.

---

*References:*
- [MyBatis-Plus TypeHandler Documentation](https://baomidou.com/en/guides/type-handler/)
- [MyBatis PR #895 — Avoiding unexpected auto-mapping](https://github.com/mybatis/mybatis-3/pull/895)
- [MyBatis-Plus Issue #357 — TypeHandler not working for queries](https://github.com/baomidou/mybatis-plus/issues/357)
- [MyBatis-Plus Issue #5124 — autoResultMap vs XML resultType](https://github.com/baomidou/mybatis-plus/issues/5124)
- [MyBatis TypeHandlerRegistry Source](https://github.com/mybatis/mybatis-3/blob/master/src/main/java/org/apache/ibatis/type/TypeHandlerRegistry.java)
