---
hidden: true
---

# Signing requests in Postman

Use this **pre-request script** to call the Blinx Merchant API from Postman without signing requests by hand. Paste it into the **Pre-request Script** tab of a request — or of a folder/collection to cover every request beneath it — and it signs each outgoing request and injects the required headers.

### Setup

1. Download the Postman collection below
2. Define two variables (environment, collection, or globals):
   * `api_key` — your tenant's raw **apiKey** (sent as `x-api-key`).
   * `api_secret` — your tenant's raw **apiSecret** (the HMAC key, never sent).
3. The script will add `x-api-key`, `x-timestamp`, and `x-signature` automatically to any request

### Postman collection

### How it works

* The signature is `HMAC-SHA256(api_secret, ${timestamp}.${METHOD}.${path}.${sha256hex(body)})`.
* Method, path (+ query), and the raw body are read straight from the request.
* Dynamic variables (e.g. `{{$guid}}`) in the **body** are resolved **once** and pinned back onto the request, so the exact bytes signed are the exact bytes Postman sends. Without this, a re-resolved value would break the signature.
* Only `raw` bodies are signed; other body modes hash as empty.

> Requires the `crypto-js` library, which Postman's sandbox bundles by default.

### Script

```js
// ─────────────────────────────────────────────────────────────────────────────
// Blinx payments — Postman PRE-REQUEST script
//
// Paste this into the "Pre-request Script" tab of the request (or a folder /
// collection, to cover every request under it). It signs the outgoing request
// and injects the x-api-key / x-timestamp / x-signature headers that the
// authenticateSeller middleware expects.
//
// Requires two variables (environment, collection, or globals):
//   api_key     the tenant's raw apiKey      -> sent as x-api-key
//   api_secret  the tenant's raw apiSecret   -> HMAC key (never sent)
//
// Signature = hex HMAC-SHA256(api_secret, `${timestamp}.${METHOD}.${path}.${sha256hex(body)}`)
//   timestamp : unix epoch MILLISECONDS (server allows ±5 min)
//   METHOD    : upper-cased HTTP method   (taken from the request)
//   path      : request target, path + query, exactly as sent
//   body      : raw request body bytes    (empty for bodyless requests)
//
// NOTE: dynamic variables like {{$guid}}/{{$randomUUID}} in the BODY produce a
// NEW value every time they're resolved. The script resolves the body ONCE and
// writes it back onto the request, so the value it signs is the exact value
// Postman sends. (Side effect: the sent body shows those variables already
// substituted — that is intentional and required for the signature to match.)
// ─────────────────────────────────────────────────────────────────────────────

const CryptoJS = require('crypto-js');

const apiKey = pm.variables.get('api_key');
const apiSecret = pm.variables.get('api_secret');
if (!apiKey || !apiSecret) {
  throw new Error('Set the "api_key" and "api_secret" variables before sending.');
}

// Method — from the request, upper-cased.
const method = pm.request.method.toUpperCase();

// Path + query, exactly as sent. getPathWithQuery() returns "/path?query" (no
// host). replaceIn resolves any {{vars}} used in the URL so it matches the wire.
const url = pm.request.url;
let target = typeof url.getPathWithQuery === 'function'
  ? url.getPathWithQuery()
  : (function () {
      const p = url.getPath();
      const q = url.getQueryString();
      return q ? p + '?' + q : p;
    })();
target = pm.variables.replaceIn(target);
if (target.charAt(0) !== '/') target = '/' + target;

// Raw body bytes. Resolve {{vars}} ONCE, then pin the resolved text back onto the
// request so the bytes we sign are the exact bytes Postman sends. Without the
// write-back, dynamic vars (e.g. {{$guid}} in "reference") would re-resolve to a
// DIFFERENT value at send time and the signature would fail. Only "raw" bodies
// are signed; anything else (formdata, urlencoded) hashes as empty.
let body = '';
if (pm.request.body && pm.request.body.mode === 'raw' && pm.request.body.raw) {
  body = pm.variables.replaceIn(pm.request.body.raw.toString());
  pm.request.body.update({ mode: 'raw', raw: body });
}

const timestamp = Date.now().toString();
const bodyHash = CryptoJS.SHA256(body).toString(CryptoJS.enc.Hex);
const canonical = timestamp + '.' + method + '.' + target + '.' + bodyHash;
const signature = CryptoJS.HmacSHA256(canonical, apiSecret).toString(CryptoJS.enc.Hex);

// upsert = add or overwrite, so re-sending doesn't stack duplicate headers.
pm.request.headers.upsert({ key: 'x-api-key', value: apiKey });
pm.request.headers.upsert({ key: 'x-timestamp', value: timestamp });
pm.request.headers.upsert({ key: 'x-signature', value: signature });

// Uncomment to debug (never logs the secret):
// console.log('canonical:', canonical);
// console.log('x-signature:', signature);
```

_Updated: 27 August 2026 12:46_
