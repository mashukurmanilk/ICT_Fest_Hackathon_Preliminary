# Bug Report — CoWork API

## Bug 1: Access Token Lifetime Inflated (Easy)
- **File:** `app/auth.py`, line 50
- **What:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` — `ACCESS_TOKEN_EXPIRE_MINUTES` is 15, so `15 * 60 = 900` minutes = 54,000 seconds. The spec requires access tokens to expire in exactly 900 **seconds** (15 minutes).
- **Why broken:** All access tokens live 60× longer than intended, undermining security.
- **Fix:** Changed to `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)` → 15 minutes = 900 seconds.

## Bug 2: Token Revocation Checks `sub` Instead of `jti` (Easy)
- **File:** `app/auth.py`, line 97
- **What:** `payload.get("sub") in _revoked_tokens` — `_revoked_tokens` stores JTI values (from `revoke_access_token`), but the check compares the user ID (`sub`) against the JTI set.
- **Why broken:** Logout never actually invalidates the token; subsequent use of a revoked token still succeeds.
- **Fix:** Changed to `payload.get("jti") in _revoked_tokens`.

## Bug 3: Timezone-Aware Input Not Converted to UTC (Easy)
- **File:** `app/timeutils.py`, line 13
- **What:** `dt = dt.replace(tzinfo=None)` strips the timezone info but does NOT convert to UTC first. For example, `2026-07-10T10:00:00+05:00` should become `2026-07-10T05:00:00` UTC but instead remains `10:00:00`.
- **Why broken:** Any datetime input with a non-UTC timezone offset is stored at the wrong time, causing incorrect scheduling, overlap checks, and reports.
- **Fix:** Changed to `dt = dt.astimezone(timezone.utc).replace(tzinfo=None)`.

## Bug 4: Duplicate Username Returns 200 Instead of 409 (Easy)
- **File:** `app/routers/auth.py`, lines 38–43
- **What:** When a user with the same username already exists within the org, the code returns the existing user's data with a 200 status instead of raising `409 USERNAME_TAKEN`.
- **Why broken:** The spec requires duplicate usernames to be rejected with `409 USERNAME_TAKEN`. Returning 200 silently succeeds without creating a new user, misleading the client.
- **Fix:** Replaced the return statement with `raise AppError(409, "USERNAME_TAKEN", "Username already taken")`.

## Bug 5: Refresh Token Not Invalidated After Use (Medium)
- **File:** `app/routers/auth.py`, lines 81–93
- **What:** The `/auth/refresh` endpoint does not revoke the old refresh token's JTI after use. Per the spec, refresh tokens are single-use — reusing one should return `401`.
- **Why broken:** A stolen refresh token can be used indefinitely for new access tokens.
- **Fix:** Added JTI tracking: after decoding the refresh token, check if its JTI is in `_revoked_tokens`; if so, reject with 401. Otherwise, add the JTI to `_revoked_tokens` before issuing new tokens.

## Bug 6: `start_time` Allows Past Bookings (5-Minute Grace Window) (Easy)
- **File:** `app/routers/bookings.py`, line 86
- **What:** `if start <= now - timedelta(seconds=300)` allows start times up to 5 minutes in the past. The spec explicitly says "start_time must be strictly in the future at request time — **no grace window**."
- **Why broken:** Bookings can be created for the past or for "right now."
- **Fix:** Changed to `if start <= now`.

## Bug 7: Missing Minimum Duration and end ≤ start Checks (Easy)
- **File:** `app/routers/bookings.py`, lines 89–94
- **What:** Only checks `duration_hours > MAX_DURATION_HOURS` (8) but never checks `duration_hours < MIN_DURATION_HOURS` (1). Also missing the `end_time <= start_time` check.
- **Why broken:** Zero-hour or negative-duration bookings could be created; also `end_time == start_time` doesn't error but should.
- **Fix:** Added `if end <= start` check and `if duration_hours < MIN_DURATION_HOURS` check.

## Bug 8: Double-Booking Overlap Uses `<=` Instead of `<` (Medium)
- **File:** `app/routers/bookings.py`, line 50
- **What:** `if b.start_time <= end and start <= b.end_time` — uses `<=` which incorrectly blocks back-to-back bookings (where one ends exactly when the other starts).
- **Why broken:** The spec says overlap is `existing.start < new.end AND new.start < existing.end`. Back-to-back bookings should be allowed but are rejected.
- **Fix:** Changed to `if b.start_time < end and start < b.end_time`.

## Bug 9: Pagination Has 3 Bugs (Medium)
- **File:** `app/routers/bookings.py`, lines 137–139
- **What:**
  1. `Booking.start_time.desc()` — sorts descending; spec says ascending.
  2. `.offset(page * limit)` — should be `(page - 1) * limit`. Page 1 should show items 0..limit-1, not skip the first `limit` items.
  3. `.limit(10)` — hardcoded 10 instead of using the `limit` parameter.
- **Why broken:** Bookings are in reverse order, page 1 skips the first batch entirely, and the user's requested page size is ignored.
- **Fix:** Changed to `.order_by(Booking.start_time.asc(), Booking.id.asc()).offset((page - 1) * limit).limit(limit)`.

## Bug 10: `get_booking` Overwrites `start_time` with `created_at` (Easy)
- **File:** `app/routers/bookings.py`, line 166
- **What:** `response["start_time"] = iso_utc(booking.created_at)` — after serializing the booking (which correctly sets `start_time`), this line overwrites it with `created_at`.
- **Why broken:** The booking detail endpoint returns the wrong `start_time`, breaking downstream logic.
- **Fix:** Removed this line entirely.

## Bug 11: Cancellation Refund Policy — Wrong Thresholds (Medium)
- **File:** `app/routers/bookings.py`, lines 200–206
- **What:**
  1. `notice_hours = int(notice.total_seconds() // 3600)` then `if notice_hours > 48` — truncating to int and using `>` means notice of exactly 48 hours doesn't get 100% (spec says `≥ 48` → 100%).
  2. The `else` branch (line 206) gives `refund_percent = 50` for `< 24h` notice; spec says 0%.
- **Why broken:** Customers get incorrect refund amounts. Specifically, < 24h notice gives 50% instead of 0%.
- **Fix:** Changed to use `timedelta` comparison: `notice >= timedelta(hours=48)` → 100%, `notice >= timedelta(hours=24)` → 50%, else → 0%.

## Bug 12: Refund Amount Rounding Incorrect (Medium)
- **File:** `app/services/refunds.py`, lines 15–17
- **What:** Converts to dollars (float), multiplies, then truncates with `int()`. The spec says "round to the nearest cent, half-cents rounding **up**." For example, 50% of 1001 cents should be 501, not 500.
- **Why broken:** Rounding errors cause refund amounts to differ from spec.
- **Fix:** Changed to integer math: `amount_cents = (booking.price_cents * percent + 50) // 100` which correctly rounds half-up.

## Bug 13: Member Visibility Not Enforced on `get_booking` (Easy)
- **File:** `app/routers/bookings.py`, lines 156–163
- **What:** The `get_booking` endpoint checks org-level access but does not check whether a member is requesting another member's booking. The spec says members may only read their own bookings.
- **Why broken:** Members can view any booking in their organization, violating booking visibility rules.
- **Fix:** Added `if user.role != "admin" and booking.user_id != user.id: raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")`.

## Bug 14: Deadlock in Notifications — Nested Locks in Opposite Order (Hard)
- **File:** `app/services/notifications.py`, lines 24–35
- **What:** `notify_created` acquires `_email_lock` → then `_audit_lock`. `notify_cancelled` acquires `_audit_lock` → then `_email_lock`. Classic ABBA deadlock: if a create and cancel happen concurrently, each holds one lock and waits for the other's, hanging the service forever.
- **Why broken:** The service hangs under concurrent booking creation and cancellation, violating the liveness requirement (Rule 16).
- **Fix:** Changed both functions to acquire locks in the same order (email first, audit second) without nesting.

## Bug 15: Race Condition on Reference Code Counter (Hard)
- **File:** `app/services/reference.py`, lines 17–21
- **What:** The counter is read, then a `time.sleep(0.12)` occurs, then the counter is incremented. Concurrent requests can read the same value before either writes back, producing duplicate reference codes.
- **Why broken:** Violates Rule 7: "Every booking's `reference_code` is unique, including under concurrent creation."
- **Fix:** Added a `threading.Lock()` around the counter read-and-increment, removed the artificial sleep.

## Bug 16: Race Condition on Stats — Read-Sleep-Write (Hard)
- **File:** `app/services/stats.py`, lines 15–26
- **What:** Same read-sleep-write pattern as reference codes. `record_create` and `record_cancel` read current stats, sleep 0.1s, then write back. Concurrent operations lose increments/decrements.
- **Why broken:** Stats become inconsistent with actual bookings (Rule 14).
- **Fix:** Added a `threading.Lock()` around all stats mutations, removed the sleep.

## Bug 17: Race Condition on Rate Limit Buckets (Hard)
- **File:** `app/services/ratelimit.py`, lines 18–26
- **What:** Same pattern: reads the bucket, sleeps, then appends and writes back. Concurrent requests can bypass the rate limit.
- **Why broken:** More than 20 requests per 60 seconds can slip through (Rule 5).
- **Fix:** Added a `threading.Lock()` around the entire bucket operation, removed the sleep.

## Bug 18: No Concurrency Protection for Double-Booking & Quota (Hard)
- **File:** `app/routers/bookings.py`, lines 100–103
- **What:** Conflict check and quota check are done without serialization. Two concurrent requests for the same slot can both pass the conflict check and create overlapping bookings.
- **Why broken:** Violates Rules 3 and 4: conflict and quota "must hold under concurrent requests."
- **Fix:** Added an application-level `threading.Lock()` (`_booking_lock`) wrapping the conflict check, quota check, and insert within `create_booking`.

## Bug 19: Usage Report Cache Not Invalidated on Booking Create (Medium)
- **File:** `app/routers/bookings.py`, line 121 (area)
- **What:** After creating a booking, `cache.invalidate_availability()` is called but `cache.invalidate_report()` is NOT. The usage report serves stale cached data.
- **Why broken:** Violates Rule 12: the report "must reflect the current state immediately."
- **Fix:** Added `cache.invalidate_report(user.org_id)` after booking creation.

## Bug 20: Availability Cache Not Invalidated on Cancel (Medium)
- **File:** `app/routers/bookings.py`, lines 216–218
- **What:** On cancellation, the report cache is invalidated but the availability cache is NOT. A cancelled booking still appears as a "busy" interval.
- **Why broken:** Violates Rule 13: availability must "reflect the current state immediately."
- **Fix:** Added `cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())` in the cancel path.

## Bug 21: Refund Amount Mismatch Between Response and RefundLog (Medium)
- **File:** `app/routers/bookings.py`, lines 208–210
- **What:** The cancel response computes `refund_amount_cents` independently using `round()`, while `log_refund` computes its own value using different math. The two can differ.
- **Why broken:** Violates Rule 6: "the amount returned by the cancel response must equal the amount stored in the RefundLog."
- **Fix:** Changed the cancel response to use `refund_entry.amount_cents` (the actual stored value) instead of an independently computed value.

## Bug 22: No Concurrency Protection for Cancel (Hard)
- **File:** `app/routers/bookings.py`, lines 195–214
- **What:** Two concurrent cancel requests for the same booking can both pass the `status == "cancelled"` check before either commits, causing duplicate RefundLog entries and potential inconsistency.
- **Why broken:** Violates Rule 6: cancellation "must hold under concurrent cancel requests for the same booking."
- **Fix:** The `_booking_lock` added in Bug 18 is also used in `cancel_booking`, with a `db.refresh(booking)` under the lock to re-check status.

## Bug 23: Cross-Tenant Admin Data Leak (Hard)
- **File:** `app/services/export.py`, lines 21-25
- **What:** The `generate_export` function for admins had a severe multi-tenancy vulnerability. If `include_all=True` and a `room_id` was provided, it delegated to `fetch_bookings_raw(db, room_id)`. This raw query fetched bookings strictly by `room_id` without verifying the room's `org_id`.
- **Why broken:** An admin could specify the `room_id` of a competitor's organization and successfully export all of their bookings, completely bypassing Rule 9.
- **Fix:** Deleted `fetch_bookings_raw` and forced the export logic to use `_fetch_scoped(db, org_id, None, room_id)`, inherently enforcing the tenant boundary at the database query level.

## Bug 24: Dictionary Iteration Runtime Errors in Cache (Medium)
- **File:** `app/cache.py`, lines 40-52
- **What:** Invalidation methods iterated over the dictionary directly via list comprehensions like `[k for k in _report_cache if k[0] == org_id]`. Under concurrent access, modifying the dictionary while another thread iterates over it causes a `RuntimeError`.
- **Why broken:** The application crashes on cache invalidation during moderate concurrent traffic.
- **Fix:** Introduced `_report_lock` and `_availability_lock` (via `threading.Lock`) to serialize all reads and writes to the cache dictionaries.
