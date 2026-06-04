# Qobuz Authentication Issue Report

Date: April 5, 2026
Status: **PARTIALLY RESOLVED** - Login works, downloads broken

## Executive Summary

Qobuz changed their API authentication around April 3, 2026. Email/password authentication is now broken, but **token-based authentication works** for login.

| Feature | Status | Notes |
|---------|--------|-------|
| Email/Password Login | ❌ BROKEN | Returns 401 |
| Token Login (user_id + user_auth_token) | ✅ WORKS | Implemented |
| Track Downloads | ❌ BROKEN | Signature algorithm changed |

## Problem Description

All login attempts to `/api.json/0.2/user/login` return:
```json
{"code":401,"status":"error","message":"User authentication is required."}
```

Affected tools:
- qobuz-dl (https://github.com/vitiko98/qobuz-dl/issues/329)
- streamrip (https://github.com/nathom/streamrip/issues/954)

---

## Solution: Token-Based Authentication

### How to Get Your Token

1. Open https://play.qobuz.com in browser and log in
2. Press F12 → Application tab → Local Storage → https://play.qobuz.com
3. Find `localuser` → copy values:
   - `id` → user_id
   - `token` → user_auth_token

### Configure qobuz-dl

Edit `~/.config/qobuz-dl/config.ini`:
```ini
[DEFAULT]
user_id = YOUR_USER_ID
user_auth_token = YOUR_TOKEN
email = 
password = 
```

Run `qobuz-dl -sc` to verify config.

---

## Technical Investigation

### What Was Tested (All Failed)

| Test | Result | Notes |
|------|--------|-------|
| GET request | 401 | Original method |
| POST request | 401 | Tried from qobuz-api-rust |
| Add device_manufacturer_id | 401 | Tried UUID 79d511e3-9a3f-4a78-baaa-89e2d78fed83 |
| Different password formats | 401 | Plain, MD5, SHA256 all fail |
| Fresh app_id/secrets from bundle.js | 401 | Still broken |

### Key Discovery: New Signature Algorithm

The web player's JavaScript (`bundle.js`) shows:

```javascript
// New signature uses SHA256 + HKDF
this._raw_sig = object + method + params + timestamp + appSecret
this._requestIdentifier = object + method + params + timestamp + rng.initialization()
sha256(_raw_sig, _requestIdentifier)
```

This is **significantly more complex** than the old MD5-based signature:
- Old: `md5(method + endpoint + params + timestamp + secret)`
- New: `sha256_hkdf(method + endpoint + params + timestamp + rng_value)`

The `rng.prototype.initialization()` generates a dynamic value in the browser using Web Crypto API:
- HKDF (HMAC-based Key Derivation Function)
- Base64-encoded seeds from Qobuz servers
- Crypto.subtle.importKey for key derivation

### Root Cause

Qobuz updated their API security (April 2026):
1. Email/password auth → **DEPRECATED** (returns 401)
2. Token auth → **WORKS** for login, metadata queries
3. Request signatures → **CHANGED** from MD5 to SHA256+HKDF

---

## Implementation (Branch: bug/newauth)

### Changes Made

1. **qopy.py**:
   - Added `skip_auth` parameter to `Client.__init__`
   - Added `auth_with_token(user_id, user_auth_token)` method

2. **core.py**:
   - Added `initialize_client_with_token()` method

3. **cli.py**:
   - Updated `_reset_config()` to support token-based setup
   - Reads `user_id` and `user_auth_token` from config
   - Auto-detects token vs password auth

### Verified Working

```
$ python3 -c "from qobuz_dl import main; main()" dl <url>
Logging...
Membership: Studio
Set max quality: 6 - 16 bit, 44.1kHz
```

Login succeeds! "Membership: Studio" confirms token authentication works.

---

## Remaining Issue: Download Signatures

The `track/getFileUrl` endpoint fails with:
```
400: Invalid Request Signature parameter
```

**Why:** The signature generation uses browser-based crypto that can't be easily replicated in Python.

### Options to Fix

1. **Reverse-engineer HKDF implementation** - Complex, requires extracting dynamic `rng` values
2. **Browser automation** - Use Selenium/Playwright to get signed URLs from real browser
3. **Wait for upstream** - Other projects may solve this first

### Signature Details Found

From bundle.js (play.qobuz.com):
```
raw_sig = object + method + sortedParams + timestamp + appSecret
requestIdentifier = object + method + sortedParams + timestamp + rngValue
signature = sha256(raw_sig, requestIdentifier)
```

Where `rngValue` comes from:
```javascript
window.rng.prototype.initialization()
// Uses HKDF with base64-encoded seeds
// Crypto: window.crypto.subtle.importKey("raw", seed, {name: "HKDF"}, ...)
```

---

## Related Projects Status

| Project | Token Auth | Downloads | Notes |
|---------|------------|-----------|-------|
| qobuz-dl (this fix) | ✅ WORKS | ❌ FAILS | Partial fix |
| qobuz-api-rust | ✅ WORKS | ✅ WORKS? | Only tested token auth |
| streamrip | ✅ WORKS | ❌ FAILS | IncompleteRead errors |
| QobuzDownloaderX-MOD | ✅ WORKS | ✅ WORKS | C#, scrapes web player |

---

## Commit History

```
76b56df Implement token-based authentication (partial fix)
33014eb docs: update auth issue with extended investigation findings
3fcd895 docs: update auth issue with investigation results
```

---

## References

- Qobuz API docs: https://www.qobuz.com/api.json/0.2/
- qobuz-api-rust: https://docs.rs/qobuz-api-rust/latest/qobuz_api_rust/
- QobuzDownloaderX Token Guide: https://github.com/ImAiiR/QobuzDownloaderX/wiki/Logging-In-(The-Alternate-Way)
- Lyrion Forum Thread: https://forums.lyrion.org/forum/user-forums/3rd-party-software/1743798-qobus-401-authentication-failure
