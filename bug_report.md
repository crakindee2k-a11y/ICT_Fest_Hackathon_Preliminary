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

## Bug 2 — Logout does not revoke the token  (Medium)
- **File / line:** `app/auth.py:97`
- **Rule:** #8 — logout immediately invalidates the presented access token; subsequent use → 401.
- **Symptom:** After `POST /auth/logout`, the same access token still authorized requests (GET /rooms returned 200 instead of 401).
- **Why broken:** `revoke_access_token` stores the token's `jti` into `_revoked_tokens`, but `get_token_payload` checked `payload.get("sub")` (user id) against that set. `jti` ≠ `sub`, so the membership test never matched and revocation never fired.
- **Fix:** Check `payload.get("jti")` instead of `sub`. Revokes exactly the presented token (per-`jti`), not all of the user's tokens. Verified: logged-out token → 401; a second independent token for the same user still → 200 (no over-revocation).

## Bug 3 — Refresh token not single-use  (Medium)
- **File / line:** `app/routers/auth.py:81` (refresh endpoint); helper added at `app/auth.py:88`.
- **Rule:** #8 — refresh tokens are single-use; `POST /auth/refresh` rotates the pair and invalidates the presented refresh token (reuse → 401).
- **Symptom:** The same refresh token could be replayed indefinitely — first and second `POST /auth/refresh` both returned 200.
- **Why broken:** `refresh()` issued a new token pair but never recorded the presented refresh token as consumed, so no reuse check ever failed.
- **Fix:** Reuse the existing `_revoked_tokens` mechanism: added `is_token_revoked(payload)` helper in `auth.py`; in `refresh()`, reject a presented refresh token whose `jti` is already revoked (401) and revoke its `jti` on successful use. Verified: refresh #1 → 200, replay of same token → 401, chained new token also single-use, access token presented to /refresh → 401 (wrong type).

## Bug 4 — Input UTC offset stripped instead of converted  (Medium)
- **File / line:** `app/timeutils.py:13`
- **Rule:** #1 — input datetimes carrying a UTC offset are converted to UTC before storage/comparison; naive input treated as UTC.
- **Symptom:** An offset-bearing input like `2026-08-01T14:00:00+05:00` (= 09:00Z) was stored/returned as `14:00Z` — a 5-hour error corrupting pricing windows, conflict checks, availability and quota.
- **Why broken:** `dt.replace(tzinfo=None)` discards the offset without shifting the clock, contradicting the function's own docstring. The wall-clock time was kept and just relabelled UTC.
- **Fix:** `dt.astimezone(timezone.utc).replace(tzinfo=None)` — `astimezone` shifts to the same UTC instant, then tzinfo is dropped for naive-UTC storage (`timezone` was already imported). Verified: `+05:00 14:00 → 09:00Z`, `-03:00 09:00 → 12:00Z`, `Z` and naive inputs unchanged.
