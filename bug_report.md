# CoWork API — Bug Report

Fixes applied to the CoWork booking API. Each bug maps to a numbered business
rule in the README. All fixes verified live against the API surface (HTTP
request → captured response), not just static reads.

## Bug 1 — Access token lifetime wrong  (Medium)
- **File / line:** `app/auth.py:50`
- **Rule:** #8 — access token `exp − iat` must be exactly 900 seconds.
- **Symptom:** Issued access tokens had `exp − iat = 54000` (15 hours) instead of 900s (15 min).
- **Why broken:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` multiplied the 15-minute config by an extra 60, yielding 900 *minutes*. The refresh-token path had no such stray factor.
- **Fix:** Dropped the `* 60`, so lifetime = `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)`. Verified: access exp−iat = 900, refresh unchanged at 604800 (7d).
