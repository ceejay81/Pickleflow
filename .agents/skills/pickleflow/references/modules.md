# PickleFlow – Module Responsibilities & Interaction Map

Load this file when designing module boundaries, planning service-to-service calls, or
debugging unexpected behavior across modules.

---

## Module Responsibility Summary

### 1. Authentication Module
**Owns**: User accounts, sessions, roles, permissions  
**Exposes**: `validateToken(jwt)`, `getUser(userId)`, `checkPermission(userId, action)`  
**Called by**: Every other module (all protected endpoints)  
**Does NOT**: Handle booking logic, payment, or operational state

### 2. Court Management Module
**Owns**: Court records, court status (Available/Occupied/Closed/Maintenance)  
**Exposes**: `getCourt(courtId)`, `updateStatus(courtId, status)`, `listAvailable(date)`  
**Called by**: Booking Engine (availability check), Check-in & Ops (status update), Staff Dashboard  
**Does NOT**: Validate booking rules or process payments

### 3. Booking Engine Module ⭐ (Central Orchestrator)
**Owns**: Booking records, booking lifecycle state machine, waitlist records  
**Exposes**: `createBooking(...)`, `cancelBooking(...)`, `updateStatus(bookingId, status)`, `getAvailableSlots(...)`  
**Calls**: Court Management (availability), Payment (trigger payment), Real-time (broadcast), Notification (trigger emails)  
**Business rules enforced here**: min notice, max advance days, no-overlap, recurring logic, waitlist management  
**Does NOT**: Process payments directly; does not update court status directly (delegates to Check-in & Ops)

### 4. Payment Module
**Owns**: Payment records, refund records, payment status  
**Exposes**: `initiatePayment(bookingId)`, `verifyPayment(webhookPayload)`, `recordWalkin(...)`, `processRefund(...)`  
**Called by**: Booking Engine (on booking creation), Staff Dashboard (walk-in flow)  
**Calls**: Booking Engine (notify of payment success → trigger status → CONFIRMED)  
**Does NOT**: Create bookings; does not auto-refund; does not manage court state

### 5. Check-in & Operations Module
**Owns**: Check-in events, court operational state transitions  
**Exposes**: `generateQR(bookingId)`, `verifyCheckin(qrToken|reference)`, `selfCheckin(qrToken)`, `releaseСourt(courtId)`, `processNoShows()`  
**Calls**: Booking Engine (update status to CHECKED_IN / NO_SHOW / COMPLETED), Court Management (update status), Real-time (broadcast events)  
**Business logic here**: Grace period check, strict end-time enforcement, no-show detection  
**Does NOT**: Create bookings; does not process payments

### 6. Real-time Module
**Owns**: WebSocket connection management, event broadcasting  
**Exposes**: `broadcast(event, payload)`  
**Called by**: Booking Engine, Check-in & Ops, Court Management (any state change)  
**Does NOT**: Persist state; does not make business decisions; pure event relay

### 7. Staff Dashboard Module
**Owns**: Dashboard aggregation view, walk-in booking shortcuts  
**Exposes**: Dashboard summary API, walk-in booking endpoint  
**Calls**: Court Management (status), Booking Engine (walk-in booking), Check-in & Ops (manual check-in, release)  
**Does NOT**: Own any domain state; purely a composition layer for the staff UI

### 8. Player Portal Module
**Owns**: Player-facing views (upcoming bookings, history, announcements)  
**Exposes**: `/player/*` endpoints  
**Calls**: Booking Engine (fetch bookings), Notification (fetch announcements)  
**Does NOT**: Duplicate booking/payment logic; surfaces existing data only

### 9. Notification Module
**Owns**: Notification templates, notification dispatch queue  
**Exposes**: `sendNotification(type, recipientId, data)`  
**Called by**: Booking Engine (confirmation, reminder, cancellation), Admin & Config (rain announcement), Check-in & Ops (no-show, waitlist)  
**Does NOT**: Make booking decisions; purely a messaging service  
**Current channels**: Email (primary); WhatsApp/SMS planned for future

### 10. Admin & Configuration Module
**Owns**: System configuration values, staff account management, audit log  
**Exposes**: `getConfig(key)`, `updateConfig(key, value)`, `logAudit(actor, action, entity, diff)`  
**Called by**: All modules for config reads; all modules for audit writes  
**Does NOT**: Enforce rules directly — it stores them; enforcement is in Booking Engine / Check-in

### 11. Analytics Module
**Owns**: Aggregated reports (revenue, utilization, volume, no-shows)  
**Exposes**: Report generation endpoints  
**Called by**: Admin UI  
**Reads from**: Booking records, Payment records, Check-in events (read-only, no side effects)  
**Does NOT**: Modify any operational state

---

## Module Interaction Diagram

```
                          ┌─────────────────────┐
                          │  Admin & Config      │◄── All modules (config reads + audit writes)
                          └─────────────────────┘
                                    ▲
                    ┌───────────────┼───────────────┐
                    │               │               │
          ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
          │   Player    │  │    Staff     │  │   Analytics  │
          │   Portal    │  │  Dashboard   │  │   Module     │
          └──────┬──────┘  └──────┬───────┘  └──────────────┘
                 │                │
                 ▼                ▼
         ┌───────────────────────────────┐
         │        Booking Engine ⭐       │
         │  (Central Business Orchestrator)│
         └────┬────────┬────────┬────────┘
              │        │        │
              ▼        ▼        ▼
     ┌──────────┐ ┌─────────┐ ┌────────────────────┐
     │  Court   │ │ Payment │ │  Check-in &        │
     │ Mgmt     │ │ Module  │ │  Operations        │
     └──────────┘ └─────────┘ └────────┬───────────┘
                                        │
                                        ▼
                              ┌─────────────────┐
                              │  Real-time      │◄── Court Mgmt, Booking Engine, Check-in
                              │  Module (WS)    │──► All connected clients (Staff + Players)
                              └─────────────────┘
                                        │
                              ┌─────────────────┐
                              │  Notification   │◄── Booking Engine, Check-in, Admin
                              │  Module         │──► Players (email)
                              └─────────────────┘
```

---

## Critical Module Rules

1. **Authentication is called first** — every non-public endpoint validates JWT before any business logic.
2. **Booking Engine does not directly update court status** — it signals Check-in & Ops, which owns operational transitions.
3. **Real-time module is always the last step** — broadcast after DB writes are committed, never before.
4. **Config values are never hard-coded** — always fetched from Admin & Configuration module at runtime.
5. **Audit writes are fire-and-forget** — a failed audit write should log an error but NOT roll back the primary transaction.
6. **Analytics is read-only** — it never writes to operational tables.
