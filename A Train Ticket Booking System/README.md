
# Train Ticket Booking System --- System Design

> **Purpose:** Software Engineering Interview Preparation\
> **Main Technologies:** Node.js, Express.js, PostgreSQL\
> **Core Problem:** A user will select four seats in one booking to travel
> from Dhaka to Jashore. The seats will be held for 10 minutes to complete
> the payment.

------------------------------------------------------------------------

## Table of Contents

1.  [Problem Statement](#1-problem-statement)
2.  [Requirements](#2-requirements)
3.  [High-Level Architecture](#3-high-level-architecture)
3.3 [Important SQL for Interview](#train-ticket-booking-system--important-sql-for-interview)
    - [Core Booking Queries](#331-core-booking-queries)
    - [Search, Cancellation and Reporting Queries](#332-search-cancellation-and-reporting-queries)
    - [Useful Integrity Constraints and Indexes](#333-useful-integrity-constraints-and-indexes)
4.  [Database Design](#4-database-design)
5.  [Status Management](#5-status-management)
6.  [ER Diagram](#6-er-diagram)
7.  [REST API Design](#7-rest-api-design)
8.  [Complete Booking Flow](#8-complete-booking-flow)
9.  [Double Booking Prevention](#9-double-booking-prevention)
10. [10-Minute Hold and Expiry Mechanism](#10-10-minute-hold-and-expiry-mechanism)
11. [Payment Failure and Delayed Webhook](#11-payment-failure-and-delayed-webhook)
12. [Database Transaction](#12-database-transaction)
13. [ACID Properties](#13-acid-properties)
14. [Security and Reliability](#14-security-and-reliability)
15. [Scalability and Performance](#15-scalability-and-performance)
16. [Short Answer to Say in the Interview](#18-short-answer-to-say-in-the-interview)

------------------------------------------------------------------------

## 1. Problem Statement

Suppose a user finds a train from Dhaka to Jashore and wants to book four
seats together: `A1`, `A2`, `A3`, `A4`.

The system's tasks will be:

1.  Show trains, routes, travel dates and available seats.
2.  Reserve four seats under one booking.
3.  Hold the seats for a maximum of 10 minutes to make the payment.
4.  If the payment is successful and verified within the stipulated time,
    confirm the booking.
5.  If the payment is not successful even after the time expires, expire the
    booking and release all four seats.
6.  Ensure that the same seat does not go into the confirmed booking of two
    users at the same time.
7.  Safely handle payment failure, duplicate webhooks and delayed payment
    confirmation.

**Core design challenge:** This is not just a CRUD application. It is a
concurrent booking and payment system, where seat availability and booking
status must always be kept consistent.

## 2. Requirements

### 2.1 Functional Requirements

-   Search trains by departure station, destination and travel date.
-   View train schedule, fare and seat availability.
-   In this use case, select exactly four seats in one booking.
-   Atomically hold all selected seats.
-   Initiate and verify payment through a payment gateway.
-   If payment is successful, confirm the booking and issue the ticket.
-   Handle expiry of unpaid bookings and release the seats.
-   Show the user booking details, booking status and payment status.
-   Support cancellation and refund workflow.

### 2.2 Non-Functional Requirements

  -----------------------------------------------------------------------
  Requirement                         Explanation
  ----------------------------------- -----------------------------------
  Consistency                         The same seat of the same trip will
                                      not be allocated to more than one
                                      active booking.

  Reliability                         Inconsistencies between payment and
                                      booking state must be detected and
                                      resolved.

  Concurrency                         If many users want the same seat at
                                      the same time, requests must be
                                      processed safely.

  Security                            A user can view or cancel only
                                      their own booking.

  Scalability                         If traffic increases, API instances
                                      and workers can be increased.

  Observability                       Booking conflicts, payment
                                      failures, expiry delays and
                                      database load must be monitored.
  -----------------------------------------------------------------------

## 3. High-Level Architecture

### 3.1 Frontend Architecture

``` mermaid
flowchart TD
    USER[Passenger]
    APP[React / Next.js Web App]
    
    subgraph STEPS[User Journey Flow]
        SEARCH[Search Trains & Schedules]
        SELECT[Select Route & Seats e.g., A1-A4]
        HOLD[10-Min Temporary Hold View]
        CHECKOUT[Payment Checkout Interface]
        STATUS[Live Status Tracking / Polling]
        RESULT[Final E-Ticket & Confirmation]
    end

    API[Backend REST API / WebSockets]

    USER --> APP
    APP --> SEARCH
    SEARCH --> SELECT
    SELECT --> HOLD
    HOLD --> CHECKOUT
    CHECKOUT --> STATUS
    STATUS --> RESULT
    
    APP <-->|HTTPS REST / WebSockets| API
  
```

**Interview explanation:** The frontend will only handle the screen and user
interaction. The frontend will not make the final decision on fare, seat
availability, payment success or booking confirmation; these will come from
the backend.


### 3.3 Full System Architecture

``` mermaid

flowchart TB

    subgraph S1["① Frontend Layer"]
        U["👤 Passenger"] --> W["React / Next.js Web App"]
    end

    subgraph S2["② Backend Layer"]
        direction LR
        G["Express API / Auth"]
        B["Booking Service"]
        P["Payment Service"]
        C["Booking Confirmation"]
        T["Ticket Service"]
        N["Notification Service"]

        G --> B
        G --> P
        B --> C
        P --> C
        C --> T
        T --> N
    end

    subgraph S3["③ Database Layer"]
        direction LR
        DB[("PostgreSQL")]
        TD[("Ticket Records")]
    end

    subgraph S4["④ External Services"]
        direction LR
        PG["Payment Gateway"]
        MSG["SMS / Email Gateway"]
    end

    W -->|"HTTPS + JWT"| G

    B <-->|"Seat Lock + Booking"| DB
    C <-->|"Confirm Booking"| DB
    T -->|"Save Ticket"| TD

    P <-->|"Payment + Webhook"| PG
    N -->|"Send E-ticket / Notification"| MSG
    MSG --> U

    classDef user fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    classDef svc fill:#E6F1FB,stroke:#185FA5,color:#042C53
    classDef db fill:#EEEDFE,stroke:#534AB7,color:#26215C
    classDef ext fill:#FAEEDA,stroke:#854F0B,color:#412402

    class U user
    class W,G,B,P,C,T,N svc
    class DB,TD db
    class PG,MSG ext
```

#### Quick Reference Table (Concepts & Their Purpose)
##### Arrow flow
```mermaid

flowchart TB

    subgraph S1["① Booking Request & Seat Hold"]
        direction LR
        U["👤 Passenger"] -->|"Select seat"| W["React / Next.js Web App"]
        W -->|"HTTPS + JWT"| G["Express API / Auth"]
        G -->|"Authenticated request"| B1["Booking Service"]
        B1 -->|"BEGIN TRANSACTION"| DB1[("PostgreSQL")]
        DB1 -->|"Check & lock seat rows"| L["Seat Availability + Row Lock"]
        L -->|"Seat available"| H["Create PENDING Booking<br/>Hold expires in 10 minutes"]
        H -->|"COMMIT hold"| DB1
    end

    subgraph S2["② Payment Processing"]
        direction LR
        P1["Payment Service"] -->|"Create payment"| PG["External Payment Gateway"]
        PG -->|"Payment result / Webhook"| WH["Webhook Handler"]
        WH -->|"Verify signature + event ID"| V["Validate & Deduplicate Event"]
        V -->|"Verified successful payment"| P2["Payment Service"]
    end

    subgraph S3["③ Confirm Booking Using ACID"]
        direction LR
        B2["Booking Service<br/>Recheck hold & payment"] -->|"BEGIN TRANSACTION"| DB2[("PostgreSQL")]
        DB2 -->|"Lock booking / seat rows"| R["Validate hold, expiry,<br/>payment and seat ownership"]
        R -->|"Valid: update booking"| C["CONFIRMED"]
        C -->|"COMMIT"| DB2
    end

    subgraph S4["④ Ticket & Notification"]
        direction LR
        T["Ticket Service"] -->|"Create ticket once"| TD[("Ticket Records")]
        TD -->|"Ticket ready"| N["Notification Service"]
        N -->|"SMS / Email + E-ticket"| U2["👤 Passenger"]
    end

    H ==>|"Booking ID + hold expiry"| P1
    P2 ==>|"Confirm booking request"| B2
    DB2 ==>|"Confirmation committed"| T

    WH -.->|"Duplicate event: no repeated processing"| V
    R -.->|"Expired / invalid: reject or compensate"| X["Do not confirm<br/>Handle refund if charged"]

    classDef user fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    classDef svc fill:#E6F1FB,stroke:#185FA5,color:#042C53
    classDef ext fill:#FAEEDA,stroke:#854F0B,color:#412402
    classDef db fill:#EEEDFE,stroke:#534AB7,color:#26215C
    classDef error fill:#FCEBEB,stroke:#A32D2D,color:#501313

    class U,U2 user
    class W,G,B1,L,H,P1,V,P2,B2,R,C,T,N svc
    class PG,WH ext
    class DB1,DB2,TD db
    class X error

```

##### Step-by-step workflow ( Details )
# Step-by-Step Workflow (Detailed)

## Phase 1: Booking Request & Seat Hold

### 1. Passenger → React / Next.js Web App

The passenger first enters the web app and selects the journey date, Departure Station, Destination Station and preferred train. The app then shows the available seats and the passenger selects one or more seats and makes a booking request.

- **Frontend:** React.js / Next.js
- **Task:** Train search, seat selection, passenger information and creating the booking request.
- **Important point:** Just because a seat is shown as Available in the frontend does not mean it is definitely Available. Another passenger may try to book the seat at the same time. So the final availability check must be done in the Backend and Database.

### 2. React App → HTTPS / TLS 1.3 → Express REST API Gateway

The frontend sends the booking request to the API Gateway via HTTPS. HTTPS and TLS 1.3 help encrypt the communication between the client and the server.

**API endpoint:**

`POST /api/bookings`

The request body may contain:

- `trainId`
- `journeyDate`
- `seatIds`
- `passengerDetails`

To prevent duplicate bookings from retry or double-click, a unique `Idempotency-Key` will be sent in the request header. If the same key and the same request come again, the Server can return the previous response, so that a new booking is not created a second time.

Responsibilities of the API Gateway:

- Request routing
- Rate limiting
- Running authentication middleware
- Sending the request to the appropriate Backend Service

If a different request is sent with the same Idempotency-Key, it must not be accepted as a new booking; a conflict or validation error must be given.

### 3. API Gateway → Authentication & Request Validation

The API Gateway or the relevant Middleware verifies the passenger's identity and the validity of the request.

**Authentication:** Verify JWT or Session to confirm the passenger's identity.

**Validation:** Verify the format and validity of the Train ID, journey date, seat IDs and passenger information.

Possible responses:

- `401 Unauthorized` — Authentication missing or invalid.
- `400 Bad Request` — Request format or input invalid.
- `429 Too Many Requests` — Rate limit exceeded.

Here only whether the request is acceptable is verified. Confirming the final availability and reservation of seats is the responsibility of the Booking Service.

### 4. Authentication → Booking Service → PostgreSQL (Seat Hold)

If authentication and request validation succeed, the request is sent to the Booking Service. The Booking Service handles the booking's Business Logic and temporarily reserves the selected seats through a PostgreSQL transaction.

**Seat Hold workflow:**

1. The database transaction will start: `BEGIN`.
2. The relevant trip, journey date and seat IDs will be verified.
3. As needed, seat rows will be locked or atomically claimed using `SELECT ... FOR UPDATE` or a conditional update.
4. It will be verified whether all selected seats are Available.
5. If all seats are found, a booking will be created and the seat status will be `HELD`.
6. The booking status will be `PENDING_PAYMENT`.
7. `expires_at` will be set according to the database time, for example 10 minutes after the current time.
8. If all changes succeed, `COMMIT` will happen. If even one seat is not found, the entire transaction will `ROLLBACK` and `409 Conflict` will be returned.

**Example:**

- `bookingId`: `BK-1001`
- `bookingStatus`: `PENDING_PAYMENT`
- `seatIds`: `A1, A2, A3, A4`
- `seatStatus`: `HELD`
- `expiresAt`: 10 minutes after the database time at which the hold was created

If even one of the four seats has been allocated to another passenger, none of the four seats can be kept as a partial hold.

Appropriate constraints must be kept in the database, such as keeping the identity of each physical seat of a trip unique with `UNIQUE (trip_id, seat_number)`. As needed, additional constraints must also be kept for active seat allocation.

**Why is the 10-minute Hold important?**

It gives the passenger a limited time to complete the payment. Creating the hold and releasing the seat after hold expiry are separate database state transitions; so this rule cannot be enforced with just a Frontend timer.

#### Expiry Worker

A Cron Job or Queue Worker will regularly look for expired `PENDING_PAYMENT` bookings.

For each booking:

1. It will start a database transaction.
2. It will lock the booking and the related seat rows or do a safe conditional update.
3. It will verify, according to the database time, that the hold has really expired.
4. If the booking is still `PENDING_PAYMENT`, it will make it `EXPIRED`.
5. It will make the `HELD` seats owned by that booking `AVAILABLE`.
6. It will commit all changes together.

Even if multiple workers process the same booking, there must be no state corruption or release of another booking's seats. So consistent locking and conditional state-transition rules must be used for the expiry worker and payment confirmation.

## Phase 2: Payment Processing

### 5. Booking Service → Payment Service

If the Seat Hold succeeds, a request to start payment will be sent to the Payment Service.

The Payment Service will verify the booking ID, the determined fare and the payment request, and create a Payment Record.

**Example:**

- `paymentId`: `PAY-5001`
- `bookingId`: `BK-1001`
- `amount`: ৳800
- `paymentStatus`: `INITIATED`

The Payment Service's responsibility is to manage the payment lifecycle; the Booking Service's responsibility is to manage the booking and seat reservation.

If the two Services have separate databases, there will not be a direct common ACID transaction between them. So idempotency, retry and payment status tracking are needed in the Payment Service too.

### 6. Payment Service → External Payment Gateway

The Payment Service will initiate payment through an approved Payment Gateway such as bKash, SSLCommerz, Nagad or aamarPay.

Workflow:

1. The Backend will create a payment session with the Payment Gateway.
2. A Checkout URL or payment session will be obtained from the Gateway.
3. The Frontend will send the passenger to the Checkout page.
4. The passenger will complete the payment.
5. The Gateway will send a callback or webhook according to the payment status.

**Important:** Payment cannot be assumed successful just by seeing the Frontend's Success Page. The payment status, amount, currency and transaction reference must be verified using the Gateway's documented server-side verification API.

### 7. External Payment Gateway → Idempotent Webhook Handler

How does a Webhook work?

1 Payment Gateway → Backend

After you pay, bKash sends a message to your Backend's POST /api/webhooks/payment endpoint: "This payment was successful."

2 Verify that the message is genuine

The Backend checks whether the message really came from the Gateway and whether it has been altered. According to the Gateway's rules, signature verification or another authentication method is used.

3 Save the Event in the Database

The Backend saves the message in the Database. It also checks whether the same message has come before. Because of the UNIQUE constraint on the Event ID, the same event cannot be saved a second time as a new event.

4 Acknowledgement to the Gateway

If the message is saved safely, the Backend sends 200 OK. That is, "I received your message." This does not mean that the payment has already been verified.

5 The Worker Verifies the Payment

A Background Worker verifies through the Gateway's server-side API whether the payment was really successful, whether the amount is ৳500 and whether the transaction reference is correct. If everything is correct, the Payment Service makes the payment SUCCEEDED.

What will happen if the same message comes three times?
Suppose the Gateway mistakenly sends the same payment success message three times.
Event	What will the Backend do?
First time	Will save the event and process it
Second time	Will identify the same Event ID and treat it as a duplicate
Third time	Will not process the payment again
This reduces the risk of the same payment being recorded multiple times or tickets being created repeatedly for the same booking.

Remember two things:
- Webhook Idempotency: Even if the same message comes repeatedly, do not process it anew.
- Payment Idempotency: The same gateway transaction must not be recorded as a successful payment more than once.

Making only the Event ID unique is not enough. Even if the same transaction comes with a different event ID, the payment record must not be made a second time, so uniqueness must also be ensured on the gateway transaction reference. For ticket creation too, duplicate creation must be prevented with the booking ID or a ticket key

### 8. Webhook Handler → Payment Service (Payment Verification)

After receiving the event from the asynchronous worker, the Payment Service will verify the Gateway's authoritative payment record.

The things to verify:

- Whether the payment reference is correct.
- Whether the payment is linked to the relevant booking.
- Whether the paid amount equals the expected fare.
- Whether the currency is correct.
- Whether the Gateway's transaction status is successful.
- Whether the same gateway transaction has already been used in another payment record.

If all verifications succeed, the state of the Payment Record will be made `SUCCEEDED`. If the payment fails, an appropriate failure status will be kept.

The Payment Service will then send the verified payment result to the Booking Service. This can be done through an internal API or durable event delivery.

It must be ensured that even if payment events are duplicate or out-of-order, the status transition is not changed incorrectly.

## Phase 3: Booking Confirmation Using ACID

### 9. Payment Service → Booking Service (Confirm Request)

The Payment Service sends the verified payment success information to the Booking Service.

Before confirmation, the Booking Service will verify:

- Whether the Booking ID is correct.
- Whether the payment is truly verified.
- Whether the payment amount and booking fare match.
- Whether the booking is still `PENDING_PAYMENT`.
- Whether the seat hold is still valid.
- Whether the hold has expired.
- Whether the selected seats are still owned by the relevant booking.
- Whether the booking has already been `CONFIRMED`.

**Late Payment Scenario:**

Suppose a passenger did not make the payment within 10 minutes. The Expiry Worker made the booking `EXPIRED` and the seats became Available for another passenger. After that, the first passenger's payment-success event came late.

In this situation, the Booking Service will not directly confirm the booking.

- If the seat is allocated to another booking, the confirmation must be rejected.
- If the payment has already succeeded, `REFUND_PENDING` or an appropriate compensation workflow must be started.
- The Refund status must also be tracked separately.
- It must be ensured that even if the payment event or internal request comes again, there is no duplicate refund or duplicate confirmation.

Even if the seat has not been allocated to someone else yet, whether the expired booking will be automatically confirmed will depend on a clear business policy. The simplest and safest policy is to not reuse the expired hold and start a new reservation flow.

If the Payment Service and Booking Service use separate databases, payment success and booking confirmation will not be in a common ACID transaction. To make this distributed workflow reliable, durable event delivery, retry, idempotency and a reconciliation job are needed.

#### Mutex (Full form: Mutual Exclusion):
is a very important concept or mechanism of Concurrent Programming.

In simple words, it is a Digital Lock.

When multiple threads or processes try to access or modify the same shared resource—such as a memory location, variable, file or database table—at the same time, a Mutex is used to avoid data conflicts or Race Conditions.

How exactly Mutex works in a train ticket booking system and why it is so essential is explained below with a real example:

1. Where does the problem occur? (Race Condition) Suppose only one seat, seat number A1, is empty on a train going from Dhaka to Jashore. At exactly the same second (the same millisecond), two passengers—Rahim and Karim—send a request to book that seat.

If there is no lock or Mutex in the system, the backend server will do two things at the same time: Rahim will see the seat is empty  will book the seat. Karim will also see the seat is empty  he will also book the seat. The result? The same seat will be confirmed in the names of two passengers (Double Booking), which is not acceptable in any railway system.

2. How Mutex works here. To solve this problem, when requests arrive at the server, a Mutex (lock) is used:

Lock Acquire: After Rahim's and Karim's requests arrive, the system first tries to take a lock on the Mutex. Suppose Rahim's request reached a fraction of a millisecond earlier and he locked the Mutex.

Critical Section (Execution): Now Rahim checked on the server whether seat A1 is vacant. Since it was vacant, he held or booked it in his name.

Wait / Block: At that very moment, when Karim wanted to check the seat, the system saw that the Mutex was already held by Rahim (locked). So Karim's request was kept in a waiting state (Blocked).

Unlock & Result: After Rahim's booking process finished, the Mutex was released (Unlocked). Now Karim's request got its chance. Karim checked the seat and saw that seat A1 had already been BOOKED. As a result, the system informed Karim, "Sorry, this seat is no longer available."

### 10. Booking Service → PostgreSQL Primary Database (Atomic Confirmation)

This is one of the most important steps of the whole system. The Booking Service will finalize the booking and seat allocation through a PostgreSQL transaction.

**Confirmation transaction workflow:**

1. `BEGIN` — Start database transaction.
2. Lock or do a safe conditional update on the relevant Booking and Seat rows.
3. Verify booking status, hold expiry and seat ownership.
4. Verify confirmation eligibility based on the verified payment reference and payment status.
5. Make the booking status `CONFIRMED`.
6. Make the relevant seat status `BOOKED`.
7. Create the Ticket Record.
8. In the same database transaction, insert a ticket-processing or notification event into the `outbox_events` table.
9. `COMMIT` — If all changes succeed, save them permanently together.
10. If any database operation fails, `ROLLBACK`.

**Example:**

- `bookingStatus`: `CONFIRMED`
- `seatStatus`: `BOOKED`
- `ticketStatus`: `PENDING_ISSUANCE`

Here, although the Ticket Record is created within the transaction, PDF/QR generation and sending SMS/email will not be done within the transaction. These will happen later through asynchronous processing.

For the Transactional Outbox to work correctly, the Ticket Record and `outbox_events` must be in the same database transaction. If the Ticket Service has a separate database, the Booking Service will commit the confirmation and the Outbox event in its own database; then the Ticket Service will consume the event and create the ticket in its own database.

**ACID Properties:**

- **Atomicity:** The changes of the relevant database transaction will be complete, otherwise they will roll back.
- **Consistency:** Database constraints, seat ownership and booking state rules will be maintained.
- **Isolation:** Concurrent booking or confirmation requests cannot allocate the same seat inconsistently.
- **Durability:** If the commit succeeds, the booking, seat and ticket record will remain permanently stored in the database.

It must be remembered that ACID applies within the boundary of one database transaction. The entire distributed workflow including the Payment Gateway and separate Microservices does not automatically become one ACID transaction.

## Phase 4: Ticket & Notification

### 11. Booking Confirmation → Ticket Processing / Notification Service

The Ticket Record was created in the database transaction of Step 10. Now the Outbox Worker will read pending events from the `outbox_events` table and start further processing.

Workflow:

1. The Outbox Worker will collect pending events.
2. The Ticket Processing Service will create the PDF or QR ticket using the necessary booking and passenger data.
3. If ticket generation succeeds, the ticket status will be made `ISSUED`.
4. The notification event will be sent to the Notification Service.
5. If processing fails, it will be retried; if needed, a dead-letter queue and monitoring will be used.

Even if the same event is processed multiple times, a duplicate ticket cannot be created for the same booking. For this, an appropriate unique constraint per booking or booking-seat and idempotent processing must be kept in the Ticket Record.

The Transactional Outbox reduces the failure risk between the database commit and event delivery. However, duplicate delivery by the worker is possible, so the downstream consumer must also be idempotent.

### 12. Notification Service → SMS / Email Gateway

The Notification Service will send the booking confirmation and ticket information to the passenger using an external Email or SMS Provider.

- **Email Gateway:** Can send ticket details, a download link or a PDF.
- **SMS Gateway:** Can send the booking confirmation and Ticket Reference.
- **Retry Mechanism:** If there is a temporary provider failure, it will try sending again.
- **Idempotency:** It will prevent duplicate notifications caused by retry as much as possible.

If the notification fails, the booking should not be cancelled. If the booking is already `CONFIRMED`, the notification can be retried.

### 13. SMS / Email Gateway → Passenger

The passenger will receive the booking confirmation and ticket information by SMS or email. The ticket can also be viewed or downloaded from the My Bookings page of the web app.

During ticket verification, the validity of the ticket must be verified from the Backend using the Ticket ID or QR Code. The QR Code should not contain sensitive passenger information that could be misused if publicly exposed.

At this stage, the booking, payment and ticket records are kept stored. As a result, Ticket Verification, Refund, Cancellation and Audit become possible in the future.

## Simple End-to-End Story

A passenger searches for a train and selects four seats—A1, A2, A3 and A4. The Frontend sends a booking request to the API Gateway via HTTPS. If authentication and validation succeed, the Booking Service atomically holds the four seats within a PostgreSQL transaction and creates a 10-minute `PENDING_PAYMENT` booking.

Then the Payment Service initiates payment and the passenger completes the payment at the Gateway. The Gateway's webhook is stored durably; an asynchronous worker verifies the payment using the Gateway's authoritative API.

When the payment is verified, the Booking Service starts a database transaction again and verifies booking expiry, seat ownership and payment validity. If all conditions are met, the booking becomes `CONFIRMED`, the seats become `BOOKED` and the Ticket Record and an `outbox_events` row are created in the same transaction.

The Outbox Worker later runs ticket PDF/QR generation and notification processing. Even if the notification fails, the booking remains confirmed and can be retried.

On the other hand, if the payment is not completed within 10 minutes, the Expiry Worker, within a transaction, makes the booking `EXPIRED` and releases the `HELD` seats of that booking by making them `AVAILABLE`. Even if a payment success event arrives later, the expired booking is not directly confirmed; a refund or reconciliation workflow is run according to policy.

#### Summarized Table for Quick Reference ( Interview Answer )

Junior Software Engineer · Easy–Medium System Design

## Step-by-step workflow

##### step: 1\. Passenger → React / Next.js

The Passenger will select the train, journey date and seats and send a booking request. Just because a seat is shown as Available in the frontend does not mean it is definitely available, because concurrent bookings can happen.

##### step: 2\. Frontend → HTTPS / TLS 1.3 → API Gateway

The Frontend will send a `POST /api/bookings` request. HTTPS will encrypt the data. I will use an `Idempotency-Key` to prevent duplicate bookings from retry or double-click.

##### step: 3\. API Gateway → Authentication & Validation

I will do Authentication with JWT/Session and request validation. Rate limiting and routing can also be here.

##### step: 4\. API Gateway → Booking Service → PostgreSQL

The Booking Service will verify seat availability using a transaction and row locking (`SELECT ... FOR UPDATE`) or a conditional update. If all requested seats are available, it will make them `HELD` and create a `PENDING_PAYMENT` booking, whose expiry is 10 minutes. Unique constraints will prevent double allocation. If any seat is not found, I will roll back the entire transaction and give `409 Conflict`.

Expiry Worker: A Cron/Queue Worker will make an expired booking `EXPIRED` and make the related held seats `AVAILABLE`. I will keep concurrent expiry processing safe with row locking and conditional state transition.

##### step: 5\. Booking Service → Payment Service

The Payment Service will verify the booking's amount and create an `INITIATED` Payment Record. The Booking Service will manage the seat/booking and the Payment Service will manage the payment.

##### step: 6\. Payment Service → External Payment Gateway

The Backend will initiate payment at a Gateway like bKash/SSLCommerz and give checkout information to the Frontend. I will not assume payment is successful just by seeing the success page.

##### step: 7\. Payment Gateway → Idempotent Webhook Handler

The Gateway will send a payment notification. I will verify the signature/authenticity and store the event in the database. I will prevent duplicate processing from duplicate webhooks using a unique `event_id` and idempotency.

##### step: 8\. Webhook Handler → Payment Service

The Payment Service will verify the status, transaction reference, amount and currency according to the Gateway's rules. If successful, the payment will become `SUCCEEDED`. If the status is uncertain, I will do reconciliation.

##### step: 9\. Payment Service → Booking Service

I will send the verified payment success event to the Booking Service. I will re-verify whether the booking is still `PENDING_PAYMENT`, seat ownership is correct and the hold is valid. If the hold has expired and the seat has gone to another booking, I will not confirm the booking; if the payment has been charged, I will start the `REFUND_PENDING` workflow.

##### step: 10\. Booking Service → PostgreSQL Primary Database

In one transaction, I will make the booking `CONFIRMED`, the seats `BOOKED`, and create the Ticket Record and `outbox_events`. If all succeed, `COMMIT`, otherwise `ROLLBACK`.

ACID is important here:

- Atomicity (everything will happen, or nothing will): All the work of one transaction will succeed, otherwise no work will be saved.
Example: During ticket booking, the seat was booked, but the payment record was not saved. If the transaction rolls back, the seat booking will also be cancelled.
- Consistency (Database rules will remain correct): After every transaction, the database's rules must remain correct.
Example: If a train has 40 seats, the 41st booking cannot be made.
- Isolation (No problem even when working together): Even if multiple users book at the same time, the database will not allow a wrong result.
Example: Even if two people request the same seat together, only one will get the seat.
- Durability (Once saved, it stays): Once a transaction is successfully committed, the data will remain stored in the database, even if the server restarts.
Example: Even if the server shuts down after the ticket booking is successful, the booking data will not be lost.

##### step: 11\. Ticket / Notification Service → SMS / Email Gateway

The Outbox Worker will process the event and create the Ticket PDF/QR and send notification. If it fails, it will retry; a confirmed booking will not be cancelled due to notification failure.

##### step: 12\. SMS / Email Gateway → Passenger

The Passenger will receive the confirmation and ticket details. The ticket can be viewed from `My Bookings` and its validity can be verified in the backend with the QR/Ticket ID.

**Simple end-to-end story:** The user searches for a train and selects four seats.
The Frontend sends a booking request to the API. The Backend locks the
four seats in a PostgreSQL transaction, gives them `HELD` status and creates
a 10-minute `PENDING_PAYMENT` booking. If the payment is successful and
verified, the booking becomes `CONFIRMED` and the seats become `BOOKED`. If
there is no payment, the expiry worker expires the booking and releases the
seats.


#### Isn't Payment happening before Booking?
##### Arrow flow
```mermaid
flowchart TD
    A["1. Booking Service<br/>Seat Check & Hold<br/>Status:PENDING_PAYMENT"]
    B["2. Payment Service<br/>Create Payment Record<br/>Status:INITIATED"]
    C["3. Payment Gateway<br/>Passenger Completes Payment"]
    D["4. Webhook Handler<br/>Verify Payment Success"]
    E["5. Booking Service<br/>Check Seat Hold Validity<br/>Status:CONFIRMED"]

    A --> B
    B --> C
    C --> D
    D --> E
```
##### So when does the Booking happen?
Two separate things:
- Booking creation: Happens before the payment. The seat is held for 10 minutes so that no one else can book it.
- Booking confirmation: Happens after the payment is successfully verified. Then the seat is permanently BOOKED.
Why this method?

Suppose you want to book 4 seats. The Backend will first hold the seats for 10 minutes. Then you will make the payment. If the payment is successful, the booking will be confirmed. If you do not pay, the hold will expire and the seats will be available again.

1. Pending Booking & Seat Hold: The Backend verifies that the seats are empty, temporarily holds them for 10 minutes and creates a PENDING_PAYMENT booking.

2. Payment Initiation: The Payment Service creates a payment record and gives the Passenger the opportunity to pay at the payment gateway.

3. Payment Verification: Through the payment gateway's webhook or status check, the Backend verifies whether the payment was really successful.

4. Booking Confirmation: The Backend re-verifies whether the seat's Hold is still effective. If everything is fine, it makes the Booking CONFIRMED and the Seat BOOKED through a Database Transaction.

5. Ticket & Notification: After the booking is confirmed, an Asynchronous Worker creates the ticket and sends SMS/Email to the Passenger

### Train Ticket Booking System — Important SQL for Interview

These queries are written assuming PostgreSQL. The queries for booking
confirmation, expiry, cancellation and seat release must be run within the
same transaction. `NOW()` uses database time, so there will be no dependence
on the frontend's clock.

### 3.3.1 Core Booking Queries

#### 1. Seat Lock — So that another user cannot book the same seat at the same time

```sql
SELECT *
FROM trip_seats
WHERE trip_id = 101
    AND id IN (101, 102)
    AND status = 'AVAILABLE'
ORDER BY id
FOR UPDATE;
```

`FOR UPDATE` locks the selected rows until the end of the transaction. The
Application must check that all requested seats were found; if not, it must
`ROLLBACK` and give `409 Conflict`.

#### 2. Create Booking — Create a booking and give 10 minutes for payment

```sql
INSERT INTO bookings (user_id, trip_id, status, total_amount, expires_at)
VALUES (1, 101, 'PENDING_PAYMENT', 800.00,
                NOW() + INTERVAL '10 minutes')
RETURNING id, status, expires_at;
```
This query creates a new booking worth 800 taka for a specific user and trip, saves it in the database with PENDING_PAYMENT status for 10 minutes, and returns the id, status and expires_at of the created booking.

#### 3. Hold Seats — Temporarily reserve seats before payment

```sql
UPDATE trip_seats
SET status = 'HELD',
        booking_id = 1,
        hold_expires_at = NOW() + INTERVAL '10 minutes'
WHERE trip_id = 101
    AND id IN (101, 102)
    AND status = 'AVAILABLE';
```

If the affected row count is not equal to the requested seat count, the
transaction must be rolled back. In the case of four seats, a partial hold
cannot be kept.

#### 4. Payment Success — Confirm the booking if payment is successful

```sql
UPDATE bookings
SET status = 'CONFIRMED',
        expires_at = NULL
WHERE id = 1
    AND status = 'PENDING_PAYMENT'
    AND expires_at > NOW()
RETURNING id, status;
```

Before this update, the payment status, amount and gateway transaction must
be verified server-side. If the affected row count is zero, the booking may
be already confirmed, expired or invalid.

#### 5. Book Seats — Permanently reserve the seats of the confirmed booking

```sql
UPDATE trip_seats
SET status = 'BOOKED',
        hold_expires_at = NULL
WHERE booking_id = 1
    AND status = 'HELD'
RETURNING id, seat_number, status;
```

The booking update and seat update must be done in the same confirmation transaction.

#### 6. Expire Booking — Expire the booking when the payment time ends

```sql
UPDATE bookings
SET status = 'EXPIRED'
WHERE id = 1
    AND status = 'PENDING_PAYMENT'
    AND expires_at <= NOW()
RETURNING id, status;
```

If the worker processes many bookings at once, candidates can be claimed
with `FOR UPDATE SKIP LOCKED`. If payment success comes before expiry, the
payment flow and the expiry flow will lock the same booking row and
re-verify the state.

#### 7. Release Seats — Make the seats of the expired booking available again

```sql
UPDATE trip_seats
SET status = 'AVAILABLE',
        booking_id = NULL,
        hold_expires_at = NULL
WHERE booking_id = 1
    AND status = 'HELD';
```

Making the booking `EXPIRED` and releasing the seats must be done in the same transaction.

#### 8. Check Payment — See the payment status of the booking

```sql
SELECT status, amount, gateway_transaction_id, created_at
FROM payments
WHERE booking_id = 1
ORDER BY created_at DESC;
```

#### 9. Prevent Duplicate Webhook — So that the same webhook event is not processed twice

```sql
INSERT INTO payment_webhook_events (provider, event_id, payload)
VALUES ('BKASH', 'evt_123', '{}'::jsonb)
ON CONFLICT (provider, event_id) DO NOTHING
RETURNING id;
```

If no row comes in `RETURNING`, the event has been processed or recorded
before. For this, the `UNIQUE (provider, event_id)` constraint must exist.

#### 10. Find Available Seats — See which seats are available

```sql
SELECT id, seat_number, fare
FROM trip_seats
WHERE trip_id = 101
    AND status = 'AVAILABLE'
ORDER BY seat_number;
```

This query is for display. The final availability must be re-verified within
the booking transaction.

### 3.3.2 Search, Cancellation and Reporting Queries

#### 11. Search Trains — Find trains on a specific route and date

```sql
SELECT tr.id, t.train_number, t.train_name,
             tr.departure_station, tr.arrival_station,
             tr.departure_time, tr.arrival_time
FROM trips tr
JOIN trains t ON t.id = tr.train_id
WHERE tr.departure_station = 'Dhaka'
    AND tr.arrival_station = 'Jashore'
    AND tr.departure_time >= TIMESTAMPTZ '2026-11-01 00:00:00+00'
    AND tr.departure_time <  TIMESTAMPTZ '2026-11-02 00:00:00+00'
ORDER BY tr.departure_time;
```

#### 12. Booking History — See all bookings of a Passenger

```sql
SELECT id, trip_id, status, total_amount, expires_at, created_at
FROM bookings
WHERE user_id = 1
ORDER BY created_at DESC;
```

#### 13. Booking Details — See train and journey information along with the booking

```sql
SELECT b.id AS booking_id,
             b.status AS booking_status,
             b.total_amount,
             t.train_name,
             tr.departure_station,
             tr.arrival_station,
             tr.departure_time,
             tr.arrival_time
FROM bookings b
JOIN trips tr ON tr.id = b.trip_id
JOIN trains t ON t.id = tr.train_id
WHERE b.id = 1
    AND b.user_id = 1;
```

Filtering by `user_id` is important for authorization.

#### 14. Cancel Booking — Cancel a pending booking

```sql
BEGIN;

UPDATE bookings
SET status = 'CANCELLED'
WHERE id = 1
    AND user_id = 1
    AND status = 'PENDING_PAYMENT'
RETURNING id;

UPDATE trip_seats
SET status = 'AVAILABLE',
        booking_id = NULL,
        hold_expires_at = NULL
WHERE booking_id = 1
    AND status = 'HELD';

COMMIT;
```

If a confirmed booking is cancelled, a separate refund policy and
`REFUND_PENDING` workflow are needed.

#### 15. Refund Status — See the state of the refund

```sql
SELECT booking_id, amount, status, gateway_transaction_id
FROM payments
WHERE booking_id = 1
    AND status IN ('REFUND_PENDING', 'REFUNDED')
ORDER BY created_at DESC;
```

#### 16. Count Available Seats — See how many seats are available

```sql
SELECT COUNT(*) AS available_seats
FROM trip_seats
WHERE trip_id = 101
    AND status = 'AVAILABLE';
```

#### 17. Total Revenue — Calculate total revenue from successful payments

```sql
SELECT COALESCE(SUM(amount), 0) AS gross_revenue
FROM payments
WHERE status = 'SUCCEEDED';
```

This is gross successful payment. To find net revenue excluding refunds, the
refund amount or refund transactions must be calculated separately.

#### 18. Prevent Double Booking — Prevent the same seat of the same trip from being in more than one active booking

If allocation `status` (`HELD`/`BOOKED`) is kept in the `booking_seats`
table, then to not allow active allocation to be duplicated even while
keeping historical rows:

```sql
CREATE UNIQUE INDEX unique_active_seat_allocation
ON booking_seats (trip_seat_id)
WHERE status IN ('HELD', 'BOOKED');
```

In the current schema of this README, ownership is kept in
`trip_seats.booking_id` and `trip_seats.status`. In that design,
`UNIQUE (trip_id, seat_number)` and row locking/conditional update are the
main protection; the same ownership should not be stored separately in two
places.

#### 19. Transaction — Make multiple SQL operations succeed or fail together

```sql
BEGIN;

UPDATE bookings
SET status = 'CONFIRMED',
        expires_at = NULL
WHERE id = 1
    AND status = 'PENDING_PAYMENT'
    AND expires_at > NOW();

-- Application checks affected row count.
-- Then update the associated HELD seats and insert the ticket.

COMMIT;
-- On failure, execute ROLLBACK instead.
```

#### 20. Second Highest Fare — A common problem in SQL interviews

```sql
SELECT MAX(fare) AS second_highest_fare
FROM trip_seats
WHERE fare < (
        SELECT MAX(fare)
        FROM trip_seats
);
```

This finds the distinct second highest fare. If there is no second distinct
value, the result will be `NULL`.

#### 21 : Find the 2nd Highest Salary
```sql
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
);
```
### 3.3.3 Useful Integrity Constraints and Indexes

```sql
ALTER TABLE trip_seats
ADD CONSTRAINT unique_trip_seat
UNIQUE (trip_id, seat_number);

CREATE INDEX idx_bookings_expiry
ON bookings (expires_at)
WHERE status = 'PENDING_PAYMENT';

CREATE UNIQUE INDEX unique_webhook_event
ON payment_webhook_events (provider, event_id);
```

Database constraints prevent duplicate data, but they do not create a single
ACID transaction between the payment gateway and the database. So webhook
idempotency, retry and reconciliation workers are still needed.

### 3.4 Responsibilities of Components

-   **Client (React/Next.js):** Shows train search, seat selection,
    payment checkout and booking status.
-   **Express API:** Receives REST requests; runs authentication,
    validation and business logic.
-   **Booking Service:** Enforces seat hold, booking create,
    confirmation, cancellation and expiry rules.
-   **PostgreSQL:** Keeps the authoritative data of trains, trips, seats,
    bookings and payments.
-   **Ticket Service:** Handles ticket issue and ticket lookup after
    verified payment.
-   **Payment Gateway:** Processes payments and sends server-to-server
    webhooks.
-   **Background Worker:** Finds expired bookings, verifies payment state
    and releases seats if needed.

## 4. Database Design

Eight main tables:

1.  `users`
2.  `trains`
3.  `trips`
4.  `trip_seats`
5.  `bookings`
6.  `booking_seats`
7.  `payments`
8.  `tickets`

### 4.1 `users`

Keeps user account information.

  Column         Type              Description
  -------------- ----------------- -------------------
  `id`           UUID, PK          Unique user ID
  `name`         VARCHAR           User's name
  `email`        VARCHAR, UNIQUE   User's email
  `created_at`   TIMESTAMPTZ       Account creation time

### 4.2 `trains`

Keeps the general information of trains.

  Column           Type              Description
  ---------------- ----------------- -----------------
  `id`             UUID, PK          Unique train ID
  `train_number`   VARCHAR, UNIQUE   Train number
  `train_name`     VARCHAR           Train name

### 4.3 `trips`

The journey of a train on a specific date and time is called a trip. For
different dates of the same train, there will be different trip records.

  Column                Type          Description
  --------------------- ------------- -----------------------
  `id`                  UUID, PK      Unique trip ID
  `train_id`            UUID, FK      `trains.id` reference
  `departure_station`   VARCHAR       Station where the journey starts
  `arrival_station`     VARCHAR       Destination station
  `departure_time`      TIMESTAMPTZ   Departure date/time
  `arrival_time`        TIMESTAMPTZ   Arrival date/time

### 4.4 `trip_seats`

Keeps the availability of each seat of a specific trip. The same physical
seat can be available separately on another trip.

  Column              Type                Description
  ------------------- ------------------- ---------------------------
  `id`                UUID, PK            Unique trip-seat ID
  `trip_id`           UUID, FK            `trips.id` reference
  `seat_number`       VARCHAR             e.g. `A1`, `A2`
  `status`            VARCHAR/ENUM        Seat status
  `hold_expires_at`   TIMESTAMPTZ, NULL   Hold deadline
  `booking_id`        UUID, NULL, FK      Current hold/booking owner
  `fare`              DECIMAL             Seat fare

**Constraint:** `UNIQUE (trip_id, seat_number)` must be kept.

### 4.5 `bookings`

Keeps the booking's overall state, user, total amount and expiry deadline.

  Column           Type           Description
  ---------------- -------------- -----------------------------
  `id`             UUID, PK       Unique booking ID
  `user_id`        UUID, FK       `users.id` reference
  `trip_id`        UUID, FK       `trips.id` reference
  `status`         VARCHAR/ENUM   Booking lifecycle status
  `total_amount`   DECIMAL        Total fare of the selected seats
  `expires_at`     TIMESTAMPTZ    10-minute deadline
  `created_at`     TIMESTAMPTZ    Booking creation time

### 4.6 `booking_seats`

Links multiple seats to one booking. For four seats, **one booking row and
four `booking_seats` rows** will be created.

  Column           Type          Description
  ---------------- ------------- ----------------------------------
  `booking_id`     UUID, PK/FK   `bookings.id` reference
  `trip_seat_id`   UUID, PK/FK   `trip_seats.id` reference
  `fare`           DECIMAL       Seat fare saved at the time of booking

Composite primary key: `PRIMARY KEY (booking_id, trip_seat_id)`.

Apart from this, the rule that the same trip-seat is not assigned to more
than one active booking must be ensured through database constraints and
allocation logic.

### 4.7 `payments`

Keeps the payment attempt and the result of the gateway transaction.

  --------------------------------------------------------------------------
  Column                     Type                    Description
  -------------------------- ----------------------- -----------------------
  `id`                       UUID, PK                Unique payment ID

  `booking_id`               UUID, FK                `bookings.id` reference

  `gateway_transaction_id`   VARCHAR, UNIQUE         Gateway transaction
                                                     reference

  `amount`                   DECIMAL                 Payment amount

  `status`                   VARCHAR/ENUM            Payment status

  `created_at`               TIMESTAMPTZ             Payment attempt
                                                     creation time
  --------------------------------------------------------------------------

For one booking, there can be multiple payment attempts as needed.
Gateway transactions and events must be processed idempotently.

### 4.8 `tickets`

Keeps the information for issuing a ticket for a confirmed booking after
payment is verified.

Column           Type              Description
--------------   ----------------  ------------------------------
`id`             UUID, PK          Unique ticket ID
`booking_id`     UUID, FK          `bookings.id` reference
`ticket_number`  VARCHAR, UNIQUE   Public ticket number
`status`         VARCHAR/ENUM      Ticket lifecycle status
`issued_at`      TIMESTAMPTZ       Time the ticket was issued

A ticket can be issued only after verified payment and a `CONFIRMED` booking.

## 5. Status Management

The status of Booking, Seat, Payment and Ticket must be kept separate.
Because their lifecycles and meanings are not the same.

### 5.1 Booking Status

  Status              Meaning
  ------------------- ------------------------------------------------------
  `PENDING_PAYMENT`   Booking has been created; payment is not yet verified.
  `CONFIRMED`         Payment verified and booking confirmation successful.
  `EXPIRED`           Payment deadline has passed and the booking has expired.
  `FAILED`            Booking process failed, according to policy.
  `CANCELLED`         Booking has been cancelled.
  `REFUND_PENDING`    Refund processing is ongoing.
  `REFUNDED`          Refund has been completed.

### 5.2 Seat Status

  Status        Meaning
  ------------- -----------------------------------------------
  `AVAILABLE`   Seat is empty for reservation.
  `HELD`        Seat is temporarily held for a booking.
  `BOOKED`      Seat is allocated for a confirmed booking.
  `BLOCKED`     Seat has been closed by the operator/admin.

### 5.3 Payment Status

  Status             Meaning
  ------------------ -----------------------------------------------
  `PENDING`          Payment initiated or awaiting the final result.
  `SUCCEEDED`        Gateway has confirmed the payment as successful.
  `FAILED`           Payment failed.
  `CANCELLED`        Payment attempt cancelled.
  `REFUND_PENDING`   Refund in process.
  `REFUNDED`         Refund completed.

### 5.4 Ticket Status

  Status         Meaning
  -------------- ---------------------------------
  `NOT_ISSUED`   Ticket has not been issued yet.
  `ISSUED`       Ticket has been issued.
  `USED`         Ticket has been used for the journey.
  `CANCELLED`    Ticket has been cancelled.

### 5.5 Status Flow Diagram

``` mermaid
flowchart TD
    A[Seat AVAILABLE] -->|User requests booking| B[ACID Transaction: <br/>Seat HELD + Booking PENDING_PAYMENT + Payment PENDING]
    
    B -->|10 Mins Timer Starts| C{Timer / Event Trigger}
    
    %% Success Flow
    C -->|Webhook: Success| D[Booking CONFIRMED]
    D --> E[Seat BOOKED]
    E --> F[Ticket ISSUED]
    F --> G[Ticket USED]
    
    %% Cancellation Flow
    D -->|Cancellation Requested| H[Booking CANCELLED]
    H --> I[Payment REFUND_PENDING]
    I --> J[Payment REFUNDED]
    
    %% Expiry / Timeout Flow
    C -->|Timeout / Expiry Worker| K[Booking EXPIRED]
    K --> L[Release Seats to AVAILABLE]
```
Just because Payment is `SUCCEEDED`, the booking cannot be automatically
assumed `CONFIRMED`. The Backend must verify reservation validity, expiry
policy and seat ownership.

## 6. ER Diagram

``` mermaid
erDiagram
    USERS ||--o{ BOOKINGS : "creates"
    TRAINS ||--o{ TRIPS : "operates"
    TRIPS ||--o{ TRIP_SEATS : "contains"
    TRIPS ||--o{ BOOKINGS : "receives"
    
    BOOKINGS ||--|{ BOOKING_SEATS : "includes"
    TRIP_SEATS ||--|{ BOOKING_SEATS : "references"
    
    BOOKINGS ||--o{ PAYMENTS : "has"
    BOOKINGS ||--o{ TICKETS : "issues"

    USERS {
        uuid id PK
        string name
        string email UK
        datetime created_at
    }

    TRAINS {
        uuid id PK
        string train_number UK
        string train_name
    }

    TRIPS {
        uuid id PK
        uuid train_id FK
        string departure_station
        string arrival_station
        datetime departure_time
        datetime arrival_time
    }

    TRIP_SEATS {
        uuid id PK
        uuid trip_id FK
        string seat_number
        string status
        datetime hold_expires_at
        uuid booking_id FK
        decimal fare
    }

    BOOKINGS {
        uuid id PK
        uuid user_id FK
        uuid trip_id FK
        string status
        decimal total_amount
        datetime expires_at
        datetime created_at
    }

    BOOKING_SEATS {
        uuid booking_id PK,FK
        uuid trip_seat_id PK,FK
        decimal fare
    }

    PAYMENTS {
        uuid id PK
        uuid booking_id FK
        string gateway_transaction_id UK
        decimal amount
        string status
        datetime created_at
    }

    TICKETS {
        uuid id PK
        uuid booking_id FK
        string ticket_number UK
        string status
        datetime issued_at
    }
```
**Relationship summary:**

-   A user can make many bookings.
-   A train can have multiple trips.
-   A trip has multiple trip-seats.
-   A booking can include multiple seats.
-   A booking can have multiple payment attempts.
-   A ticket is issued for a confirmed booking.

In the implementation, `trip_seats.booking_id` can indicate the current
hold/confirmed owner; when the seat is released, it must be cleared. How the
historical relationship of `booking_seats` will be preserved must be clearly
defined, so that there is no duplicate allocation.

## 7. REST API Design

The following REST APIs support passenger authentication, train search, seat booking, payment processing, and ticket verification.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/login` | Passenger login |
| `GET` | `/api/trains` | Search trains |
| `GET` | `/api/trips/:tripId/seats` | Check seat availability |
| `POST` | `/api/bookings` | Create booking and hold seats temporarily |
| `GET` | `/api/bookings/:id` | Get booking details and status |
| `POST` | `/api/bookings/:id/payment` | Initiate payment |
| `GET` | `/api/bookings/:id/payment-status` | Check payment status |
| `POST` | `/api/payments/webhook` | Receive payment gateway notifications |
| `POST` | `/api/bookings/:id/cancel` | Request booking cancellation |
| `GET` | `/api/tickets/:ticketNumber` | Retrieve ticket details |
| `GET` | `/api/bookings` | View passenger booking history |
| `POST` | `/api/tickets/:ticketNumber/verify` | Verify ticket or QR code |

### Important API Design Considerations

- **Authentication:** Protect passenger-specific endpoints using JWT or session-based authentication.
- **Seat Availability:** Revalidate seat availability during booking to prevent double booking.
- **Idempotency:** Use idempotency keys for booking and payment initiation to handle duplicate requests safely.
- **Payment Verification:** Verify payment status on the backend using the payment gateway's supported verification mechanism. Never trust the frontend success redirect alone.
- **Webhook Security:** Validate webhook authenticity and process duplicate events idempotently.
- **ACID Transactions:** Use PostgreSQL transactions, appropriate row locking, and database constraints to maintain booking and seat consistency.
- **Background Workers:** Use background jobs to expire unpaid seat holds, process refunds, and reconcile uncertain payment statuses. These workers do not need public REST endpoints.


## 8. Complete Booking Flow

``` mermaid
flowchart TD
    A[User selects four seats] --> B[POST /api/bookings]
    B --> C{All four seats available?}
    C -->|No| D[Rollback and return 409 Conflict]
    C -->|Yes| E[Begin transaction and lock seats]
    E --> F[Create one booking and hold four seats]
    F --> G[Set expiry to database time + 10 minutes]
    G --> H[Commit transaction]
    H --> I[Initiate payment]
    I --> J{Verified payment outcome?}
    J -->|Success and reservation valid| K[Confirm booking]
    K --> L[Mark four seats BOOKED]
    L --> M[Issue ticket]
    J -->|Unpaid at deadline| N[Expiry worker verifies payment]
    N --> O[Expire booking and release seats atomically]
    J -->|Uncertain or late event| P[Reconcile payment and seat ownership]
```

The four seats must be treated as one logical reservation. The availability
check in the diagram is only a preliminary check; final validation must be
done only after taking the row lock within the transaction. If the
requirement is that four seats be booked together, then the booking cannot
be confirmed by reserving two or three seats and leaving out the rest.

## 9. Double Booking Prevention

Suppose User A and User B want to book seat `A1` at the same time.

Naive implementation:

``` js
const seat = await findSeat(seatId);

if (seat.status === "AVAILABLE") {
  await updateSeat(seatId, "HELD");
}
```

This is unsafe: both requests may read `AVAILABLE` before the update happens.

### Correct Approach

1.  Start a database transaction.
2.  Lock the requested seat rows in a consistent order.
3.  Ensure that all four seats are of the same trip and available.
4.  Create one booking record.
5.  Make the four seats `HELD` and give them the same expiry deadline.
6.  Create four `booking_seats` records.
7.  If all succeed, commit; if any validation fails, roll back.

Illustrative SQL:

``` sql
BEGIN;

SELECT id, status
FROM trip_seats
WHERE trip_id = $1
  AND seat_number IN ('A1', 'A2', 'A3', 'A4')
ORDER BY id
FOR UPDATE;

-- Application logic:
-- 1. Verify exactly four requested seats were found.
-- 2. Verify every seat is eligible for reservation.
-- 3. Create one booking.
-- 4. Update all four seats conditionally.
-- 5. Insert four booking_seats rows.
-- 6. Roll back if any condition fails.

COMMIT;
```

`SELECT ... FOR UPDATE` alone is not enough. Seat validation, expired hold
reclaim, conditional update, unique constraints and transaction error
handling are also needed. The database transaction cannot be kept open until
payment is completed. First hold the seats and commit the transaction; then
communicate with the gateway.

### Q: How will you prevent Double Booking in a Concurrent System?
A:Usually in code we first check whether the seat is empty (if (seat.status === 'AVAILABLE')) and then update. But in a high-concurrency system (when thousands of users send requests at once) this is extremely risky. Because multiple users can read the seats as AVAILABLE in the same millisecond, and if both try to update, a Race Condition is created and double booking happens.

The correct solution to this problem is to use ACID Transaction and Row-Level Locking at the database level:
"To ensure that the same seat cannot be booked by multiple passengers, I will use ACID Transaction, Row-Level Locking and Deadlock Prevention.
##### 1. Start the Transaction: 
First I will start a transaction with BEGIN, so that all the database operations of the booking work as one atomic unit.

-A Transaction is a single unit or Logical Unit of Work of a database management system (DBMS).

A transaction ensures that—either both steps will be completed successfully (Commit), or no step will apply (Rollback). ACID Properties (the 4 main pillars of a transaction)
#####  2. Row-Level Locking:
 Then I will use SELECT ... FOR UPDATE to lock the rows of the seats that the passenger wants to book. As a result, no other concurrent transaction can lock the same row and change it; it will have to wait.
##### 3. Consistent Locking Order: 
When locking multiple seats, I will lock in the same order according to ORDER BY id. This reduces the risk of deadlock.
##### 4. What Deadlock is and why it happens: 
Deadlock happens when two transactions wait for each other's locked resources and neither can move forward.
For example:
- Transaction A first locked Seat 1, then is waiting for Seat 2.
- Transaction B first locked Seat 2, then is waiting for Seat 1.
- A is waiting for B, and B is waiting for A. As a result, a circular wait is created.
Example of Deadlock
Transaction A
Lock: Seat 1
Wait: Seat 2


Transaction B
Lock: Seat 2
Wait: Seat 1



Circular Wait → Deadlock


##### 5. Deadlock Prevention: 
To reduce this problem, I will lock the seats in the same order in all transactions, for example according to ascending id. As a result, if one transaction first locks Seat 1 and then Seat 2, the other transaction will also follow the same order. This avoids the circular-wait situation above.


Suppose Passenger 1 and Passenger 2 are sending requests from two places.

1. Passengers' requests
Passenger 1  Request  ["DF", "C3"]

Passenger 2  Request  ["C3", "DF"]

The Backend's Lock Ordering will sort by Seat ID

Passenger 1  C3 → DF


Passenger 2  C3 → DF

Both have the same lock order!

##### 6. Deadlock Recovery: 
Consistent locking order alone is not enough; deadlock can happen due to locking other tables or resources. So if the database detects a deadlock and aborts one transaction, I will detect the deadlock error in the application and use a limited number of retries and exponential backoff.
##### 7. Availability Verification: 
After taking the lock, I will verify whether each seat is of the correct trip and whether the seat's status is still AVAILABLE.
##### 8. Seat Hold and Booking:
 If all seats are available, I will change them to HELD status and create a booking record, whose expiry will be 10 minutes later.
##### 9. Commit or Rollback:
If all operations succeed, I will COMMIT. If any seat is unavailable or the operation fails, I will ROLLBACK, so that a partial booking is not created.
##### 10. Database Constraint: 
For additional protection, I will use unique constraints and conditional updates as needed, so that duplicate booking can be prevented even outside the application-level logic.
In this way, using transaction, consistent lock ordering, deadlock recovery and database constraints together, the same seat can be prevented from being allocated to multiple passengers even in concurrent booking."

#### 4 things to remember about Deadlock in the interview:
Suppose at the online and offline ticket counters of Dhaka Railway Station, two passengers (Passenger A and Passenger B) are trying to buy 2 tickets (Seat 1 and Seat 2) of the same train at the same time.
##### 1. Mutual Exclusion. Event: 
-A specific seat of the train (e.g., Seat 1) can be given to only one passenger at a time; it is not possible to give it to two together.

-How the condition works: According to the system's rules, a seat can never be shared (Non-shareable). As a result, if Seat 1 has been selected by 'Passenger A', anyone else must wait to get that seat.
##### 2. Hold and Wait. Event:
-Passenger A has successfully held Seat 1 for booking, and is waiting (Wait) to get Seat 2 to confirm the booking. Passenger B has successfully held Seat 2 for booking, and is waiting (Wait) to get Seat 1 to confirm the booking.

-How the condition works: Nobody is releasing the booking of the seat they hold, and yet they are sitting waiting for the next seat.
##### 3. No Preemption. Event:
-According to the ticketing system's rules, if a passenger has kept a seat in the payment process or cart, the system will not forcibly take the seat from their cart and give it to another passenger.

-How the condition works: Until the passenger cancels the booking themselves or the payment times out, the counter or system cannot take away the seat they booked.
##### 4. Circular Wait. Event:
Passenger A will release Seat 1, but is waiting for when Passenger B will release Seat 2. Passenger B will release Seat 2, but is waiting for when Passenger A will release Seat 1.

🚨 Result (Deadlock) Since all 4 conditions have been met together: neither Passenger A nor Passenger B can buy the ticket.

-The ticketing system is stuck or hung. In real life, to break this deadlock, the station's ticket counter or the online system uses Payment Timeout—that is, after 3/4/10 minutes it automatically cancels one booking (Rollback) so that the other can get the ticket.

### Q: In this design, what is the most important Golden Rule for payment processing?
A:The most important technical point is—the database transaction cannot be kept open until the payment is completed.

Why won't we keep it open?

Communicating with the payment gateway (e.g., bKash or a card company) takes some time (Network Delay). If we hold the database connection or transaction until the payment finishes, then with only a few user requests all the database connections will be exhausted and the whole system may go down.

What is the correct method?

1. First we will very quickly enter the database, make the seats HELD for 10 minutes and COMMIT the transaction (that is, the database will finish its work and become completely free).

2. After the database is free, we will start talking to the payment gateway.

3. If the payment is successful, the booking will be CONFIRMED, and if the payment is not made within 10 minutes, a background worker (Cron Job) will on its own release or free the seats for everyone again.

### Q: What is a Cron Job?
A:A Cron Job (or Cron Worker) is a background process or system automation tool that automatically runs a specific job or task at regular intervals or at a specific interval (for example, every 30 seconds or 1 minute).

In a train booking system, the cron job runs in the background, finds expired pending bookings and releases the stuck seats.

##### What is the problem actually? (The Problem Context)
Suppose a passenger clicks to buy a ticket in an online train booking system.

Seat Lock: The system immediately moves the seat to PENDING or LOCKED status (so that no one else can buy it).

Payment Window: The passenger is given 10 minutes to make the payment.

Abandonment: But what will happen if the passenger does not pay and closes the browser tab, leaves without entering the PIN of the bKash/Rocket app, or the internet gets disconnected?

If that seat in the database remains PENDING forever, it can never be sold to any passenger again. This is called Stale Data or Orphaned Lock.

##### 2. How does a Cron Job solve this problem? (The Mechanism)
A cron job is a scheduled automation service (Scheduled Task) that keeps running in the background on the server.


Schedule Trigger: The cron job is continuously triggered every 1 minute in the background.

Database Scanning: It goes to the database and runs a query:

"Find the bookings whose status is still PENDING but more than 10 minutes have passed."

Status Update & Seat Release: Through a Transaction, for the bookings found:

It changes the booking status from PENDING to EXPIRED or CANCELLED.

It makes the status of the related seats AVAILABLE, so that other passengers at the station or in the app can buy the seat within moments.

## 10. 10-Minute Hold and Expiry Mechanism

### 10.1 When a booking is created

Suppose:

``` text
created_at = 10:00:00
expires_at = 10:10:00
status     = PENDING_PAYMENT
```

The status of the four seats will be `HELD`.

Server-side deadline:

``` text
expiresAt = databaseCurrentTime + 10 minutes
```

The frontend can show a countdown, but the frontend is not the authority of
the timer. The Backend and database will enforce expiry.

### 10.2 Background Expiry Worker

The worker will look for bookings whose deadline has passed at regular
intervals:

``` sql
SELECT id
FROM bookings
WHERE status = 'PENDING_PAYMENT'
  AND expires_at <= NOW();
```

This only finds candidate bookings; this query alone is not sufficient to
expire a booking.

Safe worker process:

1.  If needed, verify the payment status from the gateway outside the
    database transaction. Database locks cannot be held during the gateway
    call.
2.  Start a transaction and lock/claim the booking row.
3.  Verify again that the booking is still `PENDING_PAYMENT` and the
    deadline has passed.
4.  If there is payment success or an unresolved state, do reconciliation
    according to policy; do not blindly release the seats.
5.  Make the unpaid booking `EXPIRED`.
6.  Make the four held seats under the booking `AVAILABLE` and clear the
    current booking ownership.
7.  Commit the transaction.

To ensure that there is no state corruption even if multiple worker
instances process the same booking, row locking/claiming and conditional
state transition must be used. The worker may run late, so the API will also
enforce the expiry rule.

### 10.3 Timeline

  -----------------------------------------------------------------------
  Time                                 Event
  ----------------------------------- -----------------------------------
  10:00 AM                            Four seats became `HELD`

  10:03 AM                            User started payment

  10:06 AM                            Payment was verified

  10:06 AM                            Booking `CONFIRMED`, seats `BOOKED`

  Alternative: 10:10 AM               If payment is not successful, the
                                      booking will expire and the seats
                                      will be released
  -----------------------------------------------------------------------

The payment gateway and database are separate systems. So the race condition
between payment outcome verification and the expiry worker must be handled.

## 11. Payment Failure and Delayed Webhook

### Problem: Late Webhook Arrival & Cron Job Race Condition Situation:

10:09:58 AM: The user successfully completes the payment at the payment gateway.

10:10:00 AM: The backend's Cron Job, as 10 minutes are complete, declares the pending booking EXPIRED and releases the seat (makes it AVAILABLE).

10:10:02 AM: Another new user comes and books that vacated seat.

10:10:05 AM: Due to network latency, the first user's payment Webhook reaches the server, in which the status says SUCCESS.

Where is the trouble?
Although the payment was successful, the first user's seat has already been given to someone else. Now if the backend blindly wants to confirm the payment without the seat or updates the state incorrectly, the system's data will be corrupted and Double Booking (the same seat given to two people) will occur

#### How the solution works (Step-by-Step Explanation)
##### 1. Verifying instead of blindly trusting (Verification & Authoritative Check)
What happens: The system does not directly trust the web hook (Webhook) message that comes from the payment gateway.

Why and how: Hackers can sometimes send fake messages. So the backend first checks whether the message's Signature is correct. Then it makes an API call to the payment gateway's own server and gets real-time confirmation: "Brother, has this user's money really been deposited in the bank?" Only when the gateway says yes does the system proceed to the next step.

##### 2. Preventing collisions in the database (Database Locking)
What happens: On one side the cron job is doing seat expiry work, on the other side the payment success web hook is coming to update data—both processes may attack the database at the same time.

Why and how: To prevent this, Row-Level Lock (SELECT ... FOR UPDATE) is used in the database. This means—while the cron job is working on a seat, the web hook is asked to stand outside and wait a bit (or the reverse). Until one finishes, the other cannot touch that seat.

##### 3. What if the seat is not vacant? (Seat Reconciliation)
What happens: The user's payment is successful, but the seat allocated for them has already been given to someone else by the cron job.

Why and how: Here the system can very intelligently do two things:

Alternative Seat: If another seat of that class is vacant on the train or bus, the system, without any hassle, assigns a new seat to that user and confirms the booking.

Refund Path: If the train has become completely full and not a single seat is vacant, then the booking has to be cancelled and the process of sending the user's money back has to start.

##### 4. Writing in the database does not return the money! (Real Gateway Refund API)
What happens: Many junior developers mistakenly change the status in the database to REFUNDED and think the user's money has gone to bKash or the bank. But in reality the money is not refunded.

Why and how: For real money to reach the user, a request must be sent from the backend to the payment gateway's Refund API (with gateway_transaction_id). Only when the gateway server successfully debits the money, sends it to the user's account and gives a confirmation response should the status be permanently updated to REFUND_SUCCESS in the database.

##### 5. Preventing a mess when the same message comes repeatedly (Idempotency)
What happens: Due to network problems, the payment gateway may send the same web hook repeatedly (Retries).

Why and how: The system checks the unique ID (event_id) of each web hook. If it sees that work has already been done once with this ID, it does not do things like deducting money or refunding a second time and just sends a simple 200 OK response.

## 12. Database Transaction

Illustrative transaction to reserve four seats:

``` text
BEGIN TRANSACTION

1. Lock the requested trip-seat rows.
2. Ensure that four valid seats were found.
3. Ensure all seats are eligible to be reserved.
4. Calculate the server-side fare.
5. Create one PENDING_PAYMENT booking.
6. Make the four seats HELD.
7. Set hold_expires_at = database time + 10 minutes.
8. Create four booking_seats rows.

COMMIT
```

If any validation or update fails, `ROLLBACK` must be done. After that,
payment will be initiated outside the transaction. When payment is verified,
the booking will be confirmed and the seats made `BOOKED` in a second
transaction---but following reservation validity and payment policy.

## 13. ACID Properties

ACID are the four important properties of a database transaction:

-   **A --- Atomicity**
-   **C --- Consistency**
-   **I --- Isolation**
-   **D --- Durability**

### 13.1 Atomicity --- Everything will happen, or nothing will

If `A4` among the four seats is unavailable, `A1`, `A2`, `A3` cannot be kept
reserved. The entire transaction will roll back.

``` sql
BEGIN;

-- Lock and validate all four seats.
-- Create one booking.
-- Reserve all four seats.
-- Create four booking_seats records.

COMMIT;   -- if all operations succeed
-- ROLLBACK; -- if any operation fails
```

**Interview answer:** "Atomicity ensures that all four seats are
reserved together, or none of them are reserved."

### 13.2 Consistency --- Database rules will remain correct

Before and after every transaction, database constraints and business
invariants must be maintained.

Main rules:

-   The same seat of the same trip will not be in more than one active
    booking at the same time.
-   If four seats are mandatory in this use case, the booking must have
    exactly four seats.
-   Before making a booking `CONFIRMED`, there must be verified payment and
    valid seat allocation.
-   The seats of an expired booking will be released correctly.
-   Total fare will be according to the server-side calculation.

**Interview answer:** "Consistency ensures that every successful
transaction preserves database constraints and booking business rules."

### 13.3 Isolation --- Keeping concurrent requests safe

If User A and User B want the same seat, row-level locking or atomic
conditional update will ensure that only one can successfully claim a valid
reservation.

PostgreSQL's `SELECT ... FOR UPDATE` can lock the requested rows. When the
transaction ends, the lock is released. The other request will later verify
the new state.

**Interview answer:** "Isolation prevents concurrent users from
successfully reserving the same seat. I would use row-level locking or
atomic conditional updates inside a transaction."

### 13.4 Durability --- Committed data will remain permanent

If a transaction is successfully committed, even if the application crashes
or restarts, the committed booking and seat status are expected to survive
even after database recovery. PostgreSQL's transaction logging and durable
storage configuration help in this guarantee.

However, the database commit and the external payment gateway's state are
not the same ACID transaction. So webhook retry, idempotency and
reconciliation are needed.

**Interview answer:** "Durability ensures that confirmed bookings and
their committed database changes survive application crashes."

### Q: What are ACID properties and how do they work in a train booking system?
A:ACID properties ensure that database transactions in a high-concurrency train booking system are completed very reliably. Below is an explanation of their application in our 4-seat booking flow:

#### Atomicity (Everything will happen, or nothing will):

Explanation: When a user tries to book 4 seats (A1, A2, A3, A4) together, either all four seats will be reserved together, or not even one seat will be reserved. If the 4th seat (A4) has already been taken by someone else, the hold of the remaining three seats will also be cancelled and the entire transaction will ROLLBACK.

Interview Answer: "Atomicity ensures that all four seats are reserved together, or none of them are reserved."

#### Consistency (Keeping database rules correct):

Explanation: Before and after every transaction, all constraints and business rules of the database must be maintained. For example—the same seat of the same trip cannot be in more than one active booking, the booking must have exactly four seats, the total fare must be calculated from the server side and the seats of an expired booking must be released correctly.

Interview Answer: "Consistency ensures that every successful transaction preserves database constraints and booking business rules."

#### Isolation (Keeping concurrent requests safe):

Explanation: If User A and User B want to book the same seat at the same time, isolation will ensure that only one can succeed. For this we use row-level locking (SELECT ... FOR UPDATE) inside the transaction.

Interview Answer: "Isolation prevents concurrent users from successfully reserving the same seat. I would use row-level locking or atomic conditional updates inside a transaction."

#### Durability (Committed data remains permanent):

Explanation: After a transaction is successfully COMMITTED, even if the application crashes or restarts, the committed booking and seat status remain safe in the database. However, it must be remembered that the database commit and the state of the third-party payment gateway are not the same, so payment reconciliation and webhook retry must be handled.

Interview Answer: "Durability ensures that confirmed bookings and their committed database changes survive application crashes."

### 13.5 Relationship between ACID and Expiry

  -----------------------------------------------------------------------
  ACID property                       Role in the expiry mechanism
  ----------------------------------- -----------------------------------
  Atomicity                           Booking expiry and release of the
                                      four seats will happen together.

  Consistency                         No seat will remain wrongly booked
                                      or unavailable.

  Isolation                           Conflicting updates of the payment
                                      webhook and expiry worker will be
                                      controlled.

  Durability                          After the confirmation or expiry
                                      commit, the state will remain
                                      permanent.
  -----------------------------------------------------------------------

### 13.6 Expiry vs Payment Race Condition

At exactly `10:10 AM`, the expiry worker may want to expire the booking, and
at the same time the payment webhook may report success.

Both processes will lock the booking row within a transaction and verify
the state. After getting the lock, each process will re-check the current
state and make only a valid transition.

The policy must be defined beforehand:

-   How will a successful payment before the deadline be confirmed?
-   If a success notification comes after the deadline, how will the actual
    payment time be verified?
-   If the seat has already been given to another user, what will the
    refund/reconciliation be?

Just having a lock does not determine the business policy. The state
transition must be enforced by verifying the gateway's timestamp/event
semantics.

> **Important:** ACID ensures the correctness of a database transaction; it
> does not turn the database and the external payment gateway into a single
> ACID transaction.

## 14. Security and Reliability

-   **Authentication:** Use secure session or JWT-based authentication.
-   **Authorization:** A user can view or cancel only their own booking.
-   **Input validation:** Validate trip ID, seat ID, seat count and payment
    request.
-   **Server-side pricing:** Do not trust the total amount sent by the
    client.
-   **Payment security:** Verify the gateway signature; the frontend
    success page cannot be taken as payment proof.
-   **Idempotency:** Retry or duplicate webhooks must not cause duplicate
    charge/confirmation.
-   **Database integrity:** Use foreign keys, unique constraints,
    transactions and indexes.
-   **Observability:** Log/monitor booking state transitions, payment
    events, expiry failures and reconciliation errors.
-   **Sensitive data:** Do not expose payment secrets or sensitive payment
    information in logs.

## 15. Scalability and Performance

If thousands of users want to book seats at the same time:

1.  **Stateless backend:** Run multiple API instances behind a load
    balancer.
2.  **Database indexing:** Keep query-friendly indexes on `trip_id`,
    `status`, `expires_at` and booking ownership.
3.  **Safe concurrency:** Use atomic seat allocation, database
    transactions and constraints.
4.  **Background processing:** Run expiry, notification and refund
    reconciliation in workers.
5.  **Caching:** Read-heavy data like train schedules can be cached;
    availability must be validated in the database before reservation.
6.  **Monitoring:** Track booking conflicts, payment failures, expired
    holds, worker delay and database load.

For the first version, modular monolith + PostgreSQL + background worker is
sufficient. If the need is proven, microservices can be considered.

## 18. Short Answer to Say in the Interview

> I will design the Train Ticket Booking System with Node.js, Express.js
> and PostgreSQL.
>
> The main database tables will be `Users`, `Trains`, `Trips`, `TripSeats`,
> `Bookings`, `BookingSeats`, `Payments` and `Tickets`. To link seats with a
> booking, I will use the `BookingSeats` relationship.
>
> First the user will search for trains from Dhaka to Jashore and select four
> seats. The backend, within a database transaction, will validate all seats
> and atomically hold them and create a `PENDING_PAYMENT` booking and keep a
> 10-minute `expires_at` timestamp.
>
> Then the backend will initiate payment at the payment gateway. If the
> payment is verified and remains valid according to the reservation policy,
> the booking will become `CONFIRMED`, the four seats will become `BOOKED`
> and the ticket will be issued.
>
> If the payment is not made within the deadline, a background worker will
> verify the payment status, make the booking `EXPIRED` and release the four
> seats through a transaction.
>
> The most important challenges are double booking, concurrent requests and
> delayed payment webhooks. To handle these, I will use PostgreSQL
> row-level locking or atomic conditional update, transactions, database
> constraints, idempotent webhook processing and an expiry worker.
>
> Alongside, I will implement authentication, authorization, input
> validation, logging, monitoring and concurrency/payment failure tests. I
> will keep a modular monolith at the start; as traffic and business need
> grow, I will scale the architecture.
