---
name: pickleflow
description: >
  Use this skill whenever the user is working on PickleFlow — the pickleball court booking and
  management system. Triggers include: designing or coding any PickleFlow module (authentication,
  bookings, payments, check-in, staff dashboard, player portal, admin config, analytics,
  real-time/WebSocket), writing or reviewing API endpoints, describing system behavior, modeling
  database schemas, debugging PickleFlow workflows, writing test cases, explaining business rules,
  or generating documentation. Also use when the user mentions booking lifecycle, court status
  management, GCash payment flows, QR check-in, no-show handling, waitlists, or anything related
  to running or building a pickleball facility platform. Even if the user's request seems general
  (e.g., "add a refund endpoint"), consult this skill first — PickleFlow has specific rules that
  affect implementation.
---

# PickleFlow – AI Skill Reference

PickleFlow is a **modular monolith** web application for pickleball court booking and management,
designed for facilities in the Philippines (primary market: Soccsksargen region). It serves
**Players**, **Staff**, and **Admins** through two distinct frontends: a Player Portal and a
Staff Dashboard.

> **For deep dives, load the relevant reference file:**
> - `references/api.md` — Full endpoint table with methods, access levels, and payloads
> - `references/business-rules.md` — All configurable rules, constraints, and edge cases
> - `references/modules.md` — Module-by-module responsibility and interaction map

---

## 1. System at a Glance

| Dimension        | Detail                                                        |
|------------------|---------------------------------------------------------------|
| Architecture     | Modular Monolith (layered: Presentation → Business → Data → Integration) |
| API Style        | RESTful (`/api/v1`) + WebSocket (`ws://[server]/realtime`)    |
| Auth             | JWT Bearer Tokens; Role-Based Access Control (RBAC)          |
| Payments         | GCash (online, upfront); Cash or GCash reference (walk-in)    |
| Booking Unit     | Fixed 1-hour slots                                            |
| User Roles       | Player · Staff · Admin                                        |
| Key Integrations | GCash payment gateway, QR code generation, Email notifications |
| Uptime Target    | 99% during operating hours (6:00 AM – 2:00 AM)               |
| Concurrency      | ≥ 200 concurrent users; dashboard refresh ≤ 3 seconds         |

---

## 2. The 11 Modules (Quick Reference)

| # | Module                    | Core Job                                                         |
|---|---------------------------|------------------------------------------------------------------|
| 1 | **Authentication**        | Register/login/session for 3 roles; RBAC enforcement            |
| 2 | **Court Management**      | CRUD courts; status control (Available/Occupied/Closed/Maintenance) |
| 3 | **Booking Engine** ⭐     | Central orchestrator; validates rules; manages full booking lifecycle |
| 4 | **Payment**               | GCash online flow; walk-in recording; refunds; payment history   |
| 5 | **Check-in & Operations** | QR/manual check-in; no-show detection; court release             |
| 6 | **Real-time**             | WebSocket broadcasts for all status/booking changes              |
| 7 | **Staff Dashboard**       | Color-coded court view; walk-in creation; manual ops             |
| 8 | **Player Portal**         | Self-service: bookings, QR codes, history, announcements         |
| 9 | **Notification**          | Automated emails: confirmation, reminders, cancellations, rain   |
| 10| **Admin & Config**        | Business rule settings; staff management; audit logs             |
| 11| **Analytics**             | Revenue, utilization, volume, no-show reports                    |

> ⭐ The Booking Engine is the most complex module. Always check `references/business-rules.md`
> before implementing booking logic.

---

## 3. Booking Lifecycle (State Machine)

```
[Player creates booking]
        │
        ▼
  PENDING PAYMENT  ──(timeout)──► [slot released, booking cancelled]
        │
   (GCash paid)
        │
        ▼
   CONFIRMED  ◄──────────────────────────────────────────────────────────────┐
        │                                                                     │
   (grace period)                                                             │
        ├──(no check-in)──► NO-SHOW ──► [court released]                    │
        │                                                                     │
   (check-in)                                                                 │
        │                                                                     │
        ▼                                                                     │
   CHECKED-IN                                                                 │
        │                                                                     │
   (session ends)                                                             │
        ├──(normal end)──► COMPLETED                                          │
        └──(overdue, next slot booked)──► RELEASED (strict end enforced)     │
                                                                              │
        [Player or Staff cancels] ──────────────────────────────────────────►
```

**Status values:** `PENDING_PAYMENT` · `CONFIRMED` · `CHECKED_IN` · `COMPLETED` · `CANCELLED` · `NO_SHOW`

---

## 4. Critical Business Rules (Summary)

These rules are **non-negotiable** and must be enforced by the Booking Engine. Full details in
`references/business-rules.md`.

1. **Minimum notice**: Online bookings require ≥ 30 minutes before slot start.
2. **Fixed duration**: All slots are exactly 1 hour — no partial hours.
3. **No double-booking**: A court cannot have two confirmed bookings in the same slot.
4. **Payment upfront**: Online bookings are not confirmed until GCash payment is received.
5. **Payment timeout**: Slot is reserved temporarily during payment; released on timeout (configurable).
6. **Strict end time**: If the next slot is already booked, current session has **zero** grace period.
7. **Grace period**: Applied only when the next slot is free (duration configurable by Admin).
8. **No-show detection**: Automated after configurable grace period; triggers court release.
9. **Admin warning on closure**: Warn Admin if future/active bookings exist when closing a court.
10. **Refunds**: Manual only, initiated by authorized Staff or Admin.

---

## 5. API Conventions

- **Base URL**: `/api/v1`
- **Format**: JSON
- **Auth header**: `Authorization: Bearer <jwt_token>`
- **Error shape**:
  ```json
  { "success": false, "message": "Human-readable error", "errorCode": "SNAKE_CASE_CODE" }
  ```
- **WebSocket**: `ws://[server]/realtime` — emits `court_status_updated`, `booking_confirmed`,
  `checkin_completed`, `court_released`, `no_show_detected`

> Load `references/api.md` for the complete endpoint table (all modules, methods, access levels).

---

## 6. Access Control Matrix

| Action                        | Player | Staff | Admin |
|-------------------------------|:------:|:-----:|:-----:|
| Browse available slots        | ✅     | ✅    | ✅    |
| Create online booking         | ✅     | —     | —     |
| Create walk-in booking        | —      | ✅    | ✅    |
| Cancel booking                | ✅*    | ✅    | ✅    |
| Process check-in              | ✅†    | ✅    | ✅    |
| Record walk-in payment        | —      | ✅    | ✅    |
| Process refund                | —      | ✅    | ✅    |
| Release court                 | —      | ✅    | ✅    |
| Close/reopen court            | —      | ✅    | ✅    |
| Create/edit/deactivate court  | —      | —     | ✅    |
| Configure system rules        | —      | —     | ✅    |
| Manage staff accounts         | —      | —     | ✅    |
| View audit logs               | —      | —     | ✅    |

\* Subject to cancellation policy (configurable)
† Via QR self-check-in only

---

## 7. Real-time & Dashboard Color Codes

Staff Dashboard uses color coding for instant court status recognition:

| Color  | Status      | Meaning                              |
|--------|-------------|--------------------------------------|
| 🟢 Green  | Available   | Court is free and bookable           |
| 🟡 Yellow | Occupied    | Active checked-in session            |
| 🔴 Red    | Overdue     | Session past end time, needs release |
| ⚫ Gray   | Closed      | Court closed or under maintenance    |

---

## 8. Key Data Flows

**Online Booking:**
```
Player Portal → Auth check → Booking Engine (validate rules)
  → Payment (initiate GCash) → [webhook: payment verified]
  → Booking Engine (CONFIRMED) → Real-time (broadcast)
  → Notification (confirmation email)
```

**Walk-in Booking:**
```
Staff Dashboard → Auth check → Booking Engine (validate rules)
  → Payment (record cash/GCash ref) → CONFIRMED immediately
  → Real-time (broadcast)
```

**Check-in:**
```
QR scan / staff manual → Check-in Module (verify booking)
  → Booking Engine (status: CHECKED_IN)
  → Real-time (broadcast: checkin_completed)
```

**No-show:**
```
Scheduler (cron) → /operations/no-show/process
  → Check-in Module (flag as NO_SHOW)
  → Booking Engine (update status) → Real-time (broadcast)
  → Court released for waitlist / next booking
```

---

## 9. Notifications (Automated Email Triggers)

| Event                    | Recipient | Trigger                          |
|--------------------------|-----------|----------------------------------|
| Booking confirmation     | Player    | Payment verified / walk-in created |
| Booking reminder         | Player    | Configurable time before start   |
| Cancellation notice      | Player    | Booking cancelled (any reason)   |
| Rain policy announcement | All       | Admin-initiated broadcast        |

> Future channels (WhatsApp, SMS) are planned but not in scope for current MVP.

---

## 10. Configurables (Admin Panel)

All of these are runtime-configurable via `/admin/config` — **never hard-code them**:

- Grace period duration (no-show detection window)
- Payment timeout window (slot reservation during payment)
- Maximum advance booking days
- Minimum booking notice (default: 30 min)
- Rain policy (3 predefined options)
- Facility operating hours
- Booking reminder lead time

---

## 11. Non-Functional Targets

| Attribute      | Target                                                     |
|----------------|------------------------------------------------------------|
| Performance    | 200+ concurrent users; dashboard refresh ≤ 3s              |
| Availability   | 99% uptime, 6:00 AM – 2:00 AM                              |
| Security       | Encrypted sensitive data; RBAC; full audit trail           |
| Mobile         | Player Portal must be mobile-friendly                      |
| Tablet         | Staff Dashboard optimized for tablets (large touch targets)|
| Data Integrity | Strict double-booking prevention                           |
| Auditability   | All booking, payment, and court changes logged             |

---

## 12. Common Implementation Pitfalls

- ❌ **Don't hard-code grace periods or timeouts** — always read from config.
- ❌ **Don't skip the "active bookings" warning** when closing a court via Admin.
- ❌ **Don't allow same-slot overlap** — validate atomically to prevent race conditions.
- ❌ **Don't auto-refund** — refunds are always manual and staff-initiated.
- ❌ **Don't apply grace period** when the next slot is already confirmed.
- ✅ **Do broadcast via WebSocket** after every court status change.
- ✅ **Do log every financial and booking modification** to the audit trail.
- ✅ **Do validate JWT and RBAC** on every protected endpoint.
