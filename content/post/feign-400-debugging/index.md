---
title: "Debugging Intermittent HTTP 400: A Tale of Feign, Hutool, and JDK Class Loading"
description: "How a missing 'else' in Feign, a security bypass in Hutool, and JDK's static initialization created a bug that only appears sometimes"
date: 2026-02-06
slug: "feign-400-debugging"
categories:
    - Incidents
tags:
    - Java
    - Feign
    - JDK
    - Troubleshooting
---

## The Symptom

During regression testing on Stage environment, we hit persistent HTTP 400 errors:

- Service A calls Service B via Feign, returns 400
- Same Docker image works fine in test environment
- After restarting the Stage node, it might work — or might not
- Once a node starts failing, all requests fail; once it works, it keeps working

This "behavior changes after restart" pattern pointed to class loading order or static initialization issues.

---

## A Bug That Hides

This wasn't our first encounter with this 400 error.

**First time**: The error appeared, we switched Feign's HTTP client to OkHttp, restarted, problem gone. Case closed — or so we thought.

**Later**: A developer switched back to the default client for some reason. No 400 errors appeared (class loading order happened to be "safe"), so no one followed up.

**This time**: The error resurfaced after a deployment. This time we preserved the environment and dug in.

**Lessons**:
- "Fixed after restart" ≠ actually fixed
- "Can't reproduce" ≠ bug doesn't exist
- Class loading order bugs can hide for months, then suddenly strike

---

## Packet Capture: The Smoking Gun

Using tcpdump to capture the failing request:

```bash
# Capture HTTP traffic on port 8080, show ASCII payload
tcpdump -i eth0 -A -s 0 'tcp port 8080 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

Found duplicate `Content-Length` headers:

```http
POST /api/xxx HTTP/1.1
Host: service-b:8080
Content-Type: application/json
Content-Length: 156
Content-Length: 156
```

---

## Verifying the Behavior

To confirm Tomcat rejects duplicate `Content-Length`, we crafted a raw HTTP request using netcat:

```bash
printf 'POST /api/test HTTP/1.1\r\n'\
'Host: localhost:8080\r\n'\
'Content-Type: application/json\r\n'\
'Content-Length: 13\r\n'\
'Content-Length: 13\r\n'\
'\r\n'\
'{"test":true}' | nc localhost 8080
```

Response:

```http
HTTP/1.1 400 Bad Request
```

This confirms the server correctly rejects the request per [RFC 7230 Section 3.3.2](https://datatracker.ietf.org/doc/html/rfc7230#section-3.3.2):

> If a message is received that has multiple Content-Length header fields... the recipient MUST either reject the message as invalid or replace the duplicated field-values with a single valid Content-Length field.

Tomcat chooses to reject.

---

## Tracing to Feign

Duplicate headers must be added somewhere between our application code and the TCP layer. Since we're using Feign with the default `HttpURLConnection`-based client, we examined `feign.Client.Default` and found the culprit in the `convertAndSend` method.

---

## The Bug: Feign Adds Content-Length Twice

Located the issue in `feign.Client.Default` (Feign 13.1):

```java
// feign/Client.java - convertAndSend method
for (String field : request.headers().keySet()) {
    for (String value : request.headers().get(field)) {
        if (field.equals(CONTENT_LENGTH)) {
            if (!gzipEncodedRequest && !deflateEncodedRequest) {
                contentLength = Integer.valueOf(value);
                connection.addRequestProperty(field, value);  // First add
            }
        }
        // Note: This is 'if', not 'else if'
        if (field.equals(ACCEPT_ENCODING)) {
            hasAcceptHeader = true;
            connection.addRequestProperty(field, value);
            break;
        } else {
            connection.addRequestProperty(field, value);      // Second add!
        }
    }
}
```

**Execution flow when `field = "Content-Length"`:**

```
1. First if:  field.equals(CONTENT_LENGTH) → true
              → addRequestProperty("Content-Length", "156")  // Added once

2. Second if: field.equals(ACCEPT_ENCODING) → false
              → falls through to else branch
              → addRequestProperty("Content-Length", "156")  // Added again!
```

The second `if` should be `else if`. This causes `Content-Length` to be added twice.

Fixed in Feign 13.3 (changed to `else if`).

**But wait**: If this bug exists in the code, why doesn't it trigger every time?

---

## The Mask: JDK's Protection Mechanism

`HttpURLConnection` has a restricted headers mechanism that filters out certain "dangerous" headers.

From JDK source `sun.net.www.protocol.http.HttpURLConnection`:

```java
public class HttpURLConnection extends java.net.HttpURLConnection {

    // Static final — initialized at class load time
    private static final boolean allowRestrictedHeaders;

    private static final Set<String> restrictedHeaderSet;

    static {
        // Read from system property, defaults to false
        allowRestrictedHeaders = Boolean.parseBoolean(
            AccessController.doPrivileged(
                new GetPropertyAction("sun.net.http.allowRestrictedHeaders")
            )
        );

        // Restricted headers include:
        restrictedHeaderSet = new HashSet<>(Arrays.asList(
            "access-control-request-headers",
            "access-control-request-method",
            "connection",
            "content-length",      // ← Content-Length is restricted
            "content-transfer-encoding",
            "host",
            "keep-alive",
            "origin",
            "trailer",
            "transfer-encoding",
            "upgrade",
            "via"
        ));
    }
}
```

The `addRequestProperty` implementation:

```java
@Override
public synchronized void addRequestProperty(String key, String value) {
    if (connected)
        throw new IllegalStateException("Already connected");
    if (key == null)
        throw new NullPointerException("key is null");

    // Key: if restricted header, silently return without adding
    if (isRestrictedHeader(key, value)) {
        return;
    }

    requests.add(key, value);
}

private boolean isRestrictedHeader(String key, String value) {
    if (allowRestrictedHeaders) {
        return false;  // All headers allowed
    }
    return restrictedHeaderSet.contains(key.toLowerCase());
}
```

**Key insight**:

- When `allowRestrictedHeaders = false` (default), `Content-Length` is restricted
- Feign's two `addRequestProperty` calls are **silently ignored**
- JDK calculates the actual content length from the request body and sets the header automatically in `writeRequests()`
- The Feign bug is **masked** by JDK's protection

So why does the protection sometimes fail?

---

## The Trigger: static final and Class Loading Order

Notice `allowRestrictedHeaders` is `static final`:

```java
private static final boolean allowRestrictedHeaders;

static {
    allowRestrictedHeaders = Boolean.parseBoolean(
        AccessController.doPrivileged(
            new GetPropertyAction("sun.net.http.allowRestrictedHeaders")
        )
    );
}
```

**Static final variables are initialized at class load time and never change afterward.**

This means:

1. If `HttpURLConnection` loads when the property is unset or `false` → `allowRestrictedHeaders = false` → protection ON
2. If someone sets the property to `true` **before** `HttpURLConnection` loads → `allowRestrictedHeaders = true` → protection OFF
3. If the property is set **after** `HttpURLConnection` loads → too late, static final won't change

**The question becomes: Who sets this property? When?**

---

## Tracing the Property Source

Using a Property Hook to intercept `System.setProperty` calls:

```java
public class PropertyTracer {

    public static void install() {
        Properties original = System.getProperties();
        Properties traced = new Properties() {
            {
                putAll(original);
            }

            @Override
            public synchronized Object setProperty(String key, String value) {
                if (key.contains("allowRestrictedHeaders") ||
                    key.contains("sun.net.http")) {
                    System.err.println("[TRACE] setProperty: " + key + " = " + value);
                    new Exception("Stack trace").printStackTrace(System.err);
                }
                return super.setProperty(key, value);
            }
        };
        System.setProperties(traced);
    }
}
```

Call `PropertyTracer.install()` at the earliest point in application startup. Caught this stack trace:

```
[TRACE] setProperty: sun.net.http.allowRestrictedHeaders = true
java.lang.Exception: Stack trace
    at PropertyTracer$1.setProperty(PropertyTracer.java:15)
    at cn.hutool.http.GlobalHeaders.putDefault(GlobalHeaders.java:44)
    at cn.hutool.http.GlobalHeaders.<clinit>(GlobalHeaders.java:26)
    at cn.hutool.http.HttpRequest.<clinit>(HttpRequest.java:56)
    ...
```

The source: Hutool's `GlobalHeaders` class:

```java
// cn.hutool.http.GlobalHeaders
public GlobalHeaders putDefault(boolean isReset) {
    System.setProperty("sun.net.http.allowRestrictedHeaders", "true");
    // ...
}
```

---

## The Complete Causal Chain

```
┌─────────────────────────────────────────────────────────────────┐
│                         JVM Startup                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────────┐
              │   Class Loading Order (uncertain)  │
              └───────────────────────────────────┘
                     /                    \
                    /                      \
                   ▼                        ▼
        ┌─────────────────┐        ┌─────────────────┐
        │ Hutool loads    │        │ JDK loads       │
        │ first           │        │ first           │
        └─────────────────┘        └─────────────────┘
                │                          │
                ▼                          ▼
        GlobalHeaders sets          HttpURLConnection
        allowRestrictedHeaders      loads, reads property
              = true                    = false
                │                          │
                ▼                          ▼
        HttpURLConnection           Hutool sets property
        loads, reads property       (too late)
              = true                       │
                │                          ▼
                ▼                   allowRestrictedHeaders
        Protection OFF                   = false
                │                          │
                ▼                          ▼
        Content-Length              Content-Length
        not filtered                filtered
                │                          │
                ▼                          ▼
        Feign bug exposed           Feign bug masked
        Duplicate headers           JDK sets header
                │                          │
                ▼                          ▼
           400 Error                    Success
```

**The behavior pattern**:

- Within a single JVM lifecycle, behavior is **deterministic** (always fails or always works)
- Across different JVM starts, behavior **may differ** (depends on class loading order)
- Across different environments, behavior **may differ** (depends on initialization paths)

---

## Why Does Class Loading Order Vary?

Class loading is triggered by first use. The order depends on:

- **Which code path executes first** during startup
- **Thread scheduling** in multi-threaded initialization
- **Spring bean initialization order**
- **Lazy vs eager loading** of dependencies

Even the same code can have different initialization orders across JVM restarts. A small change in bean dependencies or startup timing can flip the order.

---

## Why Does JDK Restrict These Headers?

This stems from CVE-2010-3573, a security vulnerability from 2010. Malicious Applets could set arbitrary headers (like `Host`) via `HttpURLConnection`, bypassing same-origin policy protections. JDK 6u22 introduced the restricted headers mechanism, prohibiting user code from setting these sensitive headers by default.

`Content-Length` is restricted because a mismatch between the declared length and actual body size can cause request smuggling attacks or confuse downstream proxies.

---

## The Fix

**Root cause fix**: Upgrade Feign to 13.3 or later, which fixes the `if`/`else if` issue.

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-core</artifactId>
    <version>13.3</version>
</dependency>
```

**Alternative options**:

- If you can't upgrade Feign immediately, switch to `OkHttpClient` or `ApacheHttpClient` — they bypass `HttpURLConnection` entirely
- Consider whether Hutool's modification of JDK security defaults is acceptable for your security posture

---

## Debugging Takeaways

1. **"Fixed after restart" is not the end** → Preserve the scene, find the root cause
2. **HTTP layer issues** → tcpdump is your friend
3. **Multi-layer problems** → Peel back layers: Feign → JDK → System property → Third-party lib
4. **static final trap** → Value is set at class load time, immutable afterward
5. **Tracing property sources** → Property Hook is an effective technique

---

*References:*
- [RFC 7230 Section 3.3.2 - Content-Length](https://datatracker.ietf.org/doc/html/rfc7230#section-3.3.2)
- [Feign PR #2275 - Fix duplicate Content-Length](https://github.com/OpenFeign/feign/pull/2275)
- [JDK HttpURLConnection source](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/sun/net/www/protocol/http/HttpURLConnection.java)
- [CVE-2010-3573](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3573)
