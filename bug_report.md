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

## Bug 5 — GET /bookings/{id} overwrites start_time with created_at  (Easy)
- **File / line:** `app/routers/bookings.py:166`
- **Rule:** Booking response contract — `start_time` is the booking's start time.
- **Symptom:** `GET /bookings/{id}` returned `start_time` equal to `created_at` instead of the actual booking start.
- **Why broken:** After `serialize_booking()` produced the correct payload, a stray line reassigned `response["start_time"] = iso_utc(booking.created_at)`, clobbering the correct value. Only the detail endpoint had this line; create/list use the same serializer correctly.
- **Fix:** Deleted the overwriting line. Verified: GET returns the real start (`14:00Z`), distinct from `created_at`, with all other fields and the `refunds` array intact.

## Bug 6 — Back-to-back bookings rejected as conflict  (Medium)
- **File / line:** `app/routers/bookings.py:50`
- **Rule:** #3 — overlap iff `existing.start < new.end AND new.start < existing.end`; back-to-back (one ends exactly when the other starts) is allowed.
- **Symptom:** A booking starting exactly when an existing one ends (e.g. 12:00–14:00 after 10:00–12:00) was wrongly rejected with `409 ROOM_CONFLICT`.
- **Why broken:** The overlap test used `<=` on both bounds (`b.start_time <= end and start <= b.end_time`), so touching endpoints counted as overlap.
- **Fix:** Strict `<` on both bounds (`b.start_time < end and start < b.end_time`), matching the spec's overlap condition. Verified: back-to-back on both sides → 201; exact-same, partial, contained, and straddling overlaps → 409.

## Bug 7 — Minimum duration and end>start not enforced  (Medium)
- **File / line:** `app/routers/bookings.py:93`
- **Rule:** #2 — duration is whole hours, min 1, max 8; `end_time` strictly after `start_time`.
- **Symptom:** Zero-duration bookings (`end == start`) were accepted (201, price 0), and negative-duration bookings (`end < start`) were accepted with a negative price.
- **Why broken:** The check only tested `duration_hours > MAX_DURATION_HOURS`. Zero passed the whole-hour test (`0 != int(0)` is False) and the upper-bound test; negatives likewise (`-2 > 8` is False). The defined `MIN_DURATION_HOURS = 1` constant was never used.
- **Fix:** Range-check both bounds: `duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS`. This also enforces `end > start`, since any `end <= start` yields duration ≤ 0 < min. Verified: 0h → 400, negative → 400, 1h → 201, 8h → 201, 9h → 400.

## Bug 8 — Booking allowed a 5-minute past-start grace window  (Medium)
- **File / line:** `app/routers/bookings.py:86`
- **Rule:** #2 — `start_time` must be strictly in the future at request time; no grace window of any size.
- **Symptom:** Bookings with a `start_time` up to 5 minutes in the past were accepted (e.g. 100s in the past → 201).
- **Why broken:** The guard was `start <= now - timedelta(seconds=300)`, rejecting only starts more than 5 minutes old and silently allowing the `(now-300s, now]` window.
- **Fix:** `start <= now` — rejects any non-future start, including exactly `now`. Verified: 100s-past → 400, 10s-past → 400, valid future → 201. (`timedelta` import retained; still used by the quota window.)

## Bug 9 — Refund percentage tiers wrong  (Medium)
- **File / line:** `app/routers/bookings.py:200-206`
- **Rule:** #6 — notice ≥48h → 100%, 24h ≤ notice <48h → 50%, notice <24h → 0%.
- **Symptom:** A cancel with <24h notice refunded 50% (should be 0%); a cancel with exactly 48h notice refunded 50% (should be 100%).
- **Why broken:** Two defects — (a) the top tier used `int(notice.total_seconds() // 3600) > 48`, which truncates to whole hours and uses strict `>`, so exactly-48h (and e.g. 48h30m) fell through to 50%; (b) the `else` branch (notice <24h) returned `50` instead of `0`.
- **Fix:** Compare the `timedelta` directly — `notice >= timedelta(hours=48)` → 100, `>= timedelta(hours=24)` → 50, else → 0 (removed the truncating `notice_hours` line). Verified across boundaries: 72h/48h+1m → 100, 48h−1m/36h/24h+1m → 50, 24h−1m/12h → 0.

## Bug 10 — Refund amount rounding wrong and inconsistent  (Medium)
- **File / line:** `app/services/refunds.py:15` (stored) and `app/routers/bookings.py:208` (response).
- **Rule:** #6 — refund = percentage of `price_cents` rounded to nearest cent, half-cents rounding UP (50% of 1001 = 501); the response amount must equal the stored RefundLog amount.
- **Symptom:** 50% of 1001 produced 500 instead of 501. Two independent code paths used two different wrong methods, risking divergence between the stored log and the response.
- **Why broken:** RefundLog used `int(refund_dollars * 100)` (truncates toward zero, plus float round-trip through dollars); the response used `round(...)` (banker's rounding, half-to-even → 500.5 rounds to 500).
- **Fix:** Compute once with exact integer half-up in `log_refund`: `amount_cents = (price_cents * percent + 50) // 100`, and have the cancel response read the stored `refund_entry.amount_cents` (single source of truth). Verified: 1001@50% → 501, 1000@50% → 500, @100% → full, @0% → 0, 333@50% → 167; response equals RefundLog in every case.

## Bug 11 — Report and availability caches not fully invalidated on write  (Medium)
- **File / line:** `app/routers/bookings.py:122` (create) and `:216` (cancel).
- **Rule:** #12 (usage report reflects current state immediately) and #13 (availability reflects current state immediately).
- **Symptom:** Two symmetric staleness bugs — after creating a booking the usage report was stale (missing the new booking); after cancelling a booking the room's availability was stale (still showed the cancelled slot as busy).
- **Why broken:** Invalidation was half-wired. `create_booking` invalidated availability but not the report; `cancel_booking` invalidated the report but not availability. Each write path flushed only one of the two caches.
- **Fix:** Added the missing mirror call to each path — `cache.invalidate_report(user.org_id)` on create and `cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())` on cancel. Verified: report +1 immediately after create, availability → 0 immediately after cancel, report → 0 after cancel.

## Bug 12 — Pagination: wrong order, offset, and ignored limit  (Hard)
- **File / line:** `app/routers/bookings.py:137-139`
- **Rule:** #11 — items sorted ascending `start_time` (ties ascending `id`); page N/limit L returns `[(N−1)·L, N·L)`; sequential pages never skip or repeat; response includes `total`.
- **Symptom:** Listing returned newest-first, page 1 skipped the first page of results, and the `limit` query param was ignored (always up to 10 rows).
- **Why broken:** Three stacked defects in one query — `order_by(start_time.desc(), …)` (wrong direction), `.offset(page * limit)` (page 1 skipped the first `limit` rows; should be `(page-1)*limit`), and `.limit(10)` hardcoded instead of `.limit(limit)`.
- **Fix:** `order_by(Booking.start_time.asc(), Booking.id.asc())`, `.offset((page - 1) * limit)`, `.limit(limit)`. `page`/`limit` ranges are already validated by FastAPI `Query` bounds. Verified with 5 bookings: page1/2/3 at limit 2 return the correct disjoint windows, no skip/repeat (all 5 ids exactly once), limit honored and echoed, default page1/limit10 returns all ascending.
