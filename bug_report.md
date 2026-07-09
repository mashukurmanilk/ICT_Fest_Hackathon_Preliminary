# CoWork API - Bug Report & Fixes

This report outlines the bugs identified and resolved in the CoWork API implementation. The issues ranged from authentication vulnerabilities and authorization bypasses to critical race conditions and multi-tenancy data leaks.

## 1. Authentication & Session Management (`app/auth.py` & `app/routers/auth.py`)

### 1.1 Token Lifetime Calculation Error
**Bug:** The access token lifetime was being incorrectly calculated. The configuration variable `ACCESS_TOKEN_EXPIRE_MINUTES` was being multiplied by `60` inside a `timedelta(minutes=...)` constructor.
**Impact:** Access tokens were valid for 15 hours instead of the required 15 minutes (900 seconds), violating Rule 8.
**Fix:** Removed the `* 60` multiplier so the timedelta is correctly instantiated in minutes.

### 1.2 Token Revocation Targeting Wrong Claim
**Bug:** When verifying if an access token was revoked in `get_token_payload`, the code checked if the `sub` claim (the User ID) was in the `_revoked_tokens` list, rather than the `jti` (Token ID). 
**Impact:** Logging out or revoking one token would effectively blacklist the user's ID entirely, rendering all of their active sessions invalid simultaneously.
**Fix:** Changed the revocation check to look up `payload.get("jti") in _revoked_tokens`.

### 1.3 Refresh Tokens Were Not Single-Use
**Bug:** The `/auth/refresh` endpoint did not track or invalidate the presented refresh token after use. 
**Impact:** Refresh tokens could be reused indefinitely to generate new access tokens, violating Rule 8 (which states refresh tokens are single-use).
**Fix:** Added logic to extract the `jti` from the refresh token, check it against `_revoked_tokens`, and append it to the blacklist upon successful rotation.

### 1.4 Registration Conflict Status Code
**Bug:** If a user attempted to register a username that was already taken within the organization, the API returned `200 OK` with the existing user's details.
**Impact:** Violated Rule 15 and posed a minor security risk by exposing existing account details on conflict.
**Fix:** Updated the `register` endpoint to raise `409 USERNAME_TAKEN` when a duplicate username is detected.

---

## 2. Bookings & Refunds (`app/routers/bookings.py` & `app/services/refunds.py`)

### 2.1 Unauthorized Cross-User Booking Access
**Bug:** The `GET /bookings/{id}` endpoint and `POST /bookings/{id}/cancel` endpoints fetched bookings by `id` and `org_id` but failed to verify if the caller was the owner of the booking or an admin.
**Impact:** Any user could view or cancel bookings belonging to other members of their organization, violating Rule 10.
**Fix:** Added an explicit check: `if user.role != "admin" and booking.user_id != user.id:` before proceeding.

### 2.2 Double-Cancel Race Condition
**Bug:** The cancellation endpoint checked `if booking.status == "cancelled"` *before* entering the `_booking_lock` block.
**Impact:** Two concurrent cancellation requests could bypass the check, process the refund calculation twice, and generate duplicate refund entries.
**Fix:** Moved the status check *inside* the `_booking_lock` and added `db.refresh(booking)` to ensure the check operates on the latest locked state.

### 2.3 Incorrect Cancellation Refund Logic
**Bug:** The refund calculation logic was flawed. When the notice period was under 24 hours, the `elif` branch mistakenly assigned `refund_percent = 50` instead of `0`.
**Impact:** Users received a 50% refund when they were entitled to 0%, violating Rule 6.
**Fix:** Updated the final `else` branch to assign `refund_percent = 0`.

### 2.4 Cent Rounding Math
**Bug:** `refund_amount_cents` was calculated using standard float multiplication and Python's `round()`. Python uses "banker's rounding" (round half to even), which violates the business rule of "half-cents rounding up".
**Impact:** E.g., 50% of 1001 cents became 500 instead of 501.
**Fix:** Replaced float math with integer arithmetic rounding half-up: `(booking.price_cents * refund_percent + 50) // 100`.

### 2.5 Malformed Booking Response Schema
**Bug:** The `get_booking` route manually set `response["start_time"] = iso_utc(booking.created_at)`.
**Impact:** The API returned the booking creation time as the event start time, violating the API schema contract.
**Fix:** Removed the overriding line; the serializer now natively handles returning both fields correctly.

---

## 3. Data Leaks & Multi-Tenancy (`app/services/export.py`)

### 3.1 Cross-Tenant Admin Data Leak
**Bug:** The `generate_export` function for admins had a severe multi-tenancy vulnerability. If `include_all=True` and a `room_id` was provided, it delegated to `fetch_bookings_raw(db, room_id)`. This raw query fetched bookings strictly by `room_id` without verifying the room's `org_id`.
**Impact:** An admin could specify the `room_id` of a competitor's organization and successfully export all of their bookings, completely bypassing Rule 9.
**Fix:** Deleted `fetch_bookings_raw` and forced the export logic to use `_fetch_scoped(db, org_id, None, room_id)`, inherently enforcing the tenant boundary at the database query level.

---

## 4. Concurrency & Thread-Safety (`app/cache.py`, `app/services/stats.py`, et al.)

### 4.1 Dictionary Iteration Runtime Errors
**Bug:** `app/cache.py` invalidated cached reports using `for key in [k for k in _report_cache if k[0] == org_id]`. Under concurrent access, modifying the dictionary while another thread iterates over it causes a `RuntimeError`.
**Impact:** The application would crash on cache invalidation during moderate concurrent traffic.
**Fix:** Introduced `_report_lock` and `_availability_lock` (via `threading.Lock`) to serialize all cache reads and writes.

### 4.2 Lost Updates on In-Memory Stats & Limits
**Bug:** Modules like `stats.py`, `reference.py`, and `ratelimit.py` updated shared in-memory dictionaries and counters (`_stats`, `_counter`, `_buckets`) without thread synchronization. 
**Impact:** Concurrent requests would cause "lost updates" (e.g., two bookings assigned the same reference code, or rate limit checks failing to register requests).
**Fix:** Added explicit `threading.Lock` blocks around state mutations in all affected service files. In `stats.py`, the `get()` function was also updated to return a `.copy()` of the dictionary to prevent external mutations.

### 4.3 Notification Deadlock
**Bug:** `notifications.py` acquired locks in different orders. `notify_created` acquired `_email_lock` then `_audit_lock`, while `notify_cancelled` acquired `_audit_lock` then `_email_lock`.
**Impact:** This inversion created a textbook deadlock if a booking was created and another cancelled simultaneously, locking the application.
**Fix:** Refactored the methods so the locks are no longer nested, acquiring and releasing them independently.
