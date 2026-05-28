# PickleFlow – Full API Reference

**Base URL**: `/api/v1`  
**Auth**: JWT Bearer Token (except Public endpoints)  
**Format**: JSON  
**Version**: 1.0 (May 28, 2026)

---

## 1. Authentication Module

| Endpoint                  | Method | Description                        | Access    |
|---------------------------|--------|------------------------------------|-----------|
| `/auth/register`          | POST   | Register new player account        | Public    |
| `/auth/login`             | POST   | Login (Player / Staff / Admin)     | Public    |
| `/auth/me`                | GET    | Get current user profile           | Auth      |
| `/auth/refresh`           | POST   | Refresh access token               | Auth      |
| `/auth/logout`            | POST   | Logout and invalidate token        | Auth      |
| `/auth/forgot-password`   | POST   | Request password reset email       | Public    |
| `/auth/reset-password`    | POST   | Reset password via token           | Public    |

### POST `/auth/register`
```json
Request:  { "name": "string", "mobile": "string", "email": "string", "password": "string" }
Response: { "success": true, "token": "jwt", "user": { "id", "name", "role": "PLAYER" } }
```

### POST `/auth/login`
```json
Request:  { "email": "string", "password": "string" }
Response: { "success": true, "token": "jwt", "refreshToken": "jwt", "user": { "id", "role" } }
```

---

## 2. Court Management Module

| Endpoint                     | Method | Description                           | Access        |
|------------------------------|--------|---------------------------------------|---------------|
| `/courts`                    | GET    | List all courts with availability     | Public        |
| `/courts`                    | POST   | Create a new court                    | Admin         |
| `/courts/{courtId}`          | GET    | Get court detail                      | Public        |
| `/courts/{courtId}`          | PUT    | Update court info                     | Admin         |
| `/courts/{courtId}`          | DELETE | Deactivate court                      | Admin         |
| `/courts/{courtId}/close`    | POST   | Close court (maintenance/reason)      | Admin / Staff |
| `/courts/{courtId}/reopen`   | POST   | Reopen a closed court                 | Admin / Staff |

### POST `/courts/{courtId}/close`
```json
Request:  { "reason": "string", "scheduledReopenAt": "ISO8601 datetime (optional)" }
Response: { "success": true, "warnings": ["Booking #123 affected"] }
```
> ⚠️ Must return `warnings` array if active/future bookings exist on this court.

---

## 3. Booking Engine Module

| Endpoint                         | Method | Description                               | Access        |
|----------------------------------|--------|-------------------------------------------|---------------|
| `/bookings/available-slots`      | GET    | Get available slots for a date/court      | Public        |
| `/bookings`                      | POST   | Create new online booking                 | Player        |
| `/bookings`                      | GET    | List bookings (own for Player; all for Staff/Admin) | Auth  |
| `/bookings/{bookingId}`          | GET    | Get booking detail                        | Auth          |
| `/bookings/{bookingId}/cancel`   | POST   | Cancel a booking                          | Player/Staff  |
| `/bookings/recurring`            | POST   | Create recurring bookings                 | Player        |
| `/bookings/waitlist`             | POST   | Join waitlist for a full slot             | Player        |
| `/bookings/{bookingId}/status`   | PATCH  | Update booking status                     | Staff         |

### GET `/bookings/available-slots`
```
Query params: date=YYYY-MM-DD&courtId=uuid (courtId optional)
Response: { "slots": [{ "courtId", "courtName", "startTime", "endTime", "available": true }] }
```

### POST `/bookings`
```json
Request:  { "courtId": "uuid", "date": "YYYY-MM-DD", "startTime": "HH:MM" }
Response: { "success": true, "bookingId": "uuid", "status": "PENDING_PAYMENT", "paymentUrl": "..." }
```

### POST `/bookings/recurring`
```json
Request:  { "courtId": "uuid", "startTime": "HH:MM", "dayOfWeek": 1, "startDate": "YYYY-MM-DD", "occurrences": 8 }
Response: { "success": true, "bookings": [{ "bookingId", "date", "status" }] }
```

---

## 4. Payment Module

| Endpoint                       | Method | Description                           | Access        |
|--------------------------------|--------|---------------------------------------|---------------|
| `/payments/initiate`           | POST   | Initiate GCash payment                | Player        |
| `/payments/verify`             | POST   | GCash webhook — payment confirmation  | System        |
| `/payments/walkin`             | POST   | Record walk-in payment                | Staff         |
| `/payments/{paymentId}`        | GET    | Get payment details                   | Auth          |
| `/payments/{bookingId}/refund` | POST   | Process manual refund                 | Admin / Staff |

### POST `/payments/initiate`
```json
Request:  { "bookingId": "uuid" }
Response: { "success": true, "paymentId": "uuid", "gcashUrl": "...", "expiresAt": "ISO8601" }
```

### POST `/payments/walkin`
```json
Request:  { "bookingId": "uuid", "method": "CASH|GCASH", "gcashReference": "string (if GCASH)", "amount": 0 }
Response: { "success": true, "paymentId": "uuid" }
```

### POST `/payments/{bookingId}/refund`
```json
Request:  { "reason": "string", "amount": 0 }
Response: { "success": true, "refundId": "uuid", "status": "PROCESSED" }
```
> Refunds are **always manual** — never auto-triggered by the system.

---

## 5. Check-in & Operations Module

| Endpoint                          | Method | Description                              | Access  |
|-----------------------------------|--------|------------------------------------------|---------|
| `/checkin/qr/{bookingId}`         | GET    | Generate QR code for booking             | Player  |
| `/checkin/verify`                 | POST   | Verify check-in (QR or reference)        | Staff   |
| `/checkin/self`                   | POST   | Player self check-in via QR              | Player  |
| `/operations/no-show/process`     | POST   | Automated no-show sweep (scheduler only) | System  |
| `/operations/court/release`       | POST   | Manually release an occupied court       | Staff   |
| `/operations/status`              | GET    | Real-time court status for dashboard     | Staff   |

### POST `/checkin/self`
```json
Request:  { "qrToken": "string" }
Response: { "success": true, "bookingId": "uuid", "courtNumber": "string", "checkedInAt": "ISO8601" }
```

### POST `/operations/court/release`
```json
Request:  { "courtId": "uuid", "reason": "string" }
Response: { "success": true, "releasedAt": "ISO8601" }
```

### GET `/operations/status`
```json
Response: {
  "courts": [{
    "courtId": "uuid",
    "courtNumber": "string",
    "status": "AVAILABLE|OCCUPIED|OVERDUE|CLOSED",
    "currentBooking": { "bookingId", "playerName", "endTime" } | null
  }]
}
```

---

## 6. Real-time Module (WebSocket)

**Endpoint**: `ws://[server]/realtime`  
**Auth**: Include JWT in connection header or query param.

### Events (server → client)

| Event                  | Payload                                                |
|------------------------|--------------------------------------------------------|
| `court_status_updated` | `{ courtId, status, updatedAt }`                      |
| `booking_confirmed`    | `{ bookingId, courtId, playerName, startTime }`       |
| `checkin_completed`    | `{ bookingId, courtId, checkedInAt }`                 |
| `court_released`       | `{ courtId, releasedAt, reason }`                     |
| `no_show_detected`     | `{ bookingId, courtId, detectedAt }`                  |

> All status-changing operations (check-in, release, no-show, booking confirmation) **must** emit
> the relevant WebSocket event immediately after the DB write.

---

## 7. Staff Dashboard Module

| Endpoint                  | Method | Description                             | Access  |
|---------------------------|--------|-----------------------------------------|---------|
| `/dashboard/summary`      | GET    | Real-time summary of all courts         | Staff   |
| `/dashboard/walkin`       | POST   | Quick walk-in booking creation          | Staff   |
| `/staff/roles`            | GET    | List staff roles and permissions        | Admin   |
| `/staff/permissions`      | PUT    | Update staff role permissions           | Admin   |

### GET `/dashboard/summary`
```json
Response: {
  "totalCourts": 0,
  "available": 0, "occupied": 0, "overdue": 0, "closed": 0,
  "courts": [{ "courtId", "status", "colorCode": "GREEN|YELLOW|RED|GRAY" }]
}
```

---

## 8. Player Portal Module

| Endpoint                       | Method | Description                              | Access  |
|--------------------------------|--------|------------------------------------------|---------|
| `/player/bookings/upcoming`    | GET    | Upcoming bookings with QR codes          | Player  |
| `/player/bookings/history`     | GET    | Past booking history                     | Player  |
| `/player/announcements`        | GET    | Active facility announcements            | Player  |
| `/player/profile`              | PUT    | Update player profile                    | Player  |

---

## 9. Admin & Configuration Module

| Endpoint               | Method | Description                                 | Access  |
|------------------------|--------|---------------------------------------------|---------|
| `/admin/config`        | GET    | Get current system configuration            | Admin   |
| `/admin/config`        | PUT    | Update system settings                      | Admin   |
| `/admin/audit-logs`    | GET    | View audit log entries                      | Admin   |
| `/admin/reports/revenue` | GET  | Revenue and utilization report              | Admin   |

### GET/PUT `/admin/config`
```json
{
  "gracePeriodMinutes": 15,
  "paymentTimeoutMinutes": 10,
  "maxAdvanceBookingDays": 14,
  "minBookingNoticeMinutes": 30,
  "rainPolicy": "OPTION_A|OPTION_B|OPTION_C",
  "operatingHours": { "open": "06:00", "close": "02:00" },
  "reminderLeadTimeMinutes": 60
}
```

---

## Error Codes Reference

| Error Code                | Meaning                                              |
|---------------------------|------------------------------------------------------|
| `BOOKING_CONFLICT`        | Slot already booked                                  |
| `BOOKING_NOT_FOUND`       | Booking ID does not exist                            |
| `PAYMENT_TIMEOUT`         | Payment window expired, slot released                |
| `INSUFFICIENT_NOTICE`     | Booking within minimum notice period                 |
| `COURT_CLOSED`            | Court is closed or under maintenance                 |
| `UNAUTHORIZED`            | Invalid or missing JWT                               |
| `FORBIDDEN`               | Role lacks permission for this action                |
| `CHECKIN_ALREADY_DONE`    | Booking already checked in                           |
| `NO_SHOW_PROCESSED`       | Booking already marked as no-show                    |
| `REFUND_NOT_ELIGIBLE`     | Booking not eligible for refund under current policy |
