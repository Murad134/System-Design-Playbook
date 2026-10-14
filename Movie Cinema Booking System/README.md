# Cinema Hall Seat Booking System

A concurrency-safe movie ticket booking platform designed around **atomic seat reservations, reliable payment processing, and consistent booking state transitions**.

This README describes a practical implementation using React, Node.js/Express, and PostgreSQL. The same design principles can be applied with .NET, Java, or other backend stacks.

## Table of Contents

- [1. Goals](#1-goals)
- [2. Core Design Principles](#2-core-design-principles)
- [3. High-Level Architecture](#3-high-level-architecture)
- [4. Technology Stack](#4-technology-stack)
- [5. Booking Lifecycle](#5-booking-lifecycle)
- [6. Database Design](#6-database-design)
- [7. ER Diagram](#7-er-diagram)
- [8. Preventing Double Booking](#8-preventing-double-booking)
- [9. Atomic Multi-Seat Reservation](#9-atomic-multi-seat-reservation)
- [10. Reservation Expiry](#10-reservation-expiry)
- [11. Payment Consistency](#11-payment-consistency)
- [12. REST API](#12-rest-api)
- [13. Idempotency](#13-idempotency)
- [14. ACID Transactions](#14-acid-transactions)
- [15. Edge Cases](#15-edge-cases)
- [16. Security](#16-security)
- [17. Reliability and Scalability](#17-reliability-and-scalability)
- [18. Suggested Project Structure](#18-suggested-project-structure)
- [19. Local Setup](#19-local-setup)
- [20. Testing Checklist](#20-testing-checklist)
- [21. Future Improvements](#21-future-improvements)

---

## 1. Goals

Customers can:

- Register, log in, and manage their profiles.
- Browse movies, cinemas, halls, and showtimes.
- View seat availability for a specific show.
- Reserve multiple seats temporarily.
- Pay through a supported payment gateway.
- Receive a unique booking reference and ticket after confirmation.
- View booking history and request eligible cancellations/refunds.

Administrators can manage movies, cinemas, halls, seats, show schedules, and show cancellations.

### Critical invariants

1. A seat can be confirmed for at most one booking for a given show.
2. A multi-seat reservation succeeds completely or has no effect.
3. An expired reservation token cannot confirm a booking.
4. Duplicate API requests and payment webhooks must not create duplicate bookings or payment operations.
5. A payment timeout is not automatically a payment failure.
6. Payment success must be verified before confirming a booking.
7. Booking, payment, and reservation states must follow valid transitions.

## 2. Core Design Principles

- **PostgreSQL is the source of truth** for seat ownership and booking state.
- Use transactions and row locks to serialize conflicting seat operations.
- Enforce invariants with database constraints, not frontend checks alone.
- Keep database transactions short; never hold seat locks while waiting for a payment gateway.
- Use idempotency keys for reservation and payment requests.
- Verify webhook signatures and payment details with the provider.
- Use background workers for expiry, reconciliation, refunds, and notifications.
- Use an outbox pattern for reliable event publication.

## 3. High-Level Architecture

```mermaid
flowchart TB
    Client[React Web App] --> Gateway[Load Balancer / API Gateway]
    Gateway --> Auth[Auth and Admin API]
    Gateway --> Catalog[Catalog API]
    Gateway --> Booking[Booking API]
    Gateway --> Payment[Payment API]

    Auth --> DB[(PostgreSQL)]
    Catalog --> DB
    Booking --> DB
    Payment --> DB

    Payment <--> PG[External Payment Gateway]
    PG -->|Signed Webhook| Payment

    Worker[Background Workers] --> DB
    Worker --> Queue[Message Queue]
    Queue --> Notify[Email / SMS Notifications]
    Worker --> PG
```

### Components

| Component | Responsibilities |
|---|---|
| React frontend | Movie discovery, seat selection, checkout, booking history |
| Auth/Admin API | Authentication, roles, administration |
| Catalog API | Movies, cinemas, halls, seats, showtimes |
| Booking API | Seat holds, booking lifecycle, ticket records |
| Payment API | Payment initiation, verification, webhooks, refunds |
| PostgreSQL | Authoritative booking, seat, reservation, and payment state |
| Expiry worker | Release expired seat holds safely |
| Reconciliation worker | Resolve unknown payment outcomes |
| Outbox/notification workers | Publish events and send ticket notifications |

**Deployment recommendation:** Begin with a modular monolith and one PostgreSQL database. Split services only when scale, team ownership, or operational requirements justify the additional distributed-systems complexity.

## 4. Technology Stack

Example stack:

- **Frontend:** React, TypeScript, Tailwind CSS
- **Backend:** Node.js, Express, TypeScript
- **Database:** PostgreSQL
- **Authentication:** Secure session or JWT-based authentication
- **Payments:** A supported provider such as SSLCommerz or Stripe, depending on region and business requirements
- **Background jobs:** A queue/worker system or scheduled workers
- **Caching:** Redis for non-authoritative catalog data
- **Deployment:** Docker, reverse proxy, managed PostgreSQL, centralized logs and metrics

The payment provider is an integration choice, not a fixed dependency of the design.

## 5. Booking Lifecycle

### Seat lifecycle

```mermaid
stateDiagram-v2
    [*] --> AVAILABLE
    AVAILABLE --> HELD: Reservation transaction commits
    HELD --> BOOKED: Payment verified and booking finalized
    HELD --> AVAILABLE: Expired or released hold
    BOOKED --> AVAILABLE: Valid cancellation policy permits release
```

### Booking lifecycle

Typical booking states:

- `PENDING_PAYMENT`
- `CONFIRMED`
- `EXPIRED`
- `CANCELLED`
- `REFUND_PENDING`
- `REFUNDED`

`REFUND_PENDING` and `REFUNDED` are often best modeled as financial/cancellation states alongside the booking's fulfillment state, rather than as replacements for every booking state. Choose one consistent domain model and enforce its transitions.

### Payment lifecycle

Typical payment states:

- `PENDING`
- `PROCESSING`
- `UNKNOWN`
- `SUCCEEDED`
- `FAILED`
- `REFUND_PENDING`
- `REFUNDED`

A network timeout produces an **unknown outcome**, not proof of failure. Query the provider or reconcile later before deciding whether to retry, confirm, or refund.

## 6. Database Design

The following tables are the main domain entities. Use UUID primary keys where appropriate, `TIMESTAMPTZ` for timestamps, and `NUMERIC(12,2)` (or integer minor units) for money. Store currency explicitly.

### `Users`

| Column | Notes |
|---|---|
| `user_id` | PK, UUID |
| `name` | Customer name |
| `email` | UNIQUE, NOT NULL |
| `password_hash` | Nullable if external identity provider is used |
| `role` | Customer/admin role |
| `created_at` | Creation timestamp |

### `Movies`

| Column | Notes |
|---|---|
| `movie_id` | PK |
| `title` | Movie title |
| `description` | Synopsis |
| `duration_minutes` | Positive integer |
| `language` | Language |
| `status` | Draft/active/inactive |

### `Cinemas`

| Column | Notes |
|---|---|
| `cinema_id` | PK |
| `name` | Cinema name |
| `address` | Location |

### `Halls`

| Column | Notes |
|---|---|
| `hall_id` | PK |
| `cinema_id` | FK → `Cinemas.cinema_id` |
| `name` | Hall name |
| `(cinema_id, name)` | UNIQUE |

### `Seats`

| Column | Notes |
|---|---|
| `seat_id` | PK |
| `hall_id` | FK → `Halls.hall_id` |
| `row_label` | Row identifier |
| `seat_number` | Seat number |
| `seat_type` | Standard/premium/VIP |
| `(hall_id, row_label, seat_number)` | UNIQUE |

### `Shows`

| Column | Notes |
|---|---|
| `show_id` | PK |
| `movie_id` | FK → `Movies.movie_id` |
| `hall_id` | FK → `Halls.hall_id` |
| `start_time`, `end_time` | `TIMESTAMPTZ` |
| `status` | Scheduled/cancelled/completed |

Prevent overlapping shows in the same hall with an exclusion constraint using a PostgreSQL range type where appropriate, or with a transaction/locking strategy.

### `ShowSeats` (operational seat state)

One row per seat per show. This table is the authoritative current state used to coordinate reservations.

| Column | Notes |
|---|---|
| `show_id` | PK part, FK → `Shows.show_id` |
| `seat_id` | PK part, FK → `Seats.seat_id` |
| `status` | `AVAILABLE`, `HELD`, or `BOOKED` |
| `reservation_id` | Current hold ID, nullable |
| `hold_expires_at` | Current hold expiry, nullable |
| `booking_id` | Confirmed booking ID, nullable |
| `unit_price` | Price snapshot for this show |

Primary key: `(show_id, seat_id)`.

Ensure each `seat_id` belongs to the hall used by the show. This can be enforced with suitable composite foreign keys and a schema that exposes the hall relationship, or by carefully validating the invariant when show-seat rows are created.

### `Bookings`

| Column | Notes |
|---|---|
| `booking_id` | PK |
| `booking_reference` | UNIQUE, customer-facing reference |
| `user_id` | FK → `Users.user_id` |
| `show_id` | FK → `Shows.show_id` |
| `status` | Booking lifecycle state |
| `total_amount` | Immutable amount snapshot |
| `currency` | Currency code |
| `idempotency_key` | Client request key |
| `expires_at` | Hold/payment deadline |
| `created_at` | Creation timestamp |

Add `UNIQUE (user_id, idempotency_key)` and make the key non-null for idempotent create requests. Store a request hash and response result in a dedicated idempotency table if you need to preserve replay results robustly.

### `BookingSeats`

Contains confirmed seats and the price paid for each seat.

| Column | Notes |
|---|---|
| `booking_id` | PK part, FK → `Bookings.booking_id` |
| `show_id` | Show ID |
| `seat_id` | Seat ID |
| `unit_price` | Price snapshot |

Constraints:

- `PRIMARY KEY (booking_id, seat_id)`
- `UNIQUE (show_id, seat_id)` — prevents the same show-seat from being confirmed twice.
- Use a composite foreign key to `ShowSeats(show_id, seat_id)`.

### `SeatReservations`

Stores current and historical reservation attempts.

| Column | Notes |
|---|---|
| `reservation_id` | PK |
| `booking_id` | FK → `Bookings.booking_id` |
| `show_id`, `seat_id` | Reserved show-seat |
| `reservation_token` | Unpredictable token or token hash |
| `status` | Active/released/expired/converted |
| `expires_at` | Hold deadline |
| `created_at` | Creation timestamp |

Use one row per reserved seat, or a separate reservation-header table plus reservation-seat rows. The latter is often cleaner for a multi-seat hold. Keep historical records rather than deleting them.

### `Payments`

| Column | Notes |
|---|---|
| `payment_id` | PK |
| `booking_id` | FK → `Bookings.booking_id` |
| `provider` | Gateway name |
| `provider_payment_id` | Provider transaction identifier |
| `idempotency_key` | Payment request key |
| `amount`, `currency` | Expected amount and currency |
| `status` | Payment lifecycle state |
| `created_at`, `updated_at` | Timestamps |

Add appropriate unique constraints, including a unique provider transaction identifier where the provider guarantees its uniqueness. Consider a separate `PaymentEvents` table for immutable provider event records.

### `BookingHistory`

An append-only audit trail for booking state changes.

| Column | Notes |
|---|---|
| `history_id` | PK |
| `booking_id` | FK → `Bookings.booking_id` |
| `old_status`, `new_status` | State transition |
| `event_type` | Reason/action |
| `actor_id` | Nullable FK → `Users.user_id` |
| `metadata` | JSONB, avoid secrets |
| `created_at` | Event timestamp |

### Additional recommended tables

- `IdempotencyKeys`: scope, key, request hash, status, saved response, expiry.
- `ProcessedWebhookEvents`: unique provider event ID, received timestamp, processing result.
- `OutboxEvents`: event ID, aggregate ID, event type, payload, published timestamp.
- `Refunds`: refund ID, payment ID, provider refund ID, amount, status.
- `Tickets`: ticket ID, booking ID, seat, signed/unguessable ticket code, status.

## 7. ER Diagram

```mermaid
erDiagram
    USERS ||--o{ BOOKINGS : places
    MOVIES ||--o{ SHOWS : has
    CINEMAS ||--o{ HALLS : contains
    HALLS ||--o{ SEATS : contains
    HALLS ||--o{ SHOWS : hosts
    SHOWS ||--o{ SHOW_SEATS : has
    SEATS ||--o{ SHOW_SEATS : assigned
    BOOKINGS ||--o{ BOOKING_SEATS : contains
    BOOKINGS ||--o{ SEAT_RESERVATIONS : tracks
    BOOKINGS ||--o{ PAYMENTS : has
    BOOKINGS ||--o{ BOOKING_HISTORY : records
    SHOW_SEATS ||--o| BOOKING_SEATS : confirms
```

## 8. Preventing Double Booking

Do not rely on a frontend seat map or a simple `SELECT` followed by an `UPDATE`. Two requests can both observe a seat as available.

Use multiple protections:

1. One `ShowSeats` row per `(show_id, seat_id)`.
2. `SELECT ... FOR UPDATE` to lock seat rows during reservation/finalization.
3. `UNIQUE (show_id, seat_id)` in `BookingSeats`.
4. Conditional updates that verify the expected current state and reservation ID.
5. Consistent lock ordering (for example, sorted seat IDs) to reduce deadlocks.
6. Retry the whole transaction on retryable deadlock or serialization errors.

The database is the authority for seat ownership. Cache availability can be stale and must never authorize a booking by itself.

## 9. Atomic Multi-Seat Reservation

The transaction must reserve every requested seat or none.

Illustrative PostgreSQL-style pseudocode:

```sql
BEGIN;

-- 1. Check/claim the idempotency key.
-- If this request already completed, return its saved result.

-- 2. Lock all requested seat rows in a consistent order.
SELECT show_id, seat_id, status, hold_expires_at
FROM ShowSeats
WHERE show_id = :show_id
  AND seat_id = ANY(:seat_ids)
ORDER BY seat_id
FOR UPDATE;

-- 3. Verify that every requested seat exists for this show.
-- Verify the show is bookable and all seats are AVAILABLE.
-- If any check fails: ROLLBACK and return a conflict.

-- 4. Create the booking and one reservation record per seat.
-- Set a server-controlled expiry time.

-- 5. Change all requested ShowSeats rows to HELD.
-- Check that the affected row count equals the requested seat count.

-- 6. Write BookingHistory and OutboxEvents as needed.

COMMIT;
```

This is a transaction outline, not a copy-paste-ready implementation. Production code must validate the exact seat count, current state, hold expiry, ownership, idempotency key, show status, price snapshots, and affected row counts. Any failed validation must roll back the transaction.

A practical isolation level is often PostgreSQL `READ COMMITTED` with explicit row locks and constraints. `SERIALIZABLE` is also possible, but the application must retry the entire transaction on serialization failures.

## 10. Reservation Expiry

A background worker scans expired holds in batches. Multiple workers can use `FOR UPDATE SKIP LOCKED` to avoid waiting on rows another worker has claimed.

Illustrative pattern:

```sql
BEGIN;

-- Select a batch of expired HELD rows with FOR UPDATE SKIP LOCKED.
-- Process only the exact rows returned by this transaction.

-- For each selected row, re-check:
-- status = HELD
-- hold_expires_at <= database current time
-- reservation_id still matches the expected reservation

UPDATE ShowSeats
SET status = 'AVAILABLE',
    reservation_id = NULL,
    hold_expires_at = NULL
WHERE show_id = :show_id
  AND seat_id = :seat_id
  AND status = 'HELD'
  AND reservation_id = :expected_reservation_id
  AND hold_expires_at <= clock_timestamp();

-- Update the matching ACTIVE SeatReservations and eligible Bookings.
-- Do not expire a booking that has already been confirmed.

COMMIT;
```

The worker must update the seat, reservation, and booking state in one transaction. A stale worker must never release a seat held under a newer reservation ID.

Use database time for expiry comparisons. The expiry worker is a cleanup mechanism; reservation and payment-finalization APIs must also check expiry themselves.

## 11. Payment Consistency

A payment gateway is an external system, so its operation cannot be part of the same PostgreSQL ACID transaction.

### Recommended flow

1. Create the booking and temporary seat hold in a database transaction.
2. Commit the transaction before calling the gateway.
3. Create or reuse an idempotent payment attempt.
4. Initiate payment with the provider using a stable provider idempotency key when supported.
5. Verify a signed webhook and/or query the provider server-to-server.
6. Check provider transaction ID, booking association, amount, and currency.
7. In a short database transaction, lock the booking, payment, and relevant seat rows.
8. Confirm only if the payment is verified and the original hold remains valid.
9. Atomically update the payment, booking, seat, `BookingSeats`, history, and outbox records.
10. If payment succeeded but the hold expired or the seats were reassigned, record the successful payment and initiate a tracked refund/compensation workflow.

### Confirmation transaction outline

```sql
BEGIN;

-- Lock Booking and Payment rows in a consistent order.
-- Lock the relevant ShowSeats rows.
-- Verify payment amount, currency, provider transaction ID,
-- booking association, hold expiry, reservation ID, and seat state.

-- If the original hold is still valid:
--   Payment -> SUCCEEDED
--   Create BookingSeats rows
--   ShowSeats -> BOOKED
--   Booking -> CONFIRMED
--   Write BookingHistory and OutboxEvents

-- If any condition fails, do not confirm the booking.
-- Persist the verified payment outcome and schedule reconciliation/refund
-- through a safe workflow as appropriate.

COMMIT;
```

The exact handling of a late successful payment depends on the chosen transaction boundary and schema. Never discard proof of a successful charge just because booking finalization failed. Persist the payment event/outcome durably, then reconcile the booking and refund state.

### Duplicate webhooks

- Verify the provider signature.
- Deduplicate on provider event ID.
- Make processing idempotent.
- Do not allow a stale `FAILED` event to overwrite a verified `SUCCEEDED` payment.
- Store provider event IDs and processing outcomes.
- Return a successful acknowledgement only when the event is durably accepted, according to the provider's retry contract.

### Reconciliation

Periodically query the provider for payments in `UNKNOWN` or long-running `PROCESSING` states. Reconcile provider results against local records and create alerts for mismatches. Refund requests also need idempotency and status tracking.

## 12. REST API

Base path: `/api/v1`

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/auth/register` | Register |
| `POST` | `/auth/login` | Login |
| `GET` | `/movies` | List movies |
| `GET` | `/movies/:movieId` | Movie details |
| `GET` | `/cinemas` | List cinemas |
| `GET` | `/shows?movieId=...` | Showtimes |
| `GET` | `/shows/:showId/seats` | Current seat map |
| `POST` | `/reservations` | Reserve multiple seats |
| `GET` | `/reservations/:id` | Reservation status |
| `DELETE` | `/reservations/:id` | Release an active hold |
| `POST` | `/bookings/:id/payments` | Start/reuse a payment attempt |
| `GET` | `/bookings/:id` | Booking details |
| `GET` | `/bookings` | Current user's booking history |
| `POST` | `/bookings/:id/cancel` | Request cancellation |
| `POST` | `/bookings/:id/refunds` | Request eligible refund |
| `POST` | `/payments/webhook` | Payment provider webhook |
| `POST` | `/admin/movies` | Admin movie management |
| `POST` | `/admin/shows` | Admin show management |

### Reservation request

```http
POST /api/v1/reservations
Authorization: Bearer <access-token>
Idempotency-Key: <unique-key>
Content-Type: application/json
```

```json
{
  "show_id": "show-uuid",
  "seat_ids": ["seat-a1-uuid", "seat-a2-uuid"]
}
```

Example response:

```json
{
  "booking_id": "booking-uuid",
  "booking_reference": "BK-2026-8F2K",
  "status": "PENDING_PAYMENT",
  "expires_at": "2026-10-11T15:25:00Z",
  "currency": "BDT",
  "total_amount": "500.00",
  "seats": [
    { "seat_id": "seat-a1-uuid", "unit_price": "250.00" },
    { "seat_id": "seat-a2-uuid", "unit_price": "250.00" }
  ]
}
```

The response is illustrative. The server calculates prices and expiry values; it does not trust client-supplied prices or timestamps.

### Suggested HTTP responses

| Code | Meaning |
|---|---|
| `200` | Successful read/status query |
| `201` | Resource created |
| `400` | Malformed request |
| `401` | Unauthenticated |
| `403` | Unauthorized for this operation |
| `404` | Resource not found |
| `409` | Seat conflict or idempotency-key payload conflict |
| `410` | Expired reservation token |
| `422` | Business validation failed |
| `429` | Rate limit exceeded |
| `502` | Invalid upstream response |
| `503` | Temporarily unavailable |
| `504` | Upstream timeout; payment outcome may remain unknown |

## 13. Idempotency

Use separate idempotency keys for reservation creation and payment initiation.

For each key:

1. Scope it to the authenticated user and operation.
2. Store a request hash.
3. Enforce a unique constraint.
4. Return the original result for the same key and same request.
5. Return `409 Conflict` if the same key is reused with a different request.
6. Retain the key long enough to cover client retries and provider callbacks.

A unique key alone is not sufficient for every workflow: store request status and the original response so concurrent duplicates can safely wait for or retrieve the same result.

For webhook processing, use the provider's unique event ID in a `ProcessedWebhookEvents` table.

## 14. ACID Transactions

| Property | Application |
|---|---|
| Atomicity | All seats in a reservation are held, or none are |
| Consistency | Foreign keys, unique constraints, checks, and valid state transitions |
| Isolation | Row locks and transactions serialize competing operations |
| Durability | Committed state is protected by database persistence, backups, and recovery procedures |

### Transaction boundaries

**Reservation transaction**
- Claim idempotency key.
- Lock and validate all requested seats.
- Create booking and reservations.
- Mark seats `HELD`.
- Write history/outbox records.
- Commit all changes together.

**Confirmation transaction**
- Lock booking, payment, and relevant seat rows.
- Re-validate the verified payment and hold.
- Create confirmed booking-seat records.
- Mark seats `BOOKED`.
- Update booking/payment state.
- Write history/outbox records.
- Commit all changes together.

**Expiry transaction**
- Lock and re-check expired holds.
- Mark the matching reservation expired.
- Expire eligible pending bookings.
- Release only seats belonging to that reservation.
- Commit together.

Never keep a database transaction open while waiting for an external payment gateway.

## 15. Edge Cases

| Case | Handling |
|---|---|
| Two customers reserve the same seat | Row lock; one succeeds, the other receives conflict |
| Multiple seats overlap between requests | Lock seats in a stable order; rollback the losing multi-seat request |
| A seat becomes unavailable before reservation | Validate inside the transaction, not from a stale seat map |
| Customer closes the browser | Expiry worker releases the hold |
| Hold expires during payment | Finalization checks expiry and current reservation ownership |
| Gateway succeeds but API times out | Mark outcome unknown; query/reconcile with provider |
| Duplicate webhook | Deduplicate by event ID and process idempotently |
| Payment succeeds after expiry | Record payment; do not steal seats; initiate refund/compensation |
| Two expiry workers process the same hold | `SKIP LOCKED`, conditional updates, and matching reservation IDs |
| Already-booked seat requested again | Status check plus unique constraint |
| Admin cancels a show | Stop new reservations; process existing bookings and refunds by policy |

## 16. Security

- Enforce authentication and ownership checks on every booking/payment read or mutation.
- Restrict administrative endpoints to authorized staff.
- Validate all inputs and calculate prices on the server.
- Verify webhook signatures and provider transaction details.
- Use HTTPS, secure password hashing, secret management, and appropriate token/session expiry.
- Never log passwords, payment secrets, or sensitive payment data.
- Rate-limit login, reservation, and payment endpoints.
- Audit admin actions, cancellations, and refunds.
- Protect against webhook replay using event IDs and provider verification.
- Use unpredictable ticket references or signed ticket payloads; validate ticket status at check-in.

## 17. Reliability and Scalability

- Use connection pooling and database query/index monitoring.
- Cache public movie and show listings, but never treat cached seat availability as authoritative.
- Add queue-based workers for expiry, notifications, and reconciliation.
- Use a transactional outbox for reliable event publication.
- Use read replicas for suitable read-heavy workloads, understanding replication lag.
- Retry transient failures with bounded exponential backoff and jitter.
- Make retries safe through idempotency.
- Monitor reservation conflicts, expiry lag, payment unknowns, webhook failures, refunds, and database lock waits.
- Test backup restoration and point-in-time recovery.
- Consider partitioning large history/event tables when volume justifies it.

## 18. Suggested Project Structure

```text
cinema-booking/
├── client/
│   └── src/
│       ├── components/
│       ├── features/
│       │   ├── movies/
│       │   ├── shows/
│       │   ├── seats/
│       │   ├── bookings/
│       │   └── payments/
│       ├── hooks/
│       ├── lib/
│       └── routes/
├── server/
│   └── src/
│       ├── modules/
│       │   ├── auth/
│       │   ├── users/
│       │   ├── movies/
│       │   ├── cinemas/
│       │   ├── shows/
│       │   ├── reservations/
│       │   ├── bookings/
│       │   ├── payments/
│       │   ├── refunds/
│       │   └── admin/
│       ├── workers/
│       │   ├── expiry.worker.ts
│       │   ├── reconciliation.worker.ts
│       │   └── outbox.worker.ts
│       ├── middleware/
│       ├── database/
│       ├── jobs/
│       ├── shared/
│       └── app.ts
├── database/
│   ├── migrations/
│   └── seeds/
├── tests/
│   ├── integration/
│   ├── concurrency/
│   └── e2e/
├── docker-compose.yml
├── .env.example
└── README.md
```

## 19. Local Setup

These steps assume you have Node.js, npm, and PostgreSQL installed. Adapt commands to your chosen ORM and migration tool.

1. Clone the repository.
2. Install client and server dependencies.
3. Create a PostgreSQL database.
4. Copy `.env.example` to `.env` and configure credentials.
5. Run database migrations and seed development data.
6. Start the server and client.
7. Configure a payment-provider sandbox before testing payments.

Example commands (adjust to the actual scripts in `package.json`):

```bash
git clone <repository-url>
cd cinema-booking

cd server
npm install
npm run migrate
npm run dev
```

In a second terminal:

```bash
cd client
npm install
npm run dev
```

### Example environment variables

```dotenv
NODE_ENV=development
PORT=5000
DATABASE_URL=postgresql://postgres:password@localhost:5432/cinema_booking
CLIENT_ORIGIN=http://localhost:5173

# Configure only for a supported provider and never commit real secrets.
PAYMENT_PROVIDER=
PAYMENT_API_KEY=
PAYMENT_WEBHOOK_SECRET=
```

Do not commit `.env` or real credentials. Use sandbox credentials for local development.

## 20. Testing Checklist

### Functional tests
- Register/login and access control.
- Movie, cinema, hall, show, and seat listing.
- Multi-seat reservation and pricing.
- Booking history and ticket issuance.
- Cancellation and refund eligibility.

### Concurrency tests
- Send many simultaneous requests for the same show-seat.
- Assert that no more than one confirmed booking exists.
- Request overlapping sets of multiple seats.
- Verify losing requests do not leave partial holds.
- Run multiple expiry workers concurrently.
- Test deadlock/serialization retry behavior.

### Payment tests
- Successful payment and webhook.
- Failed payment.
- Timeout with eventual provider success.
- Duplicate and out-of-order webhooks.
- Invalid webhook signature.
- Wrong amount/currency/provider transaction ID.
- Payment success after reservation expiry.
- Refund retry and reconciliation.

### Recovery tests
- Kill the API after the gateway succeeds but before local confirmation.
- Restart workers after an interrupted expiry batch.
- Replay outbox events.
- Restore a backup in a test environment.

## 21. Future Improvements

- Real-time seat-map updates using WebSockets or Server-Sent Events.
- Dynamic pricing and promotion/coupon rules.
- QR-code tickets and venue check-in.
- Multiple payment providers and provider failover.
- Multi-currency support where required.
- Analytics dashboard and sales reporting.
- Dedicated microservices only when operationally justified.

---

## Design Summary

The core safety mechanism is the combination of **PostgreSQL transactions, row-level locking, unique constraints, idempotency, verified payment events, and reconciliation**.

- Seat availability shown in the UI is advisory; the database decides.
- Multi-seat holds are atomic.
- Expiry workers release only the hold they actually own.
- A timeout is not treated as proof of payment failure.
- A late successful payment cannot reclaim seats already assigned to another customer.
- Refunds, notifications, and external events are tracked as separate reliable workflows.
