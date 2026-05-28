# PickleFlow – Business Rules & Edge Cases

This file is the authoritative reference for all booking rules, constraints, and edge-case
handling logic. Load this whenever implementing or reviewing Booking Engine, Check-in, or
Payment logic.

---

## 1. Booking Rules

### 1.1 Slot Structure
- All bookings are exactly **1 hour** — no partial hours, no multi-hour single bookings.
- Multiple bookings per player per day are allowed, as long as they **do not overlap**.
- Bookings must start and end within facility operating hours.

### 1.2 Advance Booking Window
- Players may not book more than `maxAdvanceBookingDays` days in advance (configurable).
- Minimum notice: Online bookings require ≥ `minBookingNoticeMinutes` (default: 30 min) before slot start.
- Walk-in bookings by Staff have no minimum notice restriction.

### 1.3 Conflict Prevention
- A court slot can only hold **one confirmed or pending_payment booking** at a time.
- Slot reservation must be done **atomically** (use DB transactions or optimistic locking)
  to prevent race conditions from concurrent booking attempts.

### 1.4 Recurring Bookings
- Players may create recurring bookings (e.g., every Monday at 8:00 AM).
- System creates individual booking records for each occurrence.
- Each occurrence is independently cancellable.
- If a future occurrence conflicts with an existing booking, that occurrence is skipped
  or flagged — it does not block the entire recurring series.

### 1.5 Waitlist
- When a slot is fully booked, players may join the waitlist for that slot.
- If a confirmed booking is cancelled, the next waitlisted player is notified.
- Waitlisted players have a configurable window to confirm and pay before the slot is
  offered to the next person on the list.

---

## 2. Payment Rules

### 2.1 Online Bookings
- Booking is created in `PENDING_PAYMENT` status immediately.
- Slot is **temporarily reserved** for the duration of `paymentTimeoutMinutes`.
- If payment is not received before timeout → booking is automatically cancelled and slot released.
- Booking moves to `CONFIRMED` only after GCash payment webhook confirms success.

### 2.2 Walk-in Bookings (Staff only)
- Staff creates the booking AND records payment in the same workflow.
- Walk-in bookings go directly to `CONFIRMED` — no `PENDING_PAYMENT` state.
- Accepted methods: Cash or GCash (staff enters reference number manually).

### 2.3 Refunds
- **Refunds are always manual.** No automatic refund processing exists.
- Only `Admin` or authorized `Staff` may initiate a refund.
- Refund processing is logged to the audit trail.
- Refund eligibility depends on the rain policy and cancellation policy configured by Admin.

---

## 3. Check-in Rules

### 3.1 Self Check-in (Player)
- Player scans their booking QR code at the court kiosk or staff terminal.
- System verifies: booking is `CONFIRMED`, it is the correct date/time slot, and player identity.
- On success: booking status → `CHECKED_IN`, WebSocket event emitted.

### 3.2 Manual Check-in (Staff)
- Staff can check in a player using:
  - Booking reference number
  - Player mobile number
  - QR code scan
- Same validation rules apply as self check-in.

### 3.3 Check-in Timing
- Check-in is only allowed during the booking's time slot (within configurable early window if set).
- A booking that has already been checked in cannot be checked in again (`CHECKIN_ALREADY_DONE`).

---

## 4. No-show Detection

### 4.1 Trigger
- A scheduled job runs at regular intervals and calls `POST /operations/no-show/process`.
- For each `CONFIRMED` booking whose slot start time has passed by more than `gracePeriodMinutes`
  and has **not** been checked in → mark as `NO_SHOW`.

### 4.2 Grace Period Logic (Critical Edge Case)
- **Grace period applies ONLY when the next slot on the same court is FREE (not confirmed/pending).**
- If the next slot is already confirmed → grace period is **zero**. No-show is immediate.
- This prevents chained delays that block a paying player in the next slot.

### 4.3 After No-show
- Court status updated to `AVAILABLE`.
- Booking status → `NO_SHOW`.
- Real-time broadcast: `no_show_detected`.
- If there are waitlisted players for this slot, notify them.

---

## 5. Court Release Rules

### 5.1 Normal Completion
- When a `CHECKED_IN` session ends (at the 1-hour mark), status → `COMPLETED`, court → `AVAILABLE`.

### 5.2 Strict End Time Enforcement
- If the **next slot is booked** and the current session has not ended → immediately enforce court
  release at the scheduled end time. Zero grace period.
- Staff uses `POST /operations/court/release` if player refuses to leave.

### 5.3 Overdue Sessions
- If a session runs past its end time AND no next booking exists: court status → `OVERDUE` (🔴 Red).
- Staff can manually release using the dashboard.

---

## 6. Court Closure Rules

### 6.1 Closure Warning
- Before closing a court, the system must check for existing `CONFIRMED` or `PENDING_PAYMENT`
  bookings in the future on that court.
- If any exist → return a `warnings` array in the response listing affected bookings.
- Admin must acknowledge and decide how to handle them (cancel, rebook, etc.).
- The closure itself is still allowed — warnings are informational, not blockers.

### 6.2 Court Statuses
- `AVAILABLE` — bookable, no active session
- `OCCUPIED` — active checked-in session
- `OVERDUE` — session past end time, needs manual release
- `CLOSED` — explicitly closed by Admin/Staff (with optional scheduled reopen time)
- `MAINTENANCE` — under maintenance (subset of CLOSED, with maintenance-specific reason)

---

## 7. Rain Policy

Admin selects one of three predefined rain policy options. The chosen policy determines:
- Whether bookings are automatically cancelled/rescheduled during rain
- Whether players receive credits or refunds
- What notification is sent out

The three options are defined by the facility owner and stored in the Admin config. The system
sends a rain policy announcement notification when triggered by Admin.

---

## 8. Cancellation Policy

- Players can cancel their own bookings (subject to the configured cancellation window).
- Staff can cancel any booking.
- On cancellation:
  - Booking status → `CANCELLED`
  - Court slot → released
  - If `CONFIRMED` with payment: refund eligibility evaluated per rain/cancellation policy
  - Waitlisted players notified if applicable
  - Cancellation notification sent to player

---

## 9. Audit Trail Requirements

The following actions **must** be written to the audit log:

| Action                        | Who can perform          |
|-------------------------------|--------------------------|
| Booking creation              | Player, Staff            |
| Booking cancellation          | Player, Staff, System    |
| Booking status change         | Staff, System            |
| Payment recording             | Player (via GCash), Staff|
| Refund processing             | Admin, Staff             |
| Court closure / reopening     | Admin, Staff             |
| Court status change           | Staff, System            |
| Admin config change           | Admin                    |
| Staff account creation/edit   | Admin                    |

Audit entries must include: actor (userId + role), action, affected entity ID, timestamp, and
any relevant before/after values.

---

## 10. Notification Trigger Summary

| Notification              | Trigger                                    | Recipient |
|---------------------------|--------------------------------------------|-----------|
| Booking Confirmation      | Booking → CONFIRMED                        | Player    |
| Booking Reminder          | X minutes before slot (configurable)       | Player    |
| Cancellation Notice       | Booking → CANCELLED                        | Player    |
| No-show Notice            | Booking → NO_SHOW                          | Player    |
| Rain Policy Announcement  | Admin triggers broadcast                   | All users |
| Waitlist Opportunity      | Slot opens after cancellation/no-show      | Next waitlisted player |
