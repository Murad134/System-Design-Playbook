# Train Ticket Booking System --- System Design

> **উদ্দেশ্য:** Software Engineering Interview Preparation\
> **প্রধান প্রযুক্তি:** Node.js, Express.js, PostgreSQL\
> **মূল সমস্যা:** একজন user Dhaka থেকে Jashore যাওয়ার জন্য একটি booking-এ
> চারটি seat নির্বাচন করবেন। Payment সম্পন্ন করার জন্য seat-গুলো ১০ মিনিট
> hold থাকবে।

------------------------------------------------------------------------

## সূচিপত্র

1.  [Problem Statement](#1-problem-statement)
2.  [Requirements](#2-requirements)
3.  [High-Level Architecture](#3-high-level-architecture)
3.3 [Important SQL for Interview](#train-ticket-booking-system--important-sql-for-interview)
    - [Core Booking Queries](#331-core-booking-queries)
    - [Search, Cancellation এবং Reporting Queries](#332-search-cancellation-এবং-reporting-queries)
    - [Useful Integrity Constraints এবং Indexes](#333-useful-integrity-constraints-এবং-indexes)
4.  [Database Design](#4-database-design)
5.  [Status Management](#5-status-management)
6.  [ER Diagram](#6-er-diagram)
7.  [REST API Design](#7-rest-api-design)
8.  [Complete Booking Flow](#8-complete-booking-flow)
9.  [Double Booking Prevention](#9-double-booking-prevention)
10. [১০ মিনিটের Hold ও Expiry Mechanism](#10-১০-মিনিটের-hold-ও-expiry-mechanism)
11. [Payment Failure ও Delayed Webhook](#11-payment-failure-ও-delayed-webhook)
12. [Database Transaction](#12-database-transaction)
13. [ACID Properties](#13-acid-properties)
14. [Security ও Reliability](#14-security-ও-reliability)
15. [Scalability ও Performance](#15-scalability-ও-performance)
16. [Interview-এ বলার মতো সংক্ষিপ্ত উত্তর](#18-interview-এ-বলার-মতো-সংক্ষিপ্ত-উত্তর)

------------------------------------------------------------------------

## 1. Problem Statement

ধরা যাক, একজন user Dhaka থেকে Jashore যাওয়ার জন্য একটি train খুঁজে পেলেন
এবং `A1`, `A2`, `A3`, `A4`---এই চারটি seat একসঙ্গে book করতে চান।

System-এর কাজ হবে:

1.  Train, route, travel date এবং available seat দেখানো।
2.  চারটি seat এক booking-এর অধীনে reserve করা।
3.  Payment করার জন্য seat-গুলো সর্বোচ্চ ১০ মিনিট hold রাখা।
4.  নির্ধারিত সময়ের মধ্যে payment সফল ও যাচাই করা হলে booking confirm করা।
5.  সময় শেষ হওয়ার পরও payment সফল না হলে booking expire করে চারটি seat-ই
    release করা।
6.  একই seat যেন একসঙ্গে দুইজন user-এর confirmed booking-এ না যায়, তা
    নিশ্চিত করা।
7.  Payment failure, duplicate webhook এবং delayed payment confirmation
    নিরাপদে handle করা।

**মূল design challenge:** এটি শুধু CRUD application নয়। এটি একটি concurrent
booking ও payment system, যেখানে seat availability এবং booking status সব
সময় consistent রাখতে হবে।

## 2. Requirements

### 2.1 Functional Requirements

-   Departure station, destination ও travel date দিয়ে train search করা।
-   Train schedule, fare ও seat availability দেখা।
-   এই use case-এ এক booking-এ ঠিক চারটি seat নির্বাচন করা।
-   সব selected seat atomically hold করা।
-   Payment gateway-এর মাধ্যমে payment initiate ও verify করা।
-   Payment সফল হলে booking confirm করা এবং ticket issue করা।
-   Unpaid booking-এর expiry handle করে seat release করা।
-   User-কে booking details, booking status ও payment status দেখানো।
-   Cancellation ও refund workflow সমর্থন করা।

### 2.2 Non-Functional Requirements

  -----------------------------------------------------------------------
  Requirement                         ব্যাখ্যা
  ----------------------------------- -----------------------------------
  Consistency                         একই trip-এর একই seat একাধিক active
                                      booking-এ বরাদ্দ হবে না।

  Reliability                         Payment ও booking state-এর অসামঞ্জস্য
                                      শনাক্ত ও resolve করতে হবে।

  Concurrency                         একই সময়ে বহু user একই seat চাইলে
                                      নিরাপদভাবে request process করতে
                                      হবে।

  Security                            User কেবল নিজের booking দেখতে বা
                                      cancel করতে পারবেন।

  Scalability                         Traffic বাড়লে API instance ও worker
                                      বাড়ানো যাবে।

  Observability                       Booking conflict, payment failure,
                                      expiry delay ও database load
                                      monitor করতে হবে।
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

**Interview explanation:** Frontend শুধু screen এবং user interaction handle
করবে। Fare, seat availability, payment success বা booking confirmation-এর
final decision frontend নেবে না; এগুলো backend থেকে আসবে।


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

যাত্রী প্রথমে ওয়েব অ্যাপে প্রবেশ করে যাত্রার তারিখ, Departure Station, Destination Station এবং পছন্দের ট্রেন নির্বাচন করে। এরপর অ্যাপ উপলব্ধ সিট দেখায় এবং যাত্রী একটি বা একাধিক সিট নির্বাচন করে বুকিংয়ের অনুরোধ করে।

- **Frontend:** React.js / Next.js
- **কাজ:** Train search, seat selection, passenger information এবং booking request তৈরি করা।
- **গুরুত্বপূর্ণ বিষয়:** Frontend-এ সিট Available দেখালেই সেটি নিশ্চিতভাবে Available নয়। অন্য যাত্রী একই সময়ে সিট বুক করার চেষ্টা করতে পারে। তাই চূড়ান্ত availability যাচাই Backend ও Database-এ করতে হবে।

### 2. React App → HTTPS / TLS 1.3 → Express REST API Gateway

Frontend HTTPS-এর মাধ্যমে API Gateway-তে বুকিংয়ের অনুরোধ পাঠায়। HTTPS এবং TLS 1.3 ক্লায়েন্ট ও সার্ভারের মধ্যে যোগাযোগ এনক্রিপ্ট করতে সাহায্য করে।

**API endpoint:**

`POST /api/bookings`

Request body-তে থাকতে পারে:

- `trainId`
- `journeyDate`
- `seatIds`
- `passengerDetails`

Retry বা double-click থেকে duplicate booking ঠেকাতে request header-এ একটি unique `Idempotency-Key` পাঠানো হবে। একই key এবং একই request পুনরায় এলে Server আগের response ফেরত দিতে পারবে, যাতে দ্বিতীয়বার নতুন booking তৈরি না হয়।

API Gateway-এর দায়িত্ব:

- Request routing
- Rate limiting
- Authentication middleware চালানো
- Request-এর উপযুক্ত Backend Service-এ পাঠানো

একই Idempotency-Key দিয়ে ভিন্ন request পাঠানো হলে সেটিকে নতুন booking হিসেবে গ্রহণ না করে conflict বা validation error দিতে হবে।

### 3. API Gateway → Authentication & Request Validation

API Gateway বা সংশ্লিষ্ট Middleware যাত্রীর পরিচয় এবং request-এর বৈধতা যাচাই করে।

**Authentication:** JWT বা Session যাচাই করে যাত্রীর পরিচয় নিশ্চিত করা।

**Validation:** Train ID, journey date, seat IDs এবং passenger information-এর format ও বৈধতা যাচাই করা।

সম্ভাব্য response:

- `401 Unauthorized` — Authentication অনুপস্থিত বা অবৈধ।
- `400 Bad Request` — Request format বা input অবৈধ।
- `429 Too Many Requests` — Rate limit অতিক্রম করেছে।

এখানে কেবল request গ্রহণযোগ্য কি না যাচাই করা হয়। সিটের চূড়ান্ত availability ও reservation নিশ্চিত করা Booking Service-এর দায়িত্ব।

### 4. Authentication → Booking Service → PostgreSQL (Seat Hold)

Authentication ও request validation সফল হলে request Booking Service-এ পাঠানো হয়। Booking Service বুকিংয়ের Business Logic পরিচালনা করে এবং PostgreSQL transaction-এর মাধ্যমে নির্বাচিত সিটগুলো সাময়িকভাবে reserve করে।

**Seat Hold workflow:**

1. Database transaction শুরু হবে: `BEGIN`.
2. সংশ্লিষ্ট trip, journey date এবং seat IDs যাচাই করা হবে।
3. প্রয়োজন অনুযায়ী `SELECT ... FOR UPDATE` অথবা conditional update দিয়ে seat rows lock বা atomically claim করা হবে।
4. সব নির্বাচিত সিট Available কি না যাচাই করা হবে।
5. সব সিট পাওয়া গেলে booking তৈরি হবে এবং seat status `HELD` হবে।
6. Booking status হবে `PENDING_PAYMENT`।
7. Database time অনুযায়ী `expires_at` নির্ধারণ করা হবে, যেমন বর্তমান সময়ের ১০ মিনিট পরে।
8. সব পরিবর্তন সফল হলে `COMMIT` হবে। কোনো একটি সিটও পাওয়া না গেলে সম্পূর্ণ transaction `ROLLBACK` হবে এবং `409 Conflict` ফেরত দেওয়া হবে।

**উদাহরণ:**

- `bookingId`: `BK-1001`
- `bookingStatus`: `PENDING_PAYMENT`
- `seatIds`: `A1, A2, A3, A4`
- `seatStatus`: `HELD`
- `expiresAt`: Hold তৈরির database time থেকে ১০ মিনিট পরে

চারটি সিটের মধ্যে একটি সিটও অন্য যাত্রীর কাছে বরাদ্দ হয়ে থাকলে চারটি সিটের কোনোটি partial hold হিসেবে রাখা যাবে না।

Database-এ উপযুক্ত constraints রাখতে হবে, যেমন `UNIQUE (trip_id, seat_number)` দিয়ে একটি trip-এর প্রতিটি physical seat-এর পরিচয় unique রাখা। Active seat allocation-এর জন্য প্রয়োজন অনুযায়ী additional constraints-ও রাখতে হবে।

**১০ মিনিটের Hold কেন গুরুত্বপূর্ণ?**

এটি যাত্রীকে payment সম্পন্ন করার জন্য সীমিত সময় দেয়। Hold তৈরি করা এবং hold expiry-এর পর সিট release করা আলাদা database state transition; তাই শুধু Frontend timer দিয়ে এই নিয়ম enforce করা যাবে না।

#### Expiry Worker

Cron Job বা Queue Worker নিয়মিত expired `PENDING_PAYMENT` booking খুঁজবে।

প্রতিটি booking-এর জন্য:

1. Database transaction শুরু করবে।
2. Booking ও সংশ্লিষ্ট seat rows lock করবে অথবা নিরাপদ conditional update করবে।
3. Database time অনুযায়ী hold সত্যিই expired হয়েছে কি না যাচাই করবে।
4. Booking এখনো `PENDING_PAYMENT` থাকলে `EXPIRED` করবে।
5. সেই booking-এর মালিকানাধীন `HELD` সিটগুলো `AVAILABLE` করবে।
6. সব পরিবর্তন একসঙ্গে commit করবে।

একই booking একাধিক worker process করলেও state corruption বা অন্য booking-এর সিট release হওয়া চলবে না। তাই expiry worker এবং payment confirmation-এর ক্ষেত্রে সামঞ্জস্যপূর্ণ locking ও conditional state-transition rules ব্যবহার করতে হবে।

## Phase 2: Payment Processing

### 5. Booking Service → Payment Service

Seat Hold সফল হলে Payment Service-এ payment শুরু করার request পাঠানো হবে।

Payment Service booking ID, নির্ধারিত ভাড়া এবং payment request যাচাই করে Payment Record তৈরি করবে।

**উদাহরণ:**

- `paymentId`: `PAY-5001`
- `bookingId`: `BK-1001`
- `amount`: ৳800
- `paymentStatus`: `INITIATED`

Payment Service-এর দায়িত্ব হলো payment lifecycle পরিচালনা করা; Booking Service-এর দায়িত্ব হলো booking এবং seat reservation পরিচালনা করা।

দুটি Service-এর আলাদা database থাকলে তাদের মধ্যে সরাসরি একটি common ACID transaction থাকবে না। তাই Payment Service-এও idempotency, retry এবং payment status tracking প্রয়োজন।

### 6. Payment Service → External Payment Gateway

Payment Service bKash, SSLCommerz, Nagad বা aamarPay-এর মতো অনুমোদিত Payment Gateway-এর মাধ্যমে payment initiate করবে।

Workflow:

1. Backend Payment Gateway-তে payment session তৈরি করবে।
2. Gateway থেকে Checkout URL বা payment session পাওয়া যাবে।
3. Frontend যাত্রীকে Checkout page-এ পাঠাবে।
4. যাত্রী payment সম্পন্ন করবেন।
5. Gateway payment status অনুযায়ী callback বা webhook পাঠাবে।

**গুরুত্বপূর্ণ:** শুধু Frontend-এর Success Page দেখে payment successful ধরে নেওয়া যাবে না। Gateway-এর documented server-side verification API দিয়ে payment status, amount, currency এবং transaction reference যাচাই করতে হবে।

### 7. External Payment Gateway → Idempotent Webhook Handler

Webhook কীভাবে কাজ করে?

1 Payment Gateway → Backend

তুমি টাকা দেওয়ার পর bKash তোমার Backend-এর POST /api/webhooks/payment endpoint-এ একটি message পাঠায়: “এই payment সফল হয়েছে।”

2Message আসল কি না যাচাই

Backend পরীক্ষা করে message-টি সত্যিই Gateway থেকে এসেছে কি না এবং এতে কোনো পরিবর্তন করা হয়েছে কি না। Gateway-এর নিয়ম অনুযায়ী signature verification বা অন্য authentication পদ্ধতি ব্যবহার করা হয়।

3 Event Database-এ Save

Backend message-টি Database-এ সংরক্ষণ করে। একই message আগে এসেছে কি না, সেটিও পরীক্ষা করে। Event ID-তে UNIQUE constraint থাকায় একই event দ্বিতীয়বার নতুন event হিসেবে save করা যায় না।

4 Gateway-কে Acknowledgement

Message নিরাপদে save হলে Backend 200 OK পাঠায়। অর্থাৎ, “তোমার message পেয়েছি।” এর মানে এই নয় যে payment ইতিমধ্যে যাচাই হয়ে গেছে।

5 Worker Payment Verify করে

Background Worker Gateway-এর server-side API-তে যাচাই করে payment সত্যিই সফল হয়েছে কি না, টাকার পরিমাণ ৳৫০০ কি না এবং transaction reference সঠিক কি না। সব ঠিক থাকলে Payment Service payment-টিকে SUCCEEDED করে।

একই message তিনবার এলে কী হবে?
ধরো, Gateway একই payment success message ভুলবশত তিনবার পাঠাল।
Event	Backend কী করবে?
প্রথমবার	Event save করে process করবে
দ্বিতীয়বার	একই Event ID শনাক্ত করে duplicate হিসেবে ধরবে
তৃতীয়বার	আবার payment process করবে না
এতে একই payment একাধিকবার রেকর্ড হওয়া বা একই booking-এর জন্য বারবার ticket তৈরির ঝুঁকি কমে।

দুটি বিষয় মনে রাখবে:
- Webhook Idempotency: একই message বারবার এলেও সেটি নতুন করে process না করা।
- Payment Idempotency: একই gateway transaction যেন একাধিকবার সফল payment হিসেবে রেকর্ড না হয়।

শুধু Event ID unique করলেই যথেষ্ট নয়। একই transaction ভিন্ন event ID-তে এলেও যেন দ্বিতীয়বার payment record না হয়, তাই gateway transaction reference-এও uniqueness নিশ্চিত করতে হয়। Ticket তৈরির ক্ষেত্রেও booking ID বা ticket key দিয়ে duplicate creation আটকাতে হয়

### 8. Webhook Handler → Payment Service (Payment Verification)

Asynchronous worker থেকে event পাওয়ার পর Payment Service Gateway-এর authoritative payment record যাচাই করবে।

যাচাইয়ের বিষয়গুলো:

- Payment reference সঠিক কি না।
- Payment সংশ্লিষ্ট booking-এর সঙ্গে যুক্ত কি না।
- Paid amount প্রত্যাশিত ভাড়ার সমান কি না।
- Currency সঠিক কি না।
- Gateway-এর transaction status সফল কি না।
- একই gateway transaction আগেই অন্য payment record-এ ব্যবহার হয়েছে কি না।

সব যাচাই সফল হলে Payment Record-এর অবস্থা `SUCCEEDED` করা হবে। Payment ব্যর্থ হলে উপযুক্ত failure status রাখা হবে।

Payment Service তারপর verified payment result Booking Service-এ পাঠাবে। এটি internal API বা durable event delivery-এর মাধ্যমে করা যেতে পারে।

Payment event duplicate বা out-of-order হলেও status transition যেন ভুলভাবে পরিবর্তিত না হয়, তা নিশ্চিত করতে হবে।

## Phase 3: Booking Confirmation Using ACID

### 9. Payment Service → Booking Service (Confirm Request)

Payment Service verified payment success-এর তথ্য Booking Service-এ পাঠায়।

Booking Service confirmation-এর আগে যাচাই করবে:

- Booking ID সঠিক কি না।
- Payment সত্যিই verified কি না।
- Payment amount এবং booking fare মিলছে কি না।
- Booking এখনো `PENDING_PAYMENT` কি না।
- Seat hold এখনো বৈধ কি না।
- Hold expiry হয়েছে কি না।
- নির্বাচিত সিটগুলো এখনো সংশ্লিষ্ট booking-এর মালিকানায় আছে কি না।
- Booking ইতিমধ্যে `CONFIRMED` হয়েছে কি না।

**Late Payment Scenario:**

ধরা যাক, একজন যাত্রী ১০ মিনিটের মধ্যে payment করেননি। Expiry Worker booking-কে `EXPIRED` করেছে এবং সিটগুলো অন্য যাত্রীর জন্য Available হয়েছে। এরপর প্রথম যাত্রীর payment-success event দেরিতে এল।

এই অবস্থায় Booking Service সরাসরি booking confirm করবে না।

- সিট অন্য booking-এ বরাদ্দ থাকলে confirmation প্রত্যাখ্যান করতে হবে।
- Payment ইতিমধ্যে সফল হলে `REFUND_PENDING` বা উপযুক্ত compensation workflow শুরু করতে হবে।
- Refund status-ও আলাদাভাবে track করতে হবে।
- Payment event বা internal request পুনরায় এলে duplicate refund বা duplicate confirmation যেন না হয়, তা নিশ্চিত করতে হবে।

সিট এখনো অন্য কাউকে বরাদ্দ করা না হয়ে থাকলেও expired booking স্বয়ংক্রিয়ভাবে confirm করা হবে কি না, তা স্পষ্ট business policy-এর ওপর নির্ভর করবে। সবচেয়ে সহজ ও নিরাপদ policy হলো expired hold পুনরায় ব্যবহার না করে নতুন reservation flow শুরু করা।

Payment Service ও Booking Service আলাদা database ব্যবহার করলে payment success এবং booking confirmation একটি সাধারণ ACID transaction-এ হবে না। এই distributed workflow নির্ভরযোগ্য করতে durable event delivery, retry, idempotency এবং reconciliation job প্রয়োজন।

#### Mutex (পূর্ণরূপ: Mutual Exclusion):
হলো কনকারেন্সি প্রোগ্রামিং (Concurrent Programming)-এর একটি অত্যন্ত গুরুত্বপূর্ণ কনসেপ্ট বা মেকানিজম।

সহজ কথায়, এটি হলো একটি ডিজিটাল লক (Digital Lock)।

যখন একাধিক থ্রেড (Threads) বা প্রসেস (Processes) একই সময়ে একই শেয়ারড রিসোর্স (Shared Resource)—যেমন: কোনো মেমোরি লোকেশন, ভ্যারিয়েবল, ফাইল বা ডাটাবেজ টেবিল—অ্যাক্সেস বা মডিফাই করার চেষ্টা করে, তখন ডেটা কনফ্লিক্ট বা রেস কন্ডিশন (Race Condition) এড়ানোর জন্য Mutex ব্যবহার করা হয়।

ট্রেনের টিকিট বুকিং সিস্টেমে Mutex ঠিক কীভাবে কাজ করে এবং কেন এটি এতটা জরুরি, তা একটি বাস্তব উদাহরণ দিয়ে নিচে বুঝিয়ে দেওয়া হলো:

১. সমস্যাটি কোথায় হয়? (Race Condition)ধরা যাক, ঢাকা থেকে জেসোরগামী একটি ট্রেনের A1 নম্বর সিটটি শুধু একটিই খালি আছে। ঠিক একই সেকেন্ডে (একই মিলিিসেকেন্ডে) দু'জন যাত্রী—রহিম এবং করিম—ওই সিটটি বুক করার জন্য রিকোয়েস্ট পাঠাল।

যদি সিস্টেমে কোনো লক বা Mutex না থাকে, তবে ব্যাকএন্ড সার্ভার একই সময়ে দু'টি কাজ করবে:রহিম দেখবে সিট খালি আছে  সিট বুক করে ফেলবে।করিমও দেখবে সিট খালি আছে  সেও সিট বুক করে ফেলবে।ফলাফল? একই সিট দু'জন যাত্রীর নামে কনফার্ম হয়ে যাবে (Double Booking), যা কোনো রেলওয়ে সিস্টেমেই গ্রহণযোগ্য নয়।

২. এখানে Mutex যেভাবে কাজ করেএই সমস্যা সমাধানের জন্য সার্ভারে যখন রিকোয়েস্টগুলো আসে, তখন একটি Mutex (লক) ব্যবহার করা হয়:

Lock Acquire: রহিম ও করিমের রিকোয়েস্ট আসার পর সিস্টেম প্রথমে Mutex-এ একটি লক নেওয়ার চেষ্টা করে। ধরুন, রহিম-এর রিকোয়েস্ট মিলিিসেকেন্ডের ভগ্নাংশে আগে পৌঁছাল এবং সে Mutex-টি লক করে ফেলল।

Critical Section (বাস্তবায়ন): এখন রহিম সার্ভারে চেক করল যে সিট A1 ফাঁকা আছে কি না। ফাঁকা থাকায় সেটিকে সে নিজের নামে হোল্ড বা বুক করে ফেলল।Wait / Block: ঠিক ওই মুহূর্তেই করিম যখন সিট চেক করতে চাইল, সিস্টেম দেখল Mutex ইতিমধ্যে রহিমের দখলে আছে (লক করা)। তাই করিমের রিকোয়েস্টটিকে ওয়েটিং স্টেটে (Blocked) রাখা হলো।

Unlock & Result: রহিমের বুকিং প্রসেস শেষ হওয়ার পর Mutex রিলিজ (Unlock) হলো। এবার করিমের রিকোয়েস্ট সুযোগ পেল। করিম সিট চেক করে দেখল যে A1 সিটটি ইতিমধ্যে BOOKED হয়ে গেছে। ফলে সিস্টেম করিমকে জানিয়ে দিল যে "দুঃখিত, এই সিটটি আর খালি নেই।

### 10. Booking Service → PostgreSQL Primary Database (Atomic Confirmation)

এটি পুরো সিস্টেমের সবচেয়ে গুরুত্বপূর্ণ ধাপগুলোর একটি। Booking Service PostgreSQL transaction-এর মাধ্যমে booking এবং seat allocation চূড়ান্ত করবে।

**Confirmation transaction workflow:**

1. `BEGIN` — Database transaction শুরু।
2. সংশ্লিষ্ট Booking ও Seat rows lock অথবা নিরাপদ conditional update করা।
3. Booking status, hold expiry এবং seat ownership যাচাই করা।
4. Verified payment reference ও payment status-এর ভিত্তিতে confirmation-এর eligibility যাচাই করা।
5. Booking status `CONFIRMED` করা।
6. সংশ্লিষ্ট seat status `BOOKED` করা।
7. Ticket Record তৈরি করা।
8. একই database transaction-এ `outbox_events` table-এ ticket-processing বা notification event insert করা।
9. `COMMIT` — সব পরিবর্তন সফল হলে একসঙ্গে স্থায়ীভাবে সংরক্ষণ করা।
10. কোনো database operation ব্যর্থ হলে `ROLLBACK` করা।

**উদাহরণ:**

- `bookingStatus`: `CONFIRMED`
- `seatStatus`: `BOOKED`
- `ticketStatus`: `PENDING_ISSUANCE`

এখানে Ticket Record transaction-এর মধ্যে তৈরি হলেও PDF/QR generation এবং SMS/email পাঠানো transaction-এর মধ্যে করা হবে না। এগুলো পরে asynchronous processing-এর মাধ্যমে হবে।

Transactional Outbox সঠিকভাবে কাজ করার জন্য Ticket Record এবং `outbox_events` একই database transaction-এ থাকতে হবে। Ticket Service-এর আলাদা database থাকলে Booking Service তার নিজের database-এ confirmation ও Outbox event commit করবে; এরপর Ticket Service event consume করে নিজের database-এ ticket তৈরি করবে।

**ACID Properties:**

- **Atomicity:** সংশ্লিষ্ট database transaction-এর পরিবর্তনগুলো সম্পূর্ণ হবে, নতুবা rollback হবে।
- **Consistency:** Database constraints, seat ownership এবং booking state rules বজায় থাকবে।
- **Isolation:** Concurrent booking বা confirmation request অসামঞ্জস্যপূর্ণভাবে একই সিট বরাদ্দ করতে পারবে না।
- **Durability:** Commit সফল হলে booking, seat এবং ticket record database-এ স্থায়ীভাবে সংরক্ষিত থাকবে।

মনে রাখতে হবে, ACID একটি database transaction-এর সীমার মধ্যে প্রযোজ্য। Payment Gateway এবং আলাদা Microservice-সহ পুরো distributed workflow স্বয়ংক্রিয়ভাবে একটি ACID transaction হয়ে যায় না।

## Phase 4: Ticket & Notification

### 11. Booking Confirmation → Ticket Processing / Notification Service

Step 10-এর database transaction-এ Ticket Record তৈরি হয়েছে। এখন Outbox Worker `outbox_events` table থেকে pending event পড়ে পরবর্তী processing শুরু করবে।

Workflow:

1. Outbox Worker pending event সংগ্রহ করবে।
2. Ticket Processing Service প্রয়োজনীয় booking ও passenger data ব্যবহার করে PDF বা QR ticket তৈরি করবে।
3. Ticket generation সফল হলে ticket status `ISSUED` করা হবে।
4. Notification event Notification Service-এ পাঠানো হবে।
5. Processing failure হলে retry করা হবে; প্রয়োজনে dead-letter queue ও monitoring ব্যবহার করা হবে।

একই event একাধিকবার process হলেও একই booking-এর জন্য duplicate ticket তৈরি করা যাবে না। এজন্য Ticket Record-এ booking বা booking-seat অনুযায়ী উপযুক্ত unique constraint এবং idempotent processing রাখতে হবে।

Transactional Outbox database commit এবং event delivery-এর মধ্যকার failure risk কমায়। তবে worker-এর duplicate delivery সম্ভব, তাই downstream consumer-কেও idempotent হতে হবে।

### 12. Notification Service → SMS / Email Gateway

Notification Service বাহ্যিক Email বা SMS Provider ব্যবহার করে যাত্রীকে booking confirmation এবং ticket information পাঠাবে।

- **Email Gateway:** Ticket details, download link বা PDF পাঠাতে পারে।
- **SMS Gateway:** Booking confirmation এবং Ticket Reference পাঠাতে পারে।
- **Retry Mechanism:** সাময়িক provider failure হলে পুনরায় পাঠানোর চেষ্টা করবে।
- **Idempotency:** Retry-এর কারণে duplicate notification যতটা সম্ভব প্রতিরোধ করবে।

Notification ব্যর্থ হলে booking বাতিল করা উচিত নয়। Booking ইতিমধ্যে `CONFIRMED` থাকলে notification retry করা যাবে।

### 13. SMS / Email Gateway → Passenger

যাত্রী SMS বা email-এ booking confirmation এবং ticket information পাবেন। ওয়েব অ্যাপের My Bookings page থেকেও ticket দেখা বা download করা যাবে।

Ticket verification-এর সময় Ticket ID বা QR Code ব্যবহার করে Backend থেকে ticket-এর validity যাচাই করতে হবে। QR Code-এ এমন sensitive passenger information রাখা উচিত নয়, যা publicভাবে প্রকাশিত হলে অপব্যবহার হতে পারে।

এই পর্যায়ে booking, payment এবং ticket-এর record সংরক্ষিত থাকে। ফলে ভবিষ্যতে Ticket Verification, Refund, Cancellation এবং Audit করা সম্ভব হয়।

## Simple End-to-End Story

একজন যাত্রী ট্রেন খুঁজে A1, A2, A3 ও A4—চারটি সিট নির্বাচন করেন। Frontend HTTPS-এর মাধ্যমে API Gateway-তে booking request পাঠায়। Authentication ও validation সফল হলে Booking Service PostgreSQL transaction-এর মধ্যে চারটি সিট atomically hold করে এবং ১০ মিনিটের `PENDING_PAYMENT` booking তৈরি করে।

এরপর Payment Service payment initiate করে এবং যাত্রী Gateway-তে payment সম্পন্ন করেন। Gateway-এর webhook durableভাবে সংরক্ষিত হয়; asynchronous worker Gateway-এর authoritative API দিয়ে payment verify করে।

Payment verified হলে Booking Service আবার database transaction শুরু করে booking expiry, seat ownership ও payment validity যাচাই করে। সব শর্ত পূরণ হলে booking `CONFIRMED`, seat `BOOKED` হয় এবং একই transaction-এ Ticket Record ও `outbox_events` row তৈরি হয়।

Outbox Worker পরে ticket PDF/QR তৈরি ও notification processing চালায়। Notification ব্যর্থ হলেও booking confirmed থাকে এবং retry করা যায়।

অন্যদিকে, ১০ মিনিটের মধ্যে payment সম্পন্ন না হলে Expiry Worker transaction-এর মধ্যে booking-কে `EXPIRED` করে এবং সেই booking-এর `HELD` সিটগুলো `AVAILABLE` করে release করে। পরে payment success event এলেও expired booking সরাসরি confirm না করে policy অনুযায়ী refund বা reconciliation workflow চালানো হয়।

#### Summarized Table for Quick Reference ( Interview Answer )

Junior Software Engineer · Easy–Medium System Design

## Step-by-step workflow

##### step: 1\. Passenger → React / Next.js

Passenger train, journey date ও seats select করে booking request পাঠাবে। Frontend-এ seat Available দেখালেই সেটি নিশ্চিতভাবে available নয়, কারণ concurrent booking হতে পারে।

##### step: 2\. Frontend → HTTPS / TLS 1.3 → API Gateway

Frontend `POST /api/bookings` request পাঠাবে। HTTPS data encrypt করবে। `Idempotency-Key` ব্যবহার করে retry বা double-click থেকে duplicate booking প্রতিরোধ করব।

##### step: 3\. API Gateway → Authentication & Validation

JWT/Session দিয়ে Authentication এবং request validation করব। Rate limiting ও routing-ও এখানে থাকতে পারে।

##### step: 4\. API Gateway → Booking Service → PostgreSQL

Booking Service transaction ও row locking (`SELECT ... FOR UPDATE`) অথবা conditional update দিয়ে seat availability যাচাই করবে। সব requested seat available হলে `HELD` করে `PENDING_PAYMENT` booking তৈরি করবে, যার expiry ১০ মিনিট। Unique constraints double allocation ঠেকাবে। কোনো seat পাওয়া না গেলে পুরো transaction rollback করে `409 Conflict` দেব।

Expiry Worker: Cron/Queue Worker expired booking-কে `EXPIRED` করবে এবং সংশ্লিষ্ট held seats `AVAILABLE` করবে। Row locking ও conditional state transition দিয়ে concurrent expiry processing নিরাপদ রাখব।

##### step: 5\. Booking Service → Payment Service

Payment Service booking-এর amount যাচাই করে `INITIATED` Payment Record তৈরি করবে। Booking Service seat/booking এবং Payment Service payment পরিচালনা করবে।

##### step: 6\. Payment Service → External Payment Gateway

Backend bKash/SSLCommerz-এর মতো Gateway-তে payment initiate করবে এবং Frontend-কে checkout information দেবে। শুধু success page দেখে payment সফল ধরব না।

##### step: 7\. Payment Gateway → Idempotent Webhook Handler

Gateway payment notification পাঠাবে। Signature/authenticity যাচাই করে event database-এ সংরক্ষণ করব। Unique `event_id` ও idempotency ব্যবহার করে duplicate webhook থেকে duplicate processing ঠেকাব।

##### step: 8\. Webhook Handler → Payment Service

Payment Service Gateway-এর নিয়ম অনুযায়ী status, transaction reference, amount ও currency verify করবে। সফল হলে payment `SUCCEEDED` হবে। অনিশ্চিত status হলে reconciliation করব।

##### step: 9\. Payment Service → Booking Service

Verified payment success event Booking Service-এ পাঠাব। Booking এখনও `PENDING_PAYMENT`, seat ownership সঠিক এবং hold বৈধ কি না পুনরায় যাচাই করব। Hold expire হয়ে seat অন্য booking-এ চলে গেলে booking confirm করব না; payment কাটা হয়ে থাকলে `REFUND_PENDING` workflow শুরু করব।

##### step: 10\. Booking Service → PostgreSQL Primary Database

একটি transaction-এ booking `CONFIRMED`, seats `BOOKED`, Ticket Record এবং `outbox_events` তৈরি করব। সব সফল হলে `COMMIT`, অন্যথায় `ROLLBACK`।

এখানে ACID গুরুত্বপূর্ণ:

- Atomicity (সব হবে, না হয় কিছুই হবে না): একটা transaction-এর সব কাজ সফল হবে, নাহলে কোনো কাজই save হবে না।
উদাহরণ: Ticket booking-এর সময় seat book হলো, কিন্তু payment record save হলো না। Transaction rollback হলে seat booking-ও বাতিল হবে।
- Consistency (Database-এর নিয়ম ঠিক থাকবে): প্রতিটি transaction-এর পর database-এর নিয়ম ঠিক থাকতে হবে।
উদাহরণ: ট্রেনে ৪০টি seat থাকলে ৪১ নম্বর booking করা যাবে না।
- Isolation (একসঙ্গে কাজ করলেও সমস্যা হবে না): একাধিক user একই সময়ে booking করলেও database ভুল result হতে দেবে না।
উদাহরণ: একই seat-এর জন্য দুইজন একসঙ্গে request করলেও seat একজনই পাবে।
- Durability (Save হলে থেকে যাবে): Transaction একবার successfully commit হলে data database-এ সংরক্ষিত থাকবে, server restart হলেও।
উদাহরণ: Ticket booking successful হওয়ার পর server বন্ধ হলেও booking-এর data হারিয়ে যাবে না।

##### step: 11\. Ticket / Notification Service → SMS / Email Gateway

Outbox Worker event process করে Ticket PDF/QR তৈরি এবং notification পাঠাবে। Failure হলে retry করবে; notification failure-এর কারণে confirmed booking বাতিল হবে না।

##### step: 12\. SMS / Email Gateway → Passenger

Passenger confirmation ও ticket details পাবেন। `My Bookings` থেকে ticket দেখা যাবে এবং QR/Ticket ID দিয়ে backend-এ validity verify করা যাবে।

**Simple end-to-end story:** User train search করে চারটি seat select করেন।
Frontend API-তে booking request পাঠায়। Backend PostgreSQL transaction-এ
চারটি seat lock করে `HELD` status দেয় এবং ১০ মিনিটের `PENDING_PAYMENT`
booking তৈরি করে। Payment সফল ও verified হলে booking `CONFIRMED` হয় এবং
seat `BOOKED` হয়। Payment না হলে expiry worker booking expire করে seat
release করে।


#### Payment, Booking-এর আগে হচ্ছে না ?
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
##### তাহলে Booking কখন হয়?
দুটি আলাদা বিষয়:
- Booking তৈরি: Payment-এর আগেই হয়। Seat ১০ মিনিটের জন্য hold করা হয়, যাতে অন্য কেউ সেটি book করতে না পারে।
- Booking confirm: Payment সফলভাবে verify হওয়ার পরে হয়। তখন seat স্থায়ীভাবে BOOKED হয়।
কেন এই পদ্ধতি?

ধরো, তুমি ৪টি seat book করতে চাও। Backend আগে seat-গুলো ১০ মিনিটের জন্য hold করবে। তারপর তুমি payment করবে। Payment সফল হলে booking confirm হবে। Payment না করলে hold expire হয়ে seat আবার available হবে।

1. Pending Booking & Seat Hold: Backend সিট খালি আছে কি না যাচাই করে ১০ মিনিটের জন্য সাময়িকভাবে Hold করে এবং PENDING_PAYMENT বুকিং তৈরি করে।

2. Payment Initiation: Payment Service একটি payment record তৈরি করে এবং Passenger-কে payment gateway-এ টাকা দেওয়ার সুযোগ দেয়।

3. Payment Verification: Payment gateway-এর webhook বা status check-এর মাধ্যমে Backend যাচাই করে সত্যিই পেমেন্ট সফল হয়েছে কি না।

4. Booking Confirmation: Backend পুনরায় যাচাই করে সিটের Hold এখনো কার্যকর কি না। সব ঠিক থাকলে Database Transaction-এর মাধ্যমে Booking CONFIRMED এবং Seat BOOKED করে।

5. Ticket & Notification: Booking নিশ্চিত হওয়ার পর Asynchronous Worker টিকিট তৈরি করে এবং Passenger-কে SMS/Email পাঠায়

### Train Ticket Booking System — Important SQL for Interview

এই query-গুলো PostgreSQL ধরে লেখা হয়েছে। Booking confirmation, expiry,
cancellation এবং seat release-এর query একই transaction-এর মধ্যে চালাতে হবে।
`NOW()` database time ব্যবহার করে, তাই frontend-এর clock-এর ওপর নির্ভর করা হবে
না।

### 3.3.1 Core Booking Queries

#### 1. Seat Lock — একই seat একসঙ্গে অন্য user যেন book করতে না পারে

```sql
SELECT *
FROM trip_seats
WHERE trip_id = 101
    AND id IN (101, 102)
    AND status = 'AVAILABLE'
ORDER BY id
FOR UPDATE;
```

`FOR UPDATE` transaction শেষ হওয়া পর্যন্ত selected row lock করে। Application
কে check করতে হবে যে requested সব seat পাওয়া গেছে; না হলে `ROLLBACK` করে
`409 Conflict` দিতে হবে।

#### 2. Create Booking — Booking তৈরি করে ১০ মিনিট payment-এর সময় দেওয়া

```sql
INSERT INTO bookings (user_id, trip_id, status, total_amount, expires_at)
VALUES (1, 101, 'PENDING_PAYMENT', 800.00,
                NOW() + INTERVAL '10 minutes')
RETURNING id, status, expires_at;
```
এই কুয়েরিটি নির্দিষ্ট ইউজার ও ট্রিপের জন্য ৮০০ টাকা মূল্যের একটি নতুন বুকিং তৈরি করে ১০ মিনিটের জন্য PENDING_PAYMENT স্ট্যাটাসে ডাটাবেজে সেভ করে এবং তৈরি হওয়া বুকিংয়ের id, status ও expires_at রিটার্ন করে।

#### 3. Hold Seats — Payment-এর আগে seat সাময়িকভাবে reserve করা

```sql
UPDATE trip_seats
SET status = 'HELD',
        booking_id = 1,
        hold_expires_at = NOW() + INTERVAL '10 minutes'
WHERE trip_id = 101
    AND id IN (101, 102)
    AND status = 'AVAILABLE';
```

Affected row count requested seat count-এর সমান না হলে transaction rollback
করতে হবে। চারটি seat-এর ক্ষেত্রে আংশিক hold রাখা যাবে না।

#### 4. Payment Success — Payment সফল হলে booking confirm করা

```sql
UPDATE bookings
SET status = 'CONFIRMED',
        expires_at = NULL
WHERE id = 1
    AND status = 'PENDING_PAYMENT'
    AND expires_at > NOW()
RETURNING id, status;
```

এই update-এর আগে payment status, amount এবং gateway transaction server-side
verify করতে হবে। Affected row count zero হলে booking already confirmed,
expired অথবা invalid হতে পারে।

#### 5. Book Seats — Confirmed booking-এর seat স্থায়ীভাবে reserve করা

```sql
UPDATE trip_seats
SET status = 'BOOKED',
        hold_expires_at = NULL
WHERE booking_id = 1
    AND status = 'HELD'
RETURNING id, seat_number, status;
```

Booking update এবং seat update একই confirmation transaction-এ করতে হবে।

#### 6. Expire Booking — Payment-এর সময় শেষ হলে booking expire করা

```sql
UPDATE bookings
SET status = 'EXPIRED'
WHERE id = 1
    AND status = 'PENDING_PAYMENT'
    AND expires_at <= NOW()
RETURNING id, status;
```

Worker একসঙ্গে অনেক booking process করলে `FOR UPDATE SKIP LOCKED` দিয়ে
candidate claim করা যায়। Expiry-এর আগে payment success এলে payment flow এবং
expiry flow একই booking row lock করে state পুনরায় যাচাই করবে।

#### 7. Release Seats — Expired booking-এর seat আবার available করা

```sql
UPDATE trip_seats
SET status = 'AVAILABLE',
        booking_id = NULL,
        hold_expires_at = NULL
WHERE booking_id = 1
    AND status = 'HELD';
```

Booking `EXPIRED` করা এবং seat release করা একই transaction-এ করতে হবে।

#### 8. Check Payment — Booking-এর payment status দেখা

```sql
SELECT status, amount, gateway_transaction_id, created_at
FROM payments
WHERE booking_id = 1
ORDER BY created_at DESC;
```

#### 9. Prevent Duplicate Webhook — একই webhook event যেন দুবার process না হয়

```sql
INSERT INTO payment_webhook_events (provider, event_id, payload)
VALUES ('BKASH', 'evt_123', '{}'::jsonb)
ON CONFLICT (provider, event_id) DO NOTHING
RETURNING id;
```

`RETURNING`-এ কোনো row না এলে event আগে process বা record করা হয়েছে। এজন্য
`UNIQUE (provider, event_id)` constraint থাকতে হবে।

#### 10. Find Available Seats — কোন seat available আছে তা দেখা

```sql
SELECT id, seat_number, fare
FROM trip_seats
WHERE trip_id = 101
    AND status = 'AVAILABLE'
ORDER BY seat_number;
```

এই query display-এর জন্য। Final availability booking transaction-এর মধ্যে
আবার যাচাই করতে হবে।

### 3.3.2 Search, Cancellation এবং Reporting Queries

#### 11. Search Trains — নির্দিষ্ট route ও date-এ train খুঁজে বের করা

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

#### 12. Booking History — Passenger-এর সব booking দেখা

```sql
SELECT id, trip_id, status, total_amount, expires_at, created_at
FROM bookings
WHERE user_id = 1
ORDER BY created_at DESC;
```

#### 13. Booking Details — Booking-এর সঙ্গে train ও journey information দেখা

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

`user_id` দিয়ে filter করা authorization-এর জন্য গুরুত্বপূর্ণ।

#### 14. Cancel Booking — Pending booking বাতিল করা

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

Confirmed booking cancel করলে আলাদা refund policy এবং `REFUND_PENDING`
workflow দরকার।

#### 15. Refund Status — Refund-এর অবস্থা দেখা

```sql
SELECT booking_id, amount, status, gateway_transaction_id
FROM payments
WHERE booking_id = 1
    AND status IN ('REFUND_PENDING', 'REFUNDED')
ORDER BY created_at DESC;
```

#### 16. Count Available Seats — কতগুলো seat available আছে তা দেখা

```sql
SELECT COUNT(*) AS available_seats
FROM trip_seats
WHERE trip_id = 101
    AND status = 'AVAILABLE';
```

#### 17. Total Revenue — Successful payment থেকে মোট revenue হিসাব করা

```sql
SELECT COALESCE(SUM(amount), 0) AS gross_revenue
FROM payments
WHERE status = 'SUCCEEDED';
```

এটি gross successful payment। Refund বাদ দিয়ে net revenue বের করতে refund
amount বা refund transaction আলাদাভাবে হিসাব করতে হবে।

#### 18. Prevent Double Booking — একই trip-এর একই seat একাধিক active booking-এ আটকানো

যদি `booking_seats` table-এ allocation `status` (`HELD`/`BOOKED`) রাখা হয়,
তাহলে historical rows রেখেও active allocation duplicate হতে না দেওয়ার জন্য:

```sql
CREATE UNIQUE INDEX unique_active_seat_allocation
ON booking_seats (trip_seat_id)
WHERE status IN ('HELD', 'BOOKED');
```

এই README-এর বর্তমান schema-তে ownership `trip_seats.booking_id` এবং
`trip_seats.status`-এ রাখা হয়েছে। সেই design-এ `UNIQUE (trip_id,
seat_number)` এবং row locking/conditional update-ই মূল protection; একই
ownership দু জায়গায় আলাদা করে store করা উচিত নয়।

#### 19. Transaction — একাধিক SQL operation একসঙ্গে সফল বা ব্যর্থ করা

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

#### 20. Second Highest Fare — SQL interview-এর common problem

```sql
SELECT MAX(fare) AS second_highest_fare
FROM trip_seats
WHERE fare < (
        SELECT MAX(fare)
        FROM trip_seats
);
```

এটি distinct দ্বিতীয় সর্বোচ্চ fare বের করে। কোনো দ্বিতীয় distinct value না
থাকলে result `NULL` হবে।

#### 21 : Find the 2nd Highest Salary
```sql
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
);
```
### 3.3.3 Useful Integrity Constraints এবং Indexes

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

Database constraint duplicate data আটকায়, কিন্তু payment gateway-এর সঙ্গে
database-এর একটি ACID transaction তৈরি করে না। তাই webhook idempotency,
retry এবং reconciliation worker এখনও প্রয়োজন।

### 3.4 Component-এর দায়িত্ব

-   **Client (React/Next.js):** Train search, seat selection, payment
    checkout ও booking status দেখায়।
-   **Express API:** REST request গ্রহণ করে; authentication, validation ও
    business logic চালায়।
-   **Booking Service:** Seat hold, booking create, confirmation,
    cancellation ও expiry rules enforce করে।
-   **PostgreSQL:** Train, trip, seat, booking ও payment-এর
    authoritative data রাখে।
-   **Ticket Service:** Verified payment-এর পরে ticket issue এবং ticket
    lookup handle করে।
-   **Payment Gateway:** Payment process করে এবং server-to-server
    webhook পাঠায়।
-   **Background Worker:** Expired booking খুঁজে payment state যাচাই করে
    এবং প্রয়োজন হলে seat release করে।

## 4. Database Design

আটটি প্রধান table:

1.  `users`
2.  `trains`
3.  `trips`
4.  `trip_seats`
5.  `bookings`
6.  `booking_seats`
7.  `payments`
8.  `tickets`

### 4.1 `users`

User account information রাখে।

  Column         Type              বিবরণ
  -------------- ----------------- -------------------
  `id`           UUID, PK          Unique user ID
  `name`         VARCHAR           User-এর নাম
  `email`        VARCHAR, UNIQUE   User-এর email
  `created_at`   TIMESTAMPTZ       Account তৈরির সময়

### 4.2 `trains`

Train-এর সাধারণ তথ্য রাখে।

  Column           Type              বিবরণ
  ---------------- ----------------- -----------------
  `id`             UUID, PK          Unique train ID
  `train_number`   VARCHAR, UNIQUE   Train number
  `train_name`     VARCHAR           Train name

### 4.3 `trips`

একটি নির্দিষ্ট তারিখ ও সময়ে একটি train-এর যাত্রাকে trip বলা হয়। একই
train-এর আলাদা তারিখের জন্য আলাদা trip record থাকবে।

  Column                Type          বিবরণ
  --------------------- ------------- -----------------------
  `id`                  UUID, PK      Unique trip ID
  `train_id`            UUID, FK      `trains.id` reference
  `departure_station`   VARCHAR       যাত্রা শুরুর station
  `arrival_station`     VARCHAR       গন্তব্য station
  `departure_time`      TIMESTAMPTZ   Departure date/time
  `arrival_time`        TIMESTAMPTZ   Arrival date/time

### 4.4 `trip_seats`

একটি নির্দিষ্ট trip-এর প্রতিটি seat-এর availability রাখে। একই physical seat
অন্য trip-এ আলাদাভাবে available হতে পারে।

  Column              Type                বিবরণ
  ------------------- ------------------- ---------------------------
  `id`                UUID, PK            Unique trip-seat ID
  `trip_id`           UUID, FK            `trips.id` reference
  `seat_number`       VARCHAR             যেমন `A1`, `A2`
  `status`            VARCHAR/ENUM        Seat status
  `hold_expires_at`   TIMESTAMPTZ, NULL   Hold deadline
  `booking_id`        UUID, NULL, FK      বর্তমান hold/booking owner
  `fare`              DECIMAL             Seat fare

**Constraint:** `UNIQUE (trip_id, seat_number)` রাখতে হবে।

### 4.5 `bookings`

Booking-এর overall state, user, total amount ও expiry deadline রাখে।

  Column           Type           বিবরণ
  ---------------- -------------- -----------------------------
  `id`             UUID, PK       Unique booking ID
  `user_id`        UUID, FK       `users.id` reference
  `trip_id`        UUID, FK       `trips.id` reference
  `status`         VARCHAR/ENUM   Booking lifecycle status
  `total_amount`   DECIMAL        Selected seat-গুলোর মোট fare
  `expires_at`     TIMESTAMPTZ    ১০ মিনিটের deadline
  `created_at`     TIMESTAMPTZ    Booking তৈরির সময়

### 4.6 `booking_seats`

একটি booking-এর সঙ্গে একাধিক seat যুক্ত করে। চারটি seat-এর জন্য **একটি
booking row এবং চারটি `booking_seats` row** তৈরি হবে।

  Column           Type          বিবরণ
  ---------------- ------------- ----------------------------------
  `booking_id`     UUID, PK/FK   `bookings.id` reference
  `trip_seat_id`   UUID, PK/FK   `trip_seats.id` reference
  `fare`           DECIMAL       Booking-এর সময় সংরক্ষিত seat fare

Composite primary key: `PRIMARY KEY (booking_id, trip_seat_id)`।

এ ছাড়া একই trip-seat একাধিক active booking-এ assign না হওয়ার নিয়ম
database constraint ও allocation logic দিয়ে নিশ্চিত করতে হবে।

### 4.7 `payments`

Payment attempt ও gateway transaction-এর ফলাফল রাখে।

  --------------------------------------------------------------------------
  Column                     Type                    বিবরণ
  -------------------------- ----------------------- -----------------------
  `id`                       UUID, PK                Unique payment ID

  `booking_id`               UUID, FK                `bookings.id` reference

  `gateway_transaction_id`   VARCHAR, UNIQUE         Gateway transaction
                                                     reference

  `amount`                   DECIMAL                 Payment amount

  `status`                   VARCHAR/ENUM            Payment status

  `created_at`               TIMESTAMPTZ             Payment attempt তৈরির
                                                     সময়
  --------------------------------------------------------------------------

এক booking-এর জন্য প্রয়োজন অনুযায়ী একাধিক payment attempt থাকতে পারে।
Gateway transaction ও event idempotently process করতে হবে।

### 4.8 `tickets`

Payment verify হওয়ার পর confirmed booking-এর জন্য ticket issue করার তথ্য
রাখে।

Column           Type              বিবরণ
--------------   ----------------  ------------------------------
`id`             UUID, PK          Unique ticket ID
`booking_id`     UUID, FK          `bookings.id` reference
`ticket_number`  VARCHAR, UNIQUE   Public ticket number
`status`         VARCHAR/ENUM      Ticket lifecycle status
`issued_at`      TIMESTAMPTZ       Ticket issue হওয়ার সময়

Ticket কেবল verified payment এবং `CONFIRMED` booking-এর পর issue করা যাবে।

## 5. Status Management

Booking, Seat, Payment এবং Ticket-এর status আলাদা রাখতে হবে। কারণ এগুলোর
lifecycle ও অর্থ এক নয়।

### 5.1 Booking Status

  Status              অর্থ
  ------------------- ------------------------------------------------------
  `PENDING_PAYMENT`   Booking তৈরি হয়েছে; payment এখনো verified হয়নি।
  `CONFIRMED`         Payment যাচাই এবং booking confirmation সফল।
  `EXPIRED`           Payment deadline পার হয়েছে এবং booking expire হয়েছে।
  `FAILED`            Booking process ব্যর্থ হয়েছে, policy অনুযায়ী।
  `CANCELLED`         Booking বাতিল করা হয়েছে।
  `REFUND_PENDING`    Refund processing চলছে।
  `REFUNDED`          Refund সম্পন্ন হয়েছে।

### 5.2 Seat Status

  Status        অর্থ
  ------------- -----------------------------------------------
  `AVAILABLE`   Seat reserve করার জন্য খালি।
  `HELD`        Seat সাময়িকভাবে একটি booking-এর জন্য hold করা।
  `BOOKED`      Seat confirmed booking-এর জন্য বরাদ্দ।
  `BLOCKED`     Seat operator/admin বন্ধ রেখেছে।

### 5.3 Payment Status

  Status             অর্থ
  ------------------ -----------------------------------------------
  `PENDING`          Payment initiated বা final result-এর অপেক্ষায়।
  `SUCCEEDED`        Gateway payment সফল বলে নিশ্চিত করেছে।
  `FAILED`           Payment ব্যর্থ।
  `CANCELLED`        Payment attempt বাতিল।
  `REFUND_PENDING`   Refund প্রক্রিয়াধীন।
  `REFUNDED`         Refund সম্পন্ন।

### 5.4 Ticket Status

  Status         অর্থ
  -------------- ---------------------------------
  `NOT_ISSUED`   Ticket এখনো issue হয়নি।
  `ISSUED`       Ticket issue হয়েছে।
  `USED`         Ticket যাত্রার জন্য ব্যবহার হয়েছে।
  `CANCELLED`    Ticket বাতিল হয়েছে।

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

Payment `SUCCEEDED` হলেই booking স্বয়ংক্রিয়ভাবে `CONFIRMED` ধরে নেওয়া যাবে
না। Backend-কে reservation validity, expiry policy এবং seat ownership
যাচাই করতে হবে।

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

-   একজন user অনেক booking করতে পারেন।
-   একটি train-এর একাধিক trip থাকতে পারে।
-   একটি trip-এর একাধিক trip-seat থাকে।
-   একটি booking একাধিক seat অন্তর্ভুক্ত করতে পারে।
-   একটি booking-এর একাধিক payment attempt থাকতে পারে।
-   একটি confirmed booking-এর জন্য ticket issue করা হয়।

Implementation-এ `trip_seats.booking_id` বর্তমান hold/confirmed owner
বোঝাতে পারে; seat release হলে এটি clear করতে হবে। `booking_seats`-এর
historical relationship কীভাবে সংরক্ষণ হবে, তা স্পষ্টভাবে define করতে হবে,
যেন duplicate allocation না হয়।

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

চারটি seat-কে একটি logical reservation হিসেবে ধরতে হবে। Diagram-এর
availability check কেবল preliminary check; transaction-এর মধ্যে row lock
নেওয়ার পরেই final validation করতে হবে। Requirement যদি
চারটি seat একসঙ্গে বুক করা হয়, তাহলে দুই বা তিনটি seat reserve করে বাকি
seat বাদ দিয়ে booking confirm করা যাবে না।

## 9. Double Booking Prevention

ধরা যাক, User A ও User B একই সময়ে `A1` seat book করতে চায়।

Naive implementation:

``` js
const seat = await findSeat(seatId);

if (seat.status === "AVAILABLE") {
  await updateSeat(seatId, "HELD");
}
```

এটি unsafe: দুই request-ই update হওয়ার আগে `AVAILABLE` পড়ে ফেলতে পারে।

### সঠিক পদ্ধতি

1.  Database transaction শুরু করা।
2.  Requested seat row-গুলো consistent order-এ lock করা।
3.  নিশ্চিত করা যে সব চারটি seat একই trip-এর এবং available।
4.  একটি booking record তৈরি করা।
5.  চারটি seat-কে `HELD` করা এবং একই expiry deadline দেওয়া।
6.  চারটি `booking_seats` record তৈরি করা।
7.  সব সফল হলে commit; কোনো validation fail করলে rollback।

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

`SELECT ... FOR UPDATE` একাই যথেষ্ট নয়। Seat validation, expired hold
reclaim, conditional update, unique constraints এবং transaction error
handling-ও প্রয়োজন। Payment সম্পন্ন হওয়া পর্যন্ত database transaction খোলা
রাখা যাবে না। আগে seat hold করে transaction commit করতে হবে; এরপর
gateway-এর সঙ্গে যোগাযোগ করতে হবে।

### Q: কনকারেন্ট সিস্টেমে (Concurrent System) ডাবল বুকিং (Double Booking) কীভাবে প্রতিরোধ করবেন?
A:সাধারণত কোডে আমরা প্রথমে সিট ফাঁকা আছে কি না চেক করি (if (seat.status === 'AVAILABLE')) এবং তারপর আপডেট করি। কিন্তু হাই-কনকারেন্সি সিস্টেমে (যখন হাজার হাজার ইউজার একসঙ্গে রিকোয়েস্ট পাঠায়) এটি অত্যন্ত ঝুঁকিপূর্ণ। কারণ একাধিক ইউজার একই মিলিিসেকেন্ডে সিটগুলোকে AVAILABLE পড়তে পারে এবং দুজনেই আপডেট করার চেষ্টা করলে Race Condition তৈরি হয়ে ডাবল বুকিং হয়ে যায়।

এই সমস্যার সঠিক সমাধান হলো ডেটাবেজ লেভেলে ACID Transaction এবং Row-Level Locking ব্যবহার করা:
“একই সিট যেন একাধিক passenger বুক করতে না পারে, সেটা নিশ্চিত করার জন্য আমি ACID Transaction, Row-Level Locking এবং Deadlock Prevention ব্যবহার করব।
##### ১. Transaction শুরু: 
প্রথমে BEGIN দিয়ে transaction শুরু করব, যাতে booking-এর সব database operation একটি atomic unit হিসেবে কাজ করে।

-ট্রানজেকশন (Transaction) হলো ডাটাবেজ ম্যানেজমেন্ট সিস্টেমের (DBMS) একটি একক বা লজিক্যাল কাজের একক (Logical Unit of Work)।

ট্রানজেকশন নিশ্চিত করে যে—হয় দুটি ধাপই সফলভাবে সম্পন্ন হবে (Commit), না হয় কোনো ধাপই প্রযোজ্য হবে না (Rollback)।ACID Properties (ট্রানজেকশনের ৪টি মূল স্তম্ভ)
#####  ২. Row-Level Locking:
 এরপর SELECT ... FOR UPDATE ব্যবহার করে passenger যে সিটগুলো বুক করতে চায়, সেগুলোর row lock করব। ফলে অন্য কোনো concurrent transaction একই row lock করে সেটিকে পরিবর্তন করতে পারবে না; তাকে অপেক্ষা করতে হবে।
##### ৩. Consistent Locking Order: 
একাধিক সিট lock করার সময় ORDER BY id অনুযায়ী একই ক্রমে lock করব। এতে deadlock-এর ঝুঁকি কমে।
##### ৪. Deadlock কী এবং কেন হয়: 
Deadlock হয় যখন দুটি transaction একে অপরের lock করা resource-এর জন্য অপেক্ষা করে এবং কেউই সামনে এগোতে পারে না।
উদাহরণস্বরূপ:
- Transaction A প্রথমে Seat 1 lock করল, তারপর Seat 2-এর জন্য অপেক্ষা করছে।
- Transaction B প্রথমে Seat 2 lock করল, তারপর Seat 1-এর জন্য অপেক্ষা করছে।
- A অপেক্ষা করছে B-এর জন্য, আর B অপেক্ষা করছে A-এর জন্য। ফলে একটি circular wait তৈরি হয়।
Deadlock-এর উদাহরণ
Transaction A
Lock: Seat 1
Wait: Seat 2


Transaction B
Lock: Seat 2
Wait: Seat 1



Circular Wait → Deadlock


##### ৫. Deadlock Prevention: 
এই সমস্যা কমাতে সব transaction-এ সিটগুলোকে একই ক্রমে, যেমন ascending id অনুযায়ী lock করব। ফলে একটি transaction আগে Seat 1 এবং পরে Seat 2 lock করলে অন্য transaction-ও একই order অনুসরণ করবে। এতে উপরের circular-wait পরিস্থিতি এড়ানো যায়।


ধরো, Passenger 1 ও Passenger 2 দুই জায়গা থেকে request পাঠাচ্ছে।

১. Passenger-এর request
Passenger 1  Request  ["DF", "C3"]

Passenger 2  Request  ["C3", "DF"]

Backend-এর Lock Ordering Seat ID অনুযায়ী sort করবে

Passenger 1  C3 → DF


Passenger 2  C3 → DF

দুজনের lock order একই!

##### ৬. Deadlock Recovery: 
শুধু consistent locking order যথেষ্ট নয়; অন্য table বা resource lock করার কারণে deadlock হতে পারে। তাই database deadlock detect করে কোনো একটি transaction abort করলে, application-এ deadlock error শনাক্ত করে সীমিত সংখ্যক retry এবং exponential backoff ব্যবহার করব।
##### ৭. Availability যাচাই: 
Lock নেওয়ার পর যাচাই করব প্রতিটি সিট সঠিক trip-এর কি না এবং সিটের status এখনো AVAILABLE কি না।
##### ৮. Seat Hold ও Booking:
 সব সিট available থাকলে সেগুলোকে HELD status-এ পরিবর্তন করব এবং booking record তৈরি করব, যার expiry হবে ১০ মিনিট পর।
##### ৯. Commit অথবা Rollback:
সব operation সফল হলে COMMIT করব। কোনো সিট unavailable হলে বা operation ব্যর্থ হলে ROLLBACK করব, যাতে আংশিক booking তৈরি না হয়।
##### ১০. Database Constraint: 
অতিরিক্ত সুরক্ষার জন্য প্রয়োজন অনুযায়ী unique constraint এবং conditional update ব্যবহার করব, যাতে application-level logic-এর বাইরেও duplicate booking প্রতিরোধ করা যায়।
এভাবে transaction, consistent lock ordering, deadlock recovery এবং database constraints একসঙ্গে ব্যবহার করে concurrent booking-এর মধ্যেও একই সিট একাধিক passenger-কে বরাদ্দ হওয়া প্রতিরোধ করা যায়।”

#### Deadlock নিয়ে ইন্টারভিউতে যে 4 টি বিষয় মনে রাখবে :
মনে করুন, ঢাকা রেলওয়ে স্টেশনের অনলাইন ও অফলাইন টিকিট কাউন্টারে একই সময়ে দুইজন যাত্রী (যাত্রী A এবং যাত্রী B) একই ট্রেনের ২টি টিকিট (সিট ১ এবং সিট ২) কাটার চেষ্টা করছেন।
##### ১. Mutual Exclusion (পারস্পরিক বর্জন)ঘটনা: 
-ট্রেনের নির্দিষ্ট একটি সিট (যেমন: সিট ১) একবারে মাত্র একজন যাত্রীকেই দেওয়া সম্ভব, দুজনকে একসাথে দেওয়া সম্ভব নয়।

-শর্তটি যেভাবে কাজ করে: সিস্টেমের নিয়ম অনুযায়ী একটি সিট কখনো শেয়ার করা যাবে না (Non-shareable)। ফলে সিট ১ যদি 'যাত্রী A' সিলেক্ট করে ফেলে, তবে অন্য কাউকে ওই সিটটি পাওয়ার জন্য অপেক্ষা করতেই হবে।
##### ২. Hold and Wait (ধরে রাখা এবং অপেক্ষা করা)ঘটনা:
-যাত্রী A সফলভাবে সিট ১ বুকিংয়ের জন্য হোল্ড (Hold) করে রেখেছে, এবং বুকিং কনফার্ম করতে সিট ২ পাওয়ার জন্য অপেক্ষা (Wait) করছে।যাত্রী B সফলভাবে সিট ২ বুকিংয়ের জন্য হোল্ড (Hold) করে রেখেছে, এবং বুকিং কনফার্ম করতে সিট ১ পাওয়ার জন্য অপেক্ষা (Wait) করছে।

-শর্তটি যেভাবে কাজ করে: কেউই তার হাতে থাকা সিটের বুকিং ছাড়ছে না, আবার পরের সিটের জন্য অপেক্ষা করে বসে আছে।
##### ৩. No Preemption (জোরপূর্বক কেড়ে না নেওয়া)ঘটনা:
-টিকিটিং সিস্টেমের নিয়ম অনুযায়ী, কোনো যাত্রী সিট পেমেন্ট প্রসেসে বা কার্টে রাখলে সিস্টেম জোর করে বা জোরজবরদস্তি করে তার কার্ট থেকে সিট কেড়ে নিয়ে অন্য যাত্রীকে দিয়ে দেবে না।

-শর্তটি যেভাবে কাজ করে: যাত্রী নিজে বুকিং ক্যানসেল না করা পর্যন্ত বা পেমেন্ট টাইমআউট না হওয়া পর্যন্ত কাউন্টার বা সিস্টেম তার বুকিং করা সিটটি কেড়ে নিতে পারবে না।
##### ৪. Circular Wait (চক্রাকার অপেক্ষা)ঘটনা:
যাত্রী A ছেড়ে দেবে সিট ১, কিন্তু সে অপেক্ষা করছে যাত্রী B কখন সিট ২ ছাড়বে তার জন্য।যাত্রী B ছেড়ে দেবে সিট ২, কিন্তু সে অপেক্ষা করছে যাত্রী A কখন সিট ১ ছাড়বে তার জন্য।

🚨 ফলাফল (Deadlock)যেহেতু ৪টি শর্তই একই সাথে পূরণ হয়ে গেছে, তাই:যাত্রী A বা যাত্রী B কেউই টিকিট কাটতে পারছে না।

-টিকিটিং সিস্টেম আটকে বা হ্যাং হয়ে গেছে।বাস্তব জীবনে এই ডেডলক ভাঙতে স্টেশনের টিকিট কাউন্টার বা অনলাইন সিস্টেম Payment Timeout ব্যবহার করে—অর্থাৎ 3/4/10 মিনিট পর স্বয়ংক্রিয়ভাবে একটি বুকিং ক্যানসেল (Rollback) করে দেয় যাতে অন্যজন টিকিটটি পেতে পারে।

### Q: এই ডিজাইনে পেমেন্ট প্রসেসিংয়ের ক্ষেত্রে সবচেয়ে গুরুত্বপূর্ণ গোল্ডেন রুল (Golden Rule) কী?
A:সবচেয়ে গুরুত্বপূর্ণ টেকনিক্যাল পয়েন্ট হলো—পেমেন্ট সম্পন্ন হওয়া পর্যন্ত ডেটাবেজ ট্রানজেকশন খোলা রাখা যাবে না।

কেন খোলা রাখব না?

পেমেন্ট গেটওয়ে (যেমন: বিকাশ বা কার্ড কোম্পানি)-র সাথে কমিউনিকেট করতে একটু সময় লাগে (Network Delay)। আমরা যদি পেমেন্ট শেষ হওয়া পর্যন্ত ডেটাবেজ কানেকশন বা ট্রানজেকশন হোল্ড করে রাখি, তাহলে অল্প কিছু ইউজার রিকোয়েস্ট আসলেই ডেটাবেজের সব কানেকশন শেষ (exhaust) হয়ে যাবে এবং পুরো সিস্টেম ডাউন হয়ে যেতে পারে।

সঠিক পদ্ধতি কী?

১. প্রথমে আমরা খুব দ্রুত ডেটাবেজে ঢুকে সিটগুলো ১০ মিনিটের জন্য HELD করে ট্রানজেকশন COMMIT করে দেব (অর্থাৎ ডেটাবেজ কাজ শেষ করে একদম ফ্রি হয়ে যাবে)।

২. ডেটাবেজ ফ্রি হওয়ার পর আমরা পেমেন্ট গেটওয়ের সাথে কথা বলা শুরু করব।

৩. পেমেন্ট সফল হলে বুকিং CONFIRMED হবে, আর ১০ মিনিটের মধ্যে পেমেন্ট না করলে ব্যাকগ্রাউন্ড ওয়ার্কার (Cron Job) নিজে থেকেই সিটগুলো আবার সবার জন্য রিলিজ বা খালি করে দেবে।

### Q: Cron Job কী?
A:Cron Job (বা Cron Worker) হলো একটি ব্যাকগ্রাউন্ড প্রসেস বা সিস্টেম অটোমেশন টুল, যা নির্দিষ্ট সময় পর পর বা নির্দিষ্ট একটি ইন্টারভেলে (যেমন: প্রতি ৩০ সেকেন্ড বা ১ মিনিট পর পর) স্বয়ংক্রিয়ভাবে কোনো নির্দিষ্ট কাজ বা টাস্ক রান করায়।

ট্রেন বুকিং সিস্টেমে ক্রন জব ব্যাকগ্রাউন্ডে রান করে মেয়াদোত্তীর্ণ (expired) পেন্ডিং বুকিংগুলো খুঁজে বের করে এবং আটকে থাকা সিটগুলো রিলিজ করে দেয়।

##### সমস্যাটি আসলে কী? (The Problem Context)
ধরে নেওয়া যাক, অনলাইন ট্রেন বুকিং সিস্টেমে কোনো যাত্রী একটি টিকিট কাটার জন্য ক্লিক করলেন।

Seat Lock: সিস্টেম তাৎক্ষণিকভাবে সিটটিকে PENDING বা LOCKED স্টেটাসে নিয়ে যায় (যাতে অন্য কেউ সেটি কিনতে না পারে)।

Payment Window: যাত্রীকে পেমেন্ট করার জন্য ১০ মিনিট সময় দেওয়া হয়।

Abandonment: কিন্তু যাত্রী যদি পেমেন্ট না করে ব্রাউজার ট্যাবটি কেটে দেন, বিকাশ/রকেট অ্যাপের পিন না দিয়ে বের হয়ে যান, অথবা ইন্টারনেট বিচ্ছিন্ন হয়ে যায়, তখন কী হবে?

যদি ডাটাবেজের ওই সিটটি চিরকাল PENDING হিসেবেই থেকে যায়, তবে সেটি আর কখনো কোনো যাত্রীর কাছে বিক্রি করা যাবে না। একেই বলা হয় Stale Data বা Orphaned Lock।

##### 2. Cron Job কীভাবে এই সমস্যার সমাধান করে? (The Mechanism)
ক্রন জব হলো সার্ভারে ব্যাকগ্রাউন্ডে চলতে থাকা একটি সময়সূচীভিত্তিক অটোমেশন সার্ভিস (Scheduled Task)।


Schedule Trigger: ক্রন জবটি ব্যাকগ্রাউন্ডে অনবরত প্রতি ১ মিনিট পর পর জাগ্রত (Trigger) হয়।

Database Scanning: ডাটাবেজে গিয়ে একটি কুয়েরি চালায়:

"যেসব বুকিংয়ের স্টেটাস এখনও PENDING কিন্তু ১০ মিনিটের বেশি সময় পার হয়ে গেছে, সেগুলোকে খুঁজে বের করো।"

Status Update & Seat Release: খুঁজে পাওয়া বুকিংগুলোকে একটি Transaction-এর মাধ্যমে:

বুকিং স্টেটাস PENDING থেকে পরিবর্তন করে EXPIRED বা CANCELLED করে দেয়।

সংশ্লিষ্ট সিটের স্টেটাস AVAILABLE করে দেয়, যাতে স্টেশনের বা অ্যাপের অন্য যাত্রীরা মুহূর্তের মধ্যেই সিটটি কিনে নিতে পারেন।

## 10. ১০ মিনিটের Hold ও Expiry Mechanism

### 10.1 Booking তৈরি হলে

ধরা যাক:

``` text
created_at = 10:00:00
expires_at = 10:10:00
status     = PENDING_PAYMENT
```

চারটি seat-এর status হবে `HELD`।

Server-side deadline:

``` text
expiresAt = databaseCurrentTime + 10 minutes
```

Frontend countdown দেখাতে পারে, কিন্তু timer-এর authority frontend নয়।
Backend এবং database expiry enforce করবে।

### 10.2 Background Expiry Worker

Worker নির্দিষ্ট বিরতিতে deadline পার হওয়া booking খুঁজবে:

``` sql
SELECT id
FROM bookings
WHERE status = 'PENDING_PAYMENT'
  AND expires_at <= NOW();
```

এটি কেবল candidate booking খুঁজে বের করে; query-টি একা booking expire করার
জন্য যথেষ্ট নয়।

নিরাপদ worker process:

1.  Payment status প্রয়োজন হলে database transaction-এর বাইরে gateway থেকে
    যাচাই করা। Gateway call-এর সময় database lock ধরে রাখা যাবে না।
2.  Transaction শুরু করে booking row lock/claim করা।
3.  আবার যাচাই করা যে booking এখনো `PENDING_PAYMENT` এবং deadline পার
    হয়েছে।
4.  Payment success বা unresolved state থাকলে policy অনুযায়ী
    reconciliation করা; অন্ধভাবে seat release না করা।
5.  Unpaid booking-কে `EXPIRED` করা।
6.  Booking-এর অধীনে থাকা চারটি held seat-কে `AVAILABLE` করা এবং current
    booking ownership clear করা।
7.  Transaction commit করা।

একাধিক worker instance একই booking process করলেও যেন state corruption না
হয়, সে জন্য row locking/claiming ও conditional state transition ব্যবহার
করতে হবে। Worker দেরিতে চলতে পারে, তাই API-ও expiry rule enforce করবে।

### 10.3 Timeline

  -----------------------------------------------------------------------
  সময়                                 ঘটনা
  ----------------------------------- -----------------------------------
  10:00 AM                            চারটি seat `HELD` হলো

  10:03 AM                            User payment শুরু করলেন

  10:06 AM                            Payment verify হলো

  10:06 AM                            Booking `CONFIRMED`, seat `BOOKED`

  বিকল্প: 10:10 AM                     Payment সফল না হলে booking expire
                                      হবে এবং seat release হবে
  -----------------------------------------------------------------------

Payment gateway ও database আলাদা system। তাই payment outcome যাচাই এবং
expiry worker-এর মধ্যে race condition সামলাতে হবে।

## 11. Payment Failure ও Delayed Webhook

### Problem (সমস্যা): Late Webhook Arrival & Cron Job Race Condition পরিস্থিতি:

১০:০৯:৫৮ AM: ইউজার পেমেন্ট গেটওয়েতে সফলভাবে পেমেন্ট সম্পন্ন করেন।

১০:১০:০০ AM: ব্যাকএন্ডের Cron Job ১০ মিনিট পূর্ণ হওয়ায় পেন্ডিং বুকিংটিকে EXPIRED ঘোষণা করে এবং সিটটি রিলিজ করে (AVAILABLE করে দেয়)।

১০:১০:০২ AM: অন্য একজন নতুন ইউজার এসে ওই ফাঁকা হওয়া সিটটি বুক করে ফেলেন।

১০:১০:০৫ AM: নেটওয়ার্ক ল্যাটেন্সির কারণে প্রথম ইউজারের পেমেন্টের Webhook সার্ভারে এসে পৌঁছায়, যেখানে স্ট্যাটাস SUCCESS বলা আছে।

ঝামেলা কোথায়?
পেমেন্ট সফল হওয়া সত্ত্বেও প্রথম ইউজারের সিটটি ইতিমধ্যে অন্য কাউকে দেওয়া হয়ে গেছে। এখন ব্যাকএন্ড যদি অন্ধের মতো সিট ছাড়াই পেমেন্ট কনফার্ম করতে চায় বা ভুলভাবে স্টেট আপডেট করে, তবে সিস্টেমের ডেটা করাপ্ট হবে এবং Double Booking (একই সিট দুজনকে দেওয়া) ঘটবে

#### সমাধান যেভাবে কাজ করে (Step-by-Step Explanation)
##### ১. অন্ধের মতো বিশ্বাস না করে যাচাই করা (Verification & Authoritative Check)
কী হয়: পেমেন্ট গেটওয়ে থেকে যে ওয়েব হুক (Webhook) মেসেজটি আসে, সিস্টেম সেটা সরাসরি বিশ্বাস করে না।

কেন ও কীভাবে: হ্যাকাররা অনেক সময় ফেক মেসেজ পাঠাতে পারে। তাই ব্যাকএন্ড প্রথমে চেক করে যে মেসেজের Signature ঠিক আছে কিনা। এরপর পেমেন্ট গেটওয়ের নিজের সার্ভারে API কল করে রিয়েল-টাইম কনফার্মেশন নেয় যে, "ভাই, এই ইউজারের টাকা কি সত্যি ব্যাংকে জমা হয়েছে?" যখন গেটওয়ে বলে হ্যাঁ হয়েছে, কেবল তখনই সিস্টেম পরবর্তী ধাপে যায়।

##### ২. ডাটাবেজে ধাক্কাধাক্কি আটকানো (Database Locking)
কী হয়: একদিকে ক্রন জব সিট এক্সপায়ারের কাজ করছে, অন্যদিকে পেমেন্ট সাকসেসের ওয়েব হুক ডাটা আপডেট করতে আসছে—দুটো প্রসেস একই সময়ে ডাটাবেজে আক্রমণ করতে পারে।

কেন ও কীভাবে: এটি রোধ করতে ডাটাবেজে Row-Level Lock (SELECT ... FOR UPDATE) ব্যবহার করা হয়। এর মানে হলো—ক্রন জব যখন একটি সিট নিয়ে কাজ করছে, তখন ওয়েব হুককে একটু বাইরে দাঁড়িয়ে অপেক্ষা করতে বলা হয় (বা উল্টোটা)। একজন শেষ না করা পর্যন্ত অন্যজন ওই সিট ছুঁতে পারে না।

##### ৩. সিট ফাঁকা না থাকলে কী হবে? (Seat Reconciliation)
কী হয়: ইউজারের পেমেন্ট সাকসেস, কিন্তু তার জন্য বরাদ্দ করা সিটটি ক্রন জব ইতিমধ্যে অন্য কাউকে দিয়ে দিয়েছে।

কেন ও কীভাবে: এখানে সিস্টেম খুব বুদ্ধিমত্তার সাথে দুটি কাজ করতে পারে:

বিকল্প সিট (Alternative Seat): যদি ট্রেনে বা বাসে ওই ক্লাসের অন্য কোনো সিট খালি থাকে, তবে সিস্টেম ঝামেলা না করে নতুন একটি সিট ওই ইউজারকে অ্যাসাইন করে বুকিং কনফার্ম করে দেয়।

টাকা ফেরত (Refund Path): যদি ট্রেন একদম ফুল হয়ে যায় এবং একটি সিটও ফাঁকা না থাকে, তখন বুকিংটি ক্যানসেল করে দিতে হয় এবং ইউজারের টাকা ফেরত পাঠানোর প্রসেস শুরু করতে হয়।

##### ৪. ডাটাবেজে লিখে রাখলেই টাকা ফেরত যায় না! (Real Gateway Refund API)
কী হয়: অনেক জুনিয়র ডেভেলপার ভুল করে ডাটাবেজে স্ট্যাটাস বদলে লিখে দেয় REFUNDED আর ভাবে ইউজারের টাকা বিকাশ বা ব্যাংকে চলে গেছে। কিন্তু বাস্তবে টাকা ফেরত যায় না।

কেন ও কীভাবে: ইউজারের কাছে রিয়েল টাকা পৌঁছাতে হলে ব্যাকএন্ড থেকে পেমেন্ট গেটওয়ের Refund API-এ রিকোয়েস্ট পাঠাতে হয় (gateway_transaction_id সহ)। গেটওয়ে সার্ভার যখন সফলভাবে টাকা কেটে ইউজারের অ্যাকাউন্টে পাঠিয়ে দেয় এবং কনফার্মেশন রেসপন্স দেয়, কেবল তখনই ডাটাবেজে স্ট্যাটাস পার্মানেন্টলি REFUND_SUCCESS আপডেট করতে হয়।

##### ৫. একই মেসেজ বারবার আসলে জগাখিচুড়ি পাকানো রোধ (Idempotency)
কী হয়: নেটওয়ার্ক সমস্যার কারণে পেমেন্ট গেটওয়ে একই ওয়েব হুক বারবার পাঠাতে পারে (Retries)।

কেন ও কীভাবে: সিস্টেম প্রতিটি ওয়েব হুকের একটি ইউনিক আইডি (event_id) চেক করে। যদি দেখে এই আইডি দিয়ে ইতিপূর্বে একবার কাজ করা হয়ে গেছে, তবে দ্বিতীয়বার আর টাকা কাটা বা রিফান্ড করার মতো কাজ না করে শুধু একটি সাধারণ 200 OK রেসপন্স পাঠিয়ে দেয়।

## 12. Database Transaction

চারটি seat reserve করার illustrative transaction:

``` text
BEGIN TRANSACTION

1. Requested trip-seat rows lock করো।
2. নিশ্চিত করো যে চারটি valid seat পাওয়া গেছে।
3. নিশ্চিত করো সব seat reserve করার উপযুক্ত।
4. Server-side fare calculate করো।
5. একটি PENDING_PAYMENT booking তৈরি করো।
6. চারটি seat-কে HELD করো।
7. hold_expires_at = database time + 10 minutes সেট করো।
8. চারটি booking_seats row তৈরি করো।

COMMIT
```

যেকোনো validation বা update ব্যর্থ হলে `ROLLBACK` করতে হবে। এরপর
transaction-এর বাইরে payment initiate করা হবে। Payment verify হলে দ্বিতীয়
transaction-এ booking confirm এবং seat `BOOKED` করা হবে---তবে
reservation validity ও payment policy মেনে।

## 13. ACID Properties

ACID হলো database transaction-এর চারটি গুরুত্বপূর্ণ property:

-   **A --- Atomicity**
-   **C --- Consistency**
-   **I --- Isolation**
-   **D --- Durability**

### 13.1 Atomicity --- সব হবে, না হলে কিছুই হবে না

চারটি seat-এর মধ্যে `A4` unavailable হলে `A1`, `A2`, `A3` reserve রেখে
দেওয়া যাবে না। পুরো transaction rollback হবে।

``` sql
BEGIN;

-- Lock and validate all four seats.
-- Create one booking.
-- Reserve all four seats.
-- Create four booking_seats records.

COMMIT;   -- সব operation সফল হলে
-- ROLLBACK; -- যেকোনো operation ব্যর্থ হলে
```

**Interview answer:** "Atomicity ensures that all four seats are
reserved together, or none of them are reserved."

### 13.2 Consistency --- Database rules ঠিক থাকবে

প্রতিটি transaction-এর আগে ও পরে database constraints এবং business
invariants বজায় থাকতে হবে।

মূল rules:

-   একই trip-এর একই seat একসঙ্গে একাধিক active booking-এ থাকবে না।
-   এই use case-এ চারটি seat বাধ্যতামূলক হলে booking-এ ঠিক চারটি seat
    থাকতে হবে।
-   Booking `CONFIRMED` করার আগে verified payment এবং valid seat
    allocation থাকতে হবে।
-   Expired booking-এর seat সঠিকভাবে release করা হবে।
-   Total fare server-side হিসাব অনুযায়ী হবে।

**Interview answer:** "Consistency ensures that every successful
transaction preserves database constraints and booking business rules."

### 13.3 Isolation --- Concurrent request নিরাপদ রাখা

User A ও User B একই seat চাইলে row-level locking বা atomic conditional
update নিশ্চিত করবে যে কেবল একজন valid reservation সফলভাবে claim করতে
পারে।

PostgreSQL-এর `SELECT ... FOR UPDATE` requested rows lock করতে পারে।
Transaction শেষ হলে lock release হয়। অন্য request পরে নতুন state যাচাই
করবে।

**Interview answer:** "Isolation prevents concurrent users from
successfully reserving the same seat. I would use row-level locking or
atomic conditional updates inside a transaction."

### 13.4 Durability --- Commit করা data স্থায়ী থাকবে

Transaction সফলভাবে commit হলে application crash বা restart হলেও
committed booking ও seat status database recovery-এর পরেও টিকে থাকার
কথা। PostgreSQL-এর transaction logging ও durable storage configuration এ
নিশ্চয়তায় সহায়তা করে।

তবে database commit এবং external payment gateway-এর state একই ACID
transaction নয়। এজন্য webhook retry, idempotency ও reconciliation দরকার।

**Interview answer:** "Durability ensures that confirmed bookings and
their committed database changes survive application crashes."

### Q: ACID properties কী এবং একটি ট্রেন বুকিং সিস্টেমে এগুলো কীভাবে কাজ করে?
A:ACID properties নিশ্চিত করে যে হাই-কনকারেন্সি ট্রেন বুকিং সিস্টেমে ডেটাবেজ ট্রানজেকশনগুলো অত্যন্ত নির্ভরযোগ্যভাবে সম্পন্ন হবে। নিচে আমাদের ৪টি সিট বুকিং ফ্লো-এর ক্ষেত্রে এগুলোর প্রয়োগ ব্যাখ্যা করা হলো:

#### Atomicity (অ্যাটমিসিটি - সব হবে, না হলে কিছুই হবে না):

ব্যাখ্যা: ইউজার যখন একসাথে ৪টি সিট (A1, A2, A3, A4) বুক করার চেষ্টা করে, তখন হয় চারটি সিটই একসঙ্গে রিজার্ভ হবে, অথবা একটি সিটও রিজার্ভ হবে না। যদি ৪ নম্বর সিটটি (A4) ইতিমধ্যে অন্য কেউ নিয়ে নেয়, তবে বাকি তিনটি সিটের হোল্ডও বাতিল হয়ে যাবে এবং পুরো ট্রানজেকশন ROLLBACK হয়ে যাবে।

ইন্টারভিউ অ্যানসার: "Atomicity ensures that all four seats are reserved together, or none of them are reserved."

#### Consistency (কনসিস্টেন্সি - ডেটাবেজ রুলস ঠিক রাখা):

ব্যাখ্যা: প্রতিটি ট্রানজেকশনের আগে ও পরে ডেটাবেজের সমস্ত কনস্ট্রেইন্ট এবং বিজনেস রুলস বজায় থাকতে হবে। যেমন—একই ট্রিপের একই সিট একাধিক active booking-এ থাকতে পারবে না, বুকিংয়ে ঠিক চারটি সিট থাকতে হবে, টোটাল ফেয়ার সার্ভার সাইড থেকে হিসাব হতে হবে এবং এক্সপায়ার্ড বুকিংয়ের সিট সঠিকভাবে রিলিজ হতে হবে।

ইন্টারভিউ অ্যানসার: "Consistency ensures that every successful transaction preserves database constraints and booking business rules."

#### Isolation (আইসোলেশন - কনকারেন্ট রিকোয়েস্ট নিরাপদ রাখা):

ব্যাখ্যা: User A এবং User B যদি একই সময়ে একই সিট বুক করতে চায়, তবে আইসোলেশন নিশ্চিত করবে যে কেবল একজনই সফল হতে পারবে। এর জন্য আমরা ট্রানজেকশনের ভেতরে রো-লেভেল লকিং (SELECT ... FOR UPDATE) ব্যবহার করি।

ইন্টারভিউ অ্যানসার: "Isolation prevents concurrent users from successfully reserving the same seat. I would use row-level locking or atomic conditional updates inside a transaction."

#### Durability (ডিউরেবিলিটি - কমিট করা ডেটা স্থায়ী থাকা):

ব্যাখ্যা: ট্রানজেকশন সফলভাবে COMMIT হওয়ার পর অ্যাপ্লিকেশন ক্র্যাশ করলেও বা রিস্টার্ট হলেও কমিট হওয়া বুকিং এবং সিটের স্ট্যাটাস ডেটাবেজে সুরক্ষিত থাকে। তবে মনে রাখতে হবে যে ডেটাবেজ কমিট এবং থার্ড-পার্টি পেমেন্ট গেটওয়ের স্টেট এক নয়, তাই পেমেন্ট রিকনসিলিয়েশন ও ওয়েবহুক রিট্রাই হ্যান্ডেল করতে হয়।

ইন্টারভিউ অ্যানসার: "Durability ensures that confirmed bookings and their committed database changes survive application crashes."

### 13.5 ACID ও Expiry-এর সম্পর্ক

  -----------------------------------------------------------------------
  ACID property                       Expiry mechanism-এ ভূমিকা
  ----------------------------------- -----------------------------------
  Atomicity                           Booking expire এবং চারটি seat
                                      release একসঙ্গে হবে।

  Consistency                         কোনো seat ভুলভাবে booked বা
                                      unavailable থাকবে না।

  Isolation                           Payment webhook ও expiry worker-এর
                                      conflicting update নিয়ন্ত্রণ করা হবে।

  Durability                          Confirmation বা expiry commit হওয়ার
                                      পর state স্থায়ী থাকবে।
  -----------------------------------------------------------------------

### 13.6 Expiry বনাম Payment Race Condition

ঠিক `10:10 AM`-এ expiry worker booking expire করতে চাইতে পারে, একই সময়ে
payment webhook success জানাতে পারে।

দুটি process-ই transaction-এর মধ্যে booking row lock করে state যাচাই করবে।
Lock পাওয়ার পরে প্রতিটি process বর্তমান state পুনরায় পরীক্ষা করবে এবং বৈধ
transition-ই করবে।

আগে থেকেই policy define করতে হবে:

-   Deadline-এর আগে সফল payment কীভাবে confirm হবে?
-   Deadline-এর পরে success notification এলে প্রকৃত payment time কীভাবে
    যাচাই হবে?
-   Seat ইতিমধ্যে অন্য user-কে দেওয়া হলে refund/reconciliation কী হবে?

শুধু lock থাকলেই business policy নির্ধারিত হয় না। Gateway-এর
timestamp/event semantics যাচাই করে state transition enforce করতে হবে।

> **গুরুত্বপূর্ণ:** ACID database transaction-এর correctness নিশ্চিত করে; এটি
> database ও external payment gateway-কে একটিমাত্র ACID transaction বানায়
> না।

## 14. Security ও Reliability

-   **Authentication:** Secure session বা JWT-based authentication
    ব্যবহার করা।
-   **Authorization:** User শুধু নিজের booking দেখতে বা cancel করতে
    পারবেন।
-   **Input validation:** Trip ID, seat ID, seat count ও payment request
    validate করা।
-   **Server-side pricing:** Client-এর পাঠানো total amount বিশ্বাস না
    করা।
-   **Payment security:** Gateway signature verify করা; frontend success
    page-কে payment proof ধরা যাবে না।
-   **Idempotency:** Retry বা duplicate webhook যেন duplicate
    charge/confirmation না ঘটায়।
-   **Database integrity:** Foreign key, unique constraint, transaction
    ও index ব্যবহার করা।
-   **Observability:** Booking state transition, payment event, expiry
    failure ও reconciliation error log/monitor করা।
-   **Sensitive data:** Log-এ payment secret বা sensitive payment
    information প্রকাশ না করা।

## 15. Scalability ও Performance

হাজার হাজার user একসঙ্গে seat book করতে চাইলে:

1.  **Stateless backend:** Load balancer-এর পেছনে একাধিক API instance
    চালানো।
2.  **Database indexing:** `trip_id`, `status`, `expires_at` ও booking
    ownership-এ query-উপযোগী index রাখা।
3.  **Safe concurrency:** Atomic seat allocation, database transaction ও
    constraint ব্যবহার করা।
4.  **Background processing:** Expiry, notification ও refund
    reconciliation worker-এ চালানো।
5.  **Caching:** Train schedule-এর মতো read-heavy data cache করা যায়;
    reservation-এর আগে database-এ availability validate করতেই হবে।
6.  **Monitoring:** Booking conflict, payment failure, expired hold,
    worker delay ও database load track করা।

প্রথম version-এ modular monolith + PostgreSQL + background worker যথেষ্ট।
প্রয়োজন প্রমাণিত হলে microservices বিবেচনা করা যেতে পারে।

## 18. Interview-এ বলার মতো সংক্ষিপ্ত উত্তর

> আমি Train Ticket Booking System-টি Node.js, Express.js এবং PostgreSQL
> দিয়ে design করব।
>
> Database-এর প্রধান table হবে `Users`, `Trains`, `Trips`, `TripSeats`,
> `Bookings`, `BookingSeats`, `Payments` এবং `Tickets`। একটি booking-এর সঙ্গে
> seat যুক্ত করতে `BookingSeats` relationship ব্যবহার করব।
>
> প্রথমে user Dhaka থেকে Jashore-এর train খুঁজে চারটি seat select করবেন।
> Backend database transaction-এর মধ্যে সব seat validate করে atomically
> hold করবে এবং `PENDING_PAYMENT` booking তৈরি করে ১০ মিনিটের
> `expires_at` timestamp রাখবে।
>
> এরপর backend payment gateway-তে payment initiate করবে। Payment
> verified হলে এবং reservation policy অনুযায়ী valid থাকলে booking
> `CONFIRMED` হবে, চারটি seat `BOOKED` হবে এবং ticket issue করা হবে।
>
> Deadline-এর মধ্যে payment না হলে background worker payment status যাচাই
> করে booking `EXPIRED` করবে এবং transaction-এর মাধ্যমে চারটি seat
> release করবে।
>
> সবচেয়ে গুরুত্বপূর্ণ challenge হলো double booking, concurrent request এবং
> delayed payment webhook। এগুলো handle করতে PostgreSQL row-level locking
> বা atomic conditional update, transaction, database constraint,
> idempotent webhook processing এবং expiry worker ব্যবহার করব।
>
> পাশাপাশি authentication, authorization, input validation, logging,
> monitoring এবং concurrency/payment failure tests implement করব। শুরুতে
> modular monolith রাখব; traffic ও business need বাড়লে architecture
> scale করব।
