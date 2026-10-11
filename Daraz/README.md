# 🛒 E-commerce System Design --- Daraz-এর মতো প্ল্যাটফর্ম

> **সমস্যার ধরন:** Easy--Medium\
> **মূল ফোকাস:** Inventory Management, ACID Transaction, Checkout,
> Payment, Concurrency Control\
> **ভাষা:** বাংলা\
> **উদ্দেশ্য:** Daraz-এর মতো একটি E-commerce Platform-এর scalable,
> reliable এবং concurrency-safe system design তৈরি করা।

------------------------------------------------------------------------

## সূচিপত্র

1.  [Problem Statement](#1-problem-statement)
2.  [Requirements](#2-requirements)
3.  [High-Level Architecture](#3-high-level-architecture)
4.  [Database Design](#4-database-design)
5.  [Status Management](#5-status-management)
6.  [ER Diagram](#6-er-diagram)
7.  [REST API Design](#7-rest-api-design)
8.  [Complete Checkout Flow](#8-complete-checkout-flow)
9.  [Double Purchase Prevention](#9-double-purchase-prevention)
10. [১০ মিনিটের Stock Reservation ও Expiry
    Mechanism](#10-১০-মিনিটের-stock-reservation-ও-expiry-mechanism)
11. [Payment Failure ও Delayed
    Webhook](#11-payment-failure-ও-delayed-webhook)
12. [Database Transaction](#12-database-transaction)
13. [ACID Properties](#13-acid-properties)
14. [Caching, Security ও Error
    Handling](#14-caching-security-ও-error-handling)
15. [Edge Cases ও সমাধান](#15-edge-cases-ও-সমাধান)
16. [Scalability ও Reliability](#16-scalability-ও-reliability)
17. [Monitoring ও Logging](#17-monitoring-ও-logging)
18. [Interview-এ বলার মতো সংক্ষিপ্ত
    উত্তর](#18-interview-এ-বলার-মতো-সংক্ষিপ্ত-উত্তর)
19. [Final Interview Checklist](#19-final-interview-checklist)

------------------------------------------------------------------------

## 1. Problem Statement

আমাদের একটি E-commerce Platform ডিজাইন করতে হবে, যা Daraz-এর মতো গ্রাহককে
পণ্য খোঁজা, কার্টে পণ্য যোগ করা, অর্ডার করা, অনলাইনে পেমেন্ট করা এবং ডেলিভারি
ট্র্যাক করার সুবিধা দেবে।

Seller পণ্যের তথ্য, দাম এবং stock পরিচালনা করতে পারবে। একই সময়ে অনেক
customer একই product কিনতে চাইলে সিস্টেম নিশ্চিত করবে যেন available
stock-এর চেয়ে বেশি পণ্য বিক্রি না হয়।

সিস্টেমকে আরও নিশ্চিত করতে হবে:

-   একই checkout request বারবার এলেও duplicate order তৈরি হবে না।
-   Payment সফল হলেও server crash করলে order/payment state পরে reconcile
    করা যাবে।
-   Payment ব্যর্থ বা reservation expire হলে stock সঠিকভাবে release হবে।
-   Coupon-এর redemption limit অতিক্রম হবে না।
-   Cart-এ যোগ করার পর product price বদলালেও checkout-এ backend-এর
    বর্তমান price যাচাই হবে।

### প্রধান design principle

**Inventory এবং order-এর critical update relational database
transaction-এর মধ্যে করব।** External payment gateway-কে database
transaction-এর অংশ করা যায় না; তাই idempotency, webhook, reconciliation
এবং reservation expiry ব্যবহার করব।

------------------------------------------------------------------------

## 2. Requirements

### 2.1 Functional Requirements

1.  Customer registration, login এবং profile management।
2.  Product browse, search, filter ও details দেখা।
3.  Cart-এ product যোগ, quantity update এবং remove করা।
4.  একাধিক product নিয়ে order তৈরি করা।
5.  Coupon validate ও apply করা।
6.  External payment gateway-এর মাধ্যমে payment করা।
7.  Seller-এর product listing এবং price management।
8.  Inventory quantity পরিচালনা করা।
9.  Order history এবং delivery status দেখা।
10. Eligible order cancel করা এবং refund process করা।

### 2.2 Non-functional Requirements

  -----------------------------------------------------------------------
  Requirement                         Design approach
  ----------------------------------- -----------------------------------
  Correctness                         DB constraints, transactions,
                                      row-level locking/conditional
                                      update

  Availability                        Stateless API instances, health
                                      checks, retry policy

  Scalability                         CDN, cache, horizontal API scaling,
                                      read replicas when needed

  Security                            Authentication, authorization,
                                      input validation, rate limiting

  Reliability                         Idempotency, durable payment
                                      records, webhook retry,
                                      reconciliation

  Performance                         Indexed queries, pagination,
                                      cacheable product reads

  Observability                       Structured logs, metrics, tracing,
                                      alerts
  -----------------------------------------------------------------------

### 2.3 Assumptions

-   PostgreSQL-কে primary transactional database হিসেবে ধরা হয়েছে।
-   একটি product-এর stock একটি inventory record-এ রাখা হবে।
    Multi-warehouse হলে `warehouse_id` যোগ করা যাবে।
-   Stock reservation-এর মেয়াদ উদাহরণ হিসেবে ১০ মিনিট; production-এ
    payment provider ও business rule অনুযায়ী configure করতে হবে।
-   Payment provider hosted checkout বা payment intent API দেয় এবং
    signed webhook পাঠায়।
-   টাকা floating-point হিসেবে নয়, integer minor unit-এ (যেমন পয়সা)
    সংরক্ষণ করা হবে।

------------------------------------------------------------------------

## 3. High-Level Architecture

### 3.1 Architecture Diagram

``` mermaid
flowchart TD
    C[Customer Web / Mobile App] --> CDN[CDN / WAF]
    S[Seller Dashboard] --> CDN
    CDN --> API[API Gateway / Load Balancer]
    API --> AUTH[Auth & Authorization]
    API --> CAT[Product Catalog API]
    API --> CART[Cart API]
    API --> ORD[Order / Checkout API]
    API --> INV[Inventory Module]
    API --> PAY[Payment Module]
    API --> SHIP[Delivery / Order Tracking]

    CAT --> CACHE[(Redis Cache)]
    CAT --> DB[(PostgreSQL)]
    CART --> DB
    ORD --> DB
    INV --> DB
    PAY --> DB
    SHIP --> DB

    PAY <--> PG[External Payment Gateway]
    PG --> WH[Payment Webhook Endpoint]
    WH --> PAY

    ORD --> OUTBOX[(Transactional Outbox)]
    OUTBOX --> MQ[Message Queue]
    MQ --> NOTIF[Notification Worker]
    MQ --> SEARCH[Search Index Worker]
    MQ --> ANALYTICS[Analytics Worker]
```

### 3.2 Component-এর দায়িত্ব

-   **Web/Mobile Client:** UI, product browsing, cart এবং checkout শুরু
    করা।
-   **CDN/WAF:** Static asset delivery, basic attack filtering এবং rate
    limiting-এর সহায়তা।
-   **API Gateway/Load Balancer:** Request routing, TLS termination এবং
    traffic distribution।
-   **Auth Module:** Identity যাচাই ও role/ownership authorization।
-   **Product Catalog:** Product information, category, price ও search।
-   **Cart Module:** Customer-এর cart ও quantity সংরক্ষণ।
-   **Order/Checkout:** Price validation, coupon validation, reservation
    এবং order creation orchestration।
-   **Inventory Module:** Stock reserve, confirm এবং release।
-   **Payment Module:** Gateway-এ payment initiation, webhook
    verification, payment state management ও reconciliation।
-   **PostgreSQL:** Order, inventory, payment ও coupon-এর authoritative
    transactional data।
-   **Redis:** Product/catalog-এর মতো cacheable data; stock-এর
    authoritative source নয়।
-   **Message Queue + Workers:** Email/SMS, search indexing এবং
    analytics-এর মতো asynchronous কাজ।
-   **Transactional Outbox:** DB commit-এর সঙ্গে event record-ও durable
    করে, যাতে commit-এর পর event publish করতে ভুলে না যাই।

### 3.3 Monolith নাকি Microservices?

প্রথম version-এর জন্য **modular monolith** একটি সহজ ও বাস্তবসম্মত পছন্দ: একটি
backend application, আলাদা domain modules এবং একটি PostgreSQL database।
Traffic, team size ও deployment needs বাড়লে Catalog, Order, Inventory,
Payment-কে আলাদা service করা যায়।

Inventory reservation এবং order creation-এর মতো strongly consistent
operation-গুলোকে অতিরিক্ত network call-এ ভাগ করলে complexity বাড়ে। তাই শুধু
"microservices" ব্যবহারের জন্য service split না করে domain boundary ও
operational need দেখে split করব।

------------------------------------------------------------------------

## 4. Database Design

নিচের schema একটি relational design-এর ধারণা দেয়। উদাহরণগুলো
PostgreSQL-এর জন্য; production migration-এ exact types, constraints এবং
business policy নির্ধারণ করতে হবে।

### 4.1 `users`

  Column            Type           Rule
  ----------------- -------------- --------------------------------------
  `id`              UUID           Primary key
  `name`            VARCHAR(120)   NOT NULL
  `email`           VARCHAR(255)   NOT NULL, UNIQUE
  `password_hash`   TEXT           NOT NULL; কখনো plaintext নয়
  `role`            VARCHAR(20)    CHECK: `CUSTOMER`, `SELLER`, `ADMIN`
  `created_at`      TIMESTAMPTZ    NOT NULL

**Index:** `UNIQUE(email)`। Email case-insensitive রাখতে প্রয়োজন হলে
normalized email বা `citext` ব্যবহার করা যায়।

### 4.2 `products`

  Column          Type           Rule
  --------------- -------------- -------------------------------
  `id`            UUID           Primary key
  `seller_id`     UUID           FK → `users.id`
  `name`          VARCHAR(255)   NOT NULL
  `description`   TEXT           Optional
  `price_minor`   BIGINT         CHECK `>= 0`
  `currency`      CHAR(3)        NOT NULL, যেমন `BDT`
  `status`        VARCHAR(20)    `DRAFT`, `ACTIVE`, `INACTIVE`
  `created_at`    TIMESTAMPTZ    NOT NULL
  `updated_at`    TIMESTAMPTZ    NOT NULL

**Indexes:** `(status, created_at)`; seller-এর listing-এর জন্য
`(seller_id, status)`। Search-এর জন্য প্রয়োজন অনুযায়ী PostgreSQL full-text
search বা dedicated search engine ব্যবহার করা যায়।

### 4.3 `inventory`

  Column            Type          Rule
  ----------------- ------------- ---------------------------
  `product_id`      UUID          PK এবং FK → `products.id`
  `available_qty`   INTEGER       CHECK `>= 0`
  `reserved_qty`    INTEGER       CHECK `>= 0`
  `updated_at`      TIMESTAMPTZ   NOT NULL

এখানে `available_qty` হলো নতুন order-এর জন্য reserve করা যেতে পারে এমন
stock; `reserved_qty` হলো active reservation-এ আটকে থাকা stock।
Alternative design-এ `on_hand_qty` ও `reserved_qty` রেখে
`available = on_hand - reserved` গণনা করা যায়। দুই পদ্ধতি একসঙ্গে মিশিয়ে
inconsistent calculation করা যাবে না।

**Invariant:** `available_qty >= 0` এবং `reserved_qty >= 0`। Stock
receipt, reservation, sale ও release---সব operation একই transaction-safe
নিয়মে করতে হবে।

### 4.4 `carts` ও `cart_items`

`carts`:

  Column         Type          Rule
  -------------- ------------- ------------------------------------
  `id`           UUID          PK
  `user_id`      UUID          FK → `users.id`
  `status`       VARCHAR(20)   `ACTIVE`, `CONVERTED`, `ABANDONED`
  `updated_at`   TIMESTAMPTZ   NOT NULL

`cart_items`:

  Column         Type          Rule
  -------------- ------------- --------------------
  `cart_id`      UUID          FK → `carts.id`
  `product_id`   UUID          FK → `products.id`
  `quantity`     INTEGER       CHECK `> 0`
  `added_at`     TIMESTAMPTZ   NOT NULL

**Constraint:** `PRIMARY KEY (cart_id, product_id)`। Cart-এর price-কে
authoritative ধরে checkout করা যাবে না; checkout-এ backend product price
পুনরায় পড়বে।

### 4.5 `orders`

  Column              Type           Rule
  ------------------- -------------- ---------------------------
  `id`                UUID           PK
  `user_id`           UUID           FK → `users.id`
  `status`            VARCHAR(30)    Order state
  `subtotal_minor`    BIGINT         CHECK `>= 0`
  `discount_minor`    BIGINT         CHECK `>= 0`
  `shipping_minor`    BIGINT         CHECK `>= 0`
  `total_minor`       BIGINT         CHECK `>= 0`
  `currency`          CHAR(3)        NOT NULL
  `idempotency_key`   VARCHAR(128)   Checkout retry protection
  `created_at`        TIMESTAMPTZ    NOT NULL
  `updated_at`        TIMESTAMPTZ    NOT NULL

**Constraint:** `(user_id, idempotency_key)` UNIQUE। Key-এর request
payload hash-ও সংরক্ষণ করা যেতে পারে; একই key দিয়ে ভিন্ন payload এলে
`409 Conflict` ফেরত দিতে হবে।

### 4.6 `order_items`

  Column                    Type           Rule
  ------------------------- -------------- -------------------------
  `id`                      UUID           PK
  `order_id`                UUID           FK → `orders.id`
  `product_id`              UUID           FK → `products.id`
  `product_name_snapshot`   VARCHAR(255)   Order-time product name
  `unit_price_minor`        BIGINT         CHECK `>= 0`
  `quantity`                INTEGER        CHECK `> 0`
  `line_total_minor`        BIGINT         CHECK `>= 0`

Order item-এ product name ও unit price snapshot রাখা জরুরি; ভবিষ্যতে
product-এর নাম বা দাম বদলালেও পুরোনো order-এর হিসাব বদলাবে না।

### 4.7 `payments`

  Column                  Type           Rule
  ----------------------- -------------- --------------------------------
  `id`                    UUID           PK
  `order_id`              UUID           FK → `orders.id`
  `provider`              VARCHAR(40)    Gateway name
  `provider_payment_id`   VARCHAR(255)   Provider reference
  `amount_minor`          BIGINT         CHECK `> 0`
  `currency`              CHAR(3)        NOT NULL
  `status`                VARCHAR(20)    Payment state
  `idempotency_key`       VARCHAR(128)   Provider request deduplication
  `created_at`            TIMESTAMPTZ    NOT NULL
  `updated_at`            TIMESTAMPTZ    NOT NULL

**Indexes/constraints:** `UNIQUE(provider, provider_payment_id)` যখন
provider ID উপস্থিত; `UNIQUE(idempotency_key)` যদি key global-unique হয়,
নইলে `(provider, idempotency_key)`। `order_id`-এ index।

### 4.8 `coupons` ও `coupon_redemptions`

`coupons`-এ `code` UNIQUE, `discount_type`, `discount_value`,
`max_redemptions`, `redeemed_count`, `starts_at`, `expires_at`,
`min_order_minor`, `status` থাকবে। `max_redemptions > 0`,
`redeemed_count >= 0` এবং `redeemed_count <= max_redemptions` বজায় রাখতে
হবে।

`coupon_redemptions`:

-   `id` UUID PK
-   `coupon_id` FK → `coupons.id`
-   `user_id` FK → `users.id`
-   `order_id` FK → `orders.id`
-   `status` (`RESERVED`, `REDEEMED`, `RELEASED`)
-   `created_at`, `updated_at`

**Constraint:** প্রয়োজনীয় policy অনুযায়ী `UNIQUE(coupon_id, user_id)` (এক
user একবার ব্যবহার করলে) অথবা `UNIQUE(order_id)` (এক order-এ একটি coupon
redemption)।

`redeemed_count` কেবল paid/confirmed redemption গুনবে নাকি active
reservation-ও গুনবে---এটি business rule। Reservation count করলে আলাদা
reserved counter বা redemption row-এর transactional accounting দরকার।

### 4.9 `stock_reservations` ও `stock_reservation_items`

`stock_reservations`:

-   `id` UUID PK
-   `order_id` FK → `orders.id`
-   `status` (`ACTIVE`, `CONFIRMED`, `RELEASED`, `EXPIRED`)
-   `expires_at` TIMESTAMPTZ NOT NULL
-   `created_at`, `updated_at`

`stock_reservation_items`:

-   `reservation_id` FK → `stock_reservations.id`
-   `product_id` FK → `products.id`
-   `quantity` INTEGER CHECK `> 0`
-   Composite PK: `(reservation_id, product_id)`

**Indexes:** `(status, expires_at)` expiry worker-এর জন্য; `order_id`-এ
index। এক order-এর জন্য একটি reservation চাইলে `UNIQUE(order_id)`।

### 4.10 অতিরিক্ত operational tables

-   `payment_webhook_events`: `provider`, `event_id`, `received_at`,
    `processed_at`, `status`, `payload_hash`;
    `UNIQUE(provider, event_id)` দিয়ে duplicate webhook আটকানো।
-   `idempotency_records`: `scope`, `key`, `request_hash`,
    `response_status`, `response_body`, `created_at`;
    `UNIQUE(scope, key)`।
-   `outbox_events`: `id`, `aggregate_type`, `aggregate_id`,
    `event_type`, `payload`, `created_at`, `published_at`; unpublished
    event-এর জন্য index।
-   `shipments`: order reference, carrier, tracking number, delivery
    status ও timestamps।

### 4.11 গুরুত্বপূর্ণ index-এর সারাংশ

  ----------------------------------------------------------------------------------
  Query                               Index
  ----------------------------------- ----------------------------------------------
  Customer-এর order history           `orders(user_id, created_at DESC)`

  Order items                         `order_items(order_id)`

  Product-এর seller listing           `products(seller_id, status)`

  Active reservation expiry           `stock_reservations(status, expires_at)`

  Payment lookup                      `payments(order_id)`, unique provider
                                      reference

  Webhook duplicate check             `payment_webhook_events(provider, event_id)`

  Coupon code lookup                  `coupons(code)` UNIQUE
  ----------------------------------------------------------------------------------

Index query দ্রুত করে, তবে write overhead বাড়ায়। Query plan ও production
workload দেখে index নির্বাচন করতে হবে।

------------------------------------------------------------------------

## 5. Status Management

Status transition-গুলো explicit রাখতে হবে; যেকোনো status থেকে যেকোনো
status-এ update অনুমোদন করা যাবে না।

### 5.1 Order Status

``` mermaid
stateDiagram-v2
    [*] --> PENDING_PAYMENT
    PENDING_PAYMENT --> CONFIRMED: Payment verified
    PENDING_PAYMENT --> PAYMENT_FAILED: Payment failed
    PENDING_PAYMENT --> EXPIRED: Reservation expired
    PENDING_PAYMENT --> CANCELLED: Eligible cancellation
    CONFIRMED --> PROCESSING
    PROCESSING --> SHIPPED
    SHIPPED --> DELIVERED
    CONFIRMED --> CANCELLED: Policy permits
    DELIVERED --> REFUND_PENDING: Refund approved
    REFUND_PENDING --> REFUNDED: Refund confirmed
```

-   `PENDING_PAYMENT`: Order তৈরি হয়েছে, payment এখনো নিশ্চিত নয়।
-   `CONFIRMED`: Verified successful payment-এর পর order নিশ্চিত।
-   `PAYMENT_FAILED`: Payment ব্যর্থ; reservation release করার workflow
    চলবে।
-   `EXPIRED`: Payment সময়মতো নিশ্চিত হয়নি এবং reservation expire হয়েছে।
-   `CANCELLED`: Business policy অনুযায়ী order বাতিল।
-   `PROCESSING`, `SHIPPED`, `DELIVERED`: Fulfillment-এর ধাপ।
-   `REFUND_PENDING`, `REFUNDED`: Refund process-এর ধাপ।

### 5.2 Payment Status

`CREATED → PENDING → SUCCEEDED | FAILED | CANCELLED`; refund হলে
`REFUND_PENDING → REFUNDED` হতে পারে। Payment gateway-এর response বা
signed webhook verify না করে `SUCCEEDED` করা যাবে না।

### 5.3 Reservation Status

`ACTIVE → CONFIRMED` অথবা `ACTIVE → RELEASED / EXPIRED`। একই reservation
একাধিকবার release হলেও stock দ্বিতীয়বার ফেরত দেওয়া যাবে না। এজন্য
conditional status transition এবং inventory update একই DB transaction-এ
হবে।

### 5.4 Delivery Status

`PENDING → PACKED → SHIPPED → OUT_FOR_DELIVERY → DELIVERED`। Carrier-এর
event duplicate বা out-of-order আসতে পারে, তাই status transition
validation দরকার।

------------------------------------------------------------------------

## 6. ER Diagram

``` mermaid
erDiagram
    USERS ||--o{ PRODUCTS : sells
    USERS ||--o{ CARTS : owns
    CARTS ||--o{ CART_ITEMS : contains
    PRODUCTS ||--o{ CART_ITEMS : added_to
    PRODUCTS ||--|| INVENTORY : has
    USERS ||--o{ ORDERS : places
    ORDERS ||--|{ ORDER_ITEMS : contains
    PRODUCTS ||--o{ ORDER_ITEMS : referenced_by
    ORDERS ||--o{ PAYMENTS : has
    ORDERS ||--o| STOCK_RESERVATIONS : reserves
    STOCK_RESERVATIONS ||--|{ STOCK_RESERVATION_ITEMS : includes
    PRODUCTS ||--o{ STOCK_RESERVATION_ITEMS : reserved
    COUPONS ||--o{ COUPON_REDEMPTIONS : tracks
    USERS ||--o{ COUPON_REDEMPTIONS : redeems
    ORDERS ||--o| COUPON_REDEMPTIONS : applies
    ORDERS ||--o| SHIPMENTS : ships
```

**Note:** বাস্তব schema-তে seller এবং customer একই `users` table-এ
role-ভিত্তিক হতে পারে। Marketplace-এ একটি order একাধিক seller-এর product
ধারণ করলে seller-wise sub-order/fulfillment model লাগতে পারে।

------------------------------------------------------------------------

## 7. REST API Design

সব endpoint HTTPS ব্যবহার করবে। Protected endpoint-এ authentication এবং
resource ownership/role authorization লাগবে। Request/response উদাহরণ
সংক্ষিপ্তভাবে দেখানো হয়েছে।

### 7.1 Authentication ও Profile

  Method    Endpoint                  কাজ
  --------- ------------------------- ----------------
  `POST`    `/api/v1/auth/register`   Account তৈরি
  `POST`    `/api/v1/auth/login`      Login
  `POST`    `/api/v1/auth/refresh`    Token refresh
  `GET`     `/api/v1/users/me`        নিজের profile
  `PATCH`   `/api/v1/users/me`        Profile update

### 7.2 Product ও Inventory

  -----------------------------------------------------------------------------------------------------------
  Method                  Endpoint                                                    কাজ
  ----------------------- ----------------------------------------------------------- -----------------------
  `GET`                   `/api/v1/products?q=&category=&minPrice=&maxPrice=&page=`   Search/filter

  `GET`                   `/api/v1/products/:productId`                               Product details

  `POST`                  `/api/v1/seller/products`                                   Seller product তৈরি

  `PATCH`                 `/api/v1/seller/products/:productId`                        নিজের product update

  `PUT`                   `/api/v1/seller/products/:productId/inventory`              Stock adjust; audit
                                                                                      log-সহ
  -----------------------------------------------------------------------------------------------------------

Seller stock update-এ client থেকে সরাসরি arbitrary `available_qty`
overwrite না করে `adjustment`/stock movement এবং authorization ব্যবহার
করা নিরাপদ।

### 7.3 Cart

  Method     Endpoint                          কাজ
  ---------- --------------------------------- -----------------
  `GET`      `/api/v1/cart`                    নিজের cart
  `POST`     `/api/v1/cart/items`              Product যোগ
  `PATCH`    `/api/v1/cart/items/:productId`   Quantity update
  `DELETE`   `/api/v1/cart/items/:productId`   Item remove

উদাহরণ request:

``` json
{
  "productId": "product-uuid",
  "quantity": 2
}
```

Cart-এ quantity positive integer হতে হবে। Cart operation stock guarantee
দেয় না; stock checkout-এর সময় reserve হবে।

### 7.4 Checkout ও Order

  Method   Endpoint                           কাজ
  -------- ---------------------------------- ---------------------------
  `POST`   `/api/v1/checkout`                 Checkout ও reservation শুরু
  `GET`    `/api/v1/orders`                   নিজের order history
  `GET`    `/api/v1/orders/:orderId`          Order details
  `POST`   `/api/v1/orders/:orderId/cancel`   Eligible cancellation

Checkout request-এর `Idempotency-Key` header অবশ্যই পাঠাতে হবে:

``` http
POST /api/v1/checkout
Authorization: Bearer <token>
Idempotency-Key: checkout-unique-client-key
Content-Type: application/json
```

``` json
{
  "items": [
    { "productId": "product-uuid-1", "quantity": 2 },
    { "productId": "product-uuid-2", "quantity": 1 }
  ],
  "couponCode": "SAVE10",
  "shippingAddressId": "address-uuid"
}
```

Client-এর পাঠানো price/discount/total বিশ্বাস করা যাবে না। Backend
database থেকে বর্তমান price, eligibility, stock এবং coupon rules যাচাই
করে total গণনা করবে।

### 7.5 Payment ও Refund

  -------------------------------------------------------------------------------------------
  Method                  Endpoint                                    কাজ
  ----------------------- ------------------------------------------- -----------------------
  `POST`                  `/api/v1/orders/:orderId/payment-session`   Gateway payment session
                                                                      তৈরি

  `GET`                   `/api/v1/payments/:paymentId`               Payment status

  `POST`                  `/api/v1/webhooks/payments/:provider`       Gateway webhook

  `POST`                  `/api/v1/orders/:orderId/refund`            Authorized refund
                                                                      request
  -------------------------------------------------------------------------------------------

Webhook endpoint-এ provider signature, timestamp/replay protection এবং
event ID যাচাই করতে হবে। সাধারণ user-এর JWT দিয়ে webhook authenticate
করা যাবে না; provider signature verification ব্যবহার করতে হবে।

### 7.6 সাধারণ Error Response

``` json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "একটি বা একাধিক পণ্যের পর্যাপ্ত stock নেই।",
    "requestId": "req-unique-id"
  }
}
```

প্রস্তাবিত status code:

-   `400 Bad Request`: Invalid input।
-   `401 Unauthorized`: Login/token সমস্যা।
-   `403 Forbidden`: Role/ownership অনুমোদিত নয়।
-   `404 Not Found`: Resource নেই বা access policy অনুযায়ী অপ্রকাশিত।
-   `409 Conflict`: Stock conflict, idempotency-key payload mismatch বা
    invalid state transition।
-   `422 Unprocessable Entity`: Coupon/checkout business rule ব্যর্থ।
-   `429 Too Many Requests`: Rate limit অতিক্রম।
-   `500 Internal Server Error`: Unexpected server error।
-   `502/503`: Gateway/provider unavailable বা service সাময়িকভাবে
    unavailable।

------------------------------------------------------------------------

## 8. Complete Checkout Flow

``` mermaid
sequenceDiagram
    actor Customer
    participant API as Checkout API
    participant DB as PostgreSQL
    participant Pay as Payment Service
    participant PG as Payment Gateway
    participant WH as Webhook Handler

    Customer->>API: POST /checkout + Idempotency-Key
    API->>DB: Begin transaction
    API->>DB: Validate products, prices, coupon
    API->>DB: Atomically reserve stock
    API->>DB: Create order, items, reservation, outbox
    API->>DB: Commit
    API-->>Customer: Order PENDING_PAYMENT + orderId
    Customer->>Pay: Start payment session
    Pay->>PG: Create payment (provider idempotency key)
    PG-->>Pay: Payment URL / intent
    Pay-->>Customer: Payment URL / client secret
    Customer->>PG: Complete payment
    PG->>WH: Signed payment webhook
    WH->>DB: Verify event + transactionally update payment/order/reservation
    WH-->>PG: 2xx acknowledgement
```

### Step-by-step

1.  Customer checkout request পাঠায় এবং unique `Idempotency-Key` দেয়।
2.  Backend authenticated user, product status, quantity, current price,
    coupon এবং shipping rule যাচাই করে।
3.  DB transaction শুরু করে product ID order-এ consistent order-এ
    lock/update করে, যাতে deadlock risk কমে।
4.  প্রতিটি item-এর stock conditional update দিয়ে reserve করে। যেকোনো
    item reserve ব্যর্থ হলে পুরো transaction rollback হবে।
5.  Order, order items, reservation, coupon reservation এবং outbox event
    একই transaction-এ তৈরি হয়।
6.  Transaction commit হলে `PENDING_PAYMENT` order ও reservation expiry
    time client-কে ফেরত দেওয়া হয়।
7.  Payment service provider-এর idempotency key ব্যবহার করে payment
    session তৈরি করে। External API call দীর্ঘ DB transaction-এর ভেতরে
    রাখা হবে না।
8.  Gateway-এর verified webhook payment result জানায়। Webhook event
    duplicate হলেও একবারই apply হবে।
9.  Payment success হলে order confirm ও reservation confirm করা
    হবে---তবে reservation এখনও active কিনা এবং business policy কী বলে তা
    transaction-এ যাচাই করতে হবে।
10. Payment failed/expired হলে reservation release workflow চালু হবে।
11. Notification ও অন্যান্য side effect outbox/queue worker দিয়ে করা হবে।

### কেন payment call DB transaction-এর বাইরে?

Gateway-এর network latency বেশি হতে পারে। DB lock ধরে রেখে external call
করলে contention, timeout ও deadlock বাড়তে পারে। তাই প্রথমে stock/order
state durable করি, তারপর payment session তৈরি করি। Payment session তৈরির
মাঝখানে failure হলে retry/reconciliation দিয়ে recover করা যায়।

------------------------------------------------------------------------

## 9. Double Purchase Prevention

### Problem: শেষ ১টি product-এর জন্য ২ জন customer

ধরা যাক, stock `1`; Customer A ও Customer B একই সময়ে কিনতে চায়। যদি
দুজনেই আগে `SELECT available_qty` করে `1` দেখে এবং পরে আলাদাভাবে stock
কমায়, তাহলে overselling হতে পারে।

### Strategy A: Atomic Conditional Update

``` sql
UPDATE inventory
SET available_qty = available_qty - :qty,
    reserved_qty = reserved_qty + :qty,
    updated_at = NOW()
WHERE product_id = :product_id
  AND available_qty >= :qty
RETURNING product_id, available_qty, reserved_qty;
```

-   একটি row ফেরত এলে reservation সফল।
-   কোনো row ফেরত না এলে stock নেই বা পর্যাপ্ত নয়।
-   এই update-টি order/reservation creation-এর সঙ্গে একই transaction-এ
    থাকবে।
-   একাধিক product থাকলে সব item reserve করতে হবে; একটি item ব্যর্থ হলে
    পুরো transaction rollback হবে।

PostgreSQL একই row-তে concurrent update serialize করে এবং
`WHERE available_qty >= :qty` condition update-এর সময় পুনরায় যাচাই হয়।
ফলে একই final unit দুই customer reserve করতে পারবে না।

### Strategy B: `SELECT ... FOR UPDATE`

``` sql
BEGIN;

SELECT product_id, available_qty
FROM inventory
WHERE product_id = :product_id
FOR UPDATE;

-- Application checks available_qty >= requested quantity.

UPDATE inventory
SET available_qty = available_qty - :qty,
    reserved_qty = reserved_qty + :qty
WHERE product_id = :product_id;

COMMIT;
```

Row lock নেওয়ার পর stock check ও update একই transaction-এ করতে হবে।
সাধারণ ক্ষেত্রে conditional update কম round trip-এ কাজ করে; জটিল inventory
logic-এ row lock সুবিধাজনক হতে পারে।

### একাধিক product-এর lock ordering

একটি order-এ product A, B এবং অন্য order-এ B, A থাকলে deadlock হতে পারে।
সব checkout-এ product ID ascending order-এ process/lock করলে deadlock
risk কমে। তবুও deadlock বা serialization failure হলে সীমিত retry করতে
হবে।

### গুরুত্বপূর্ণ নিয়ম

-   Stock-এর source of truth PostgreSQL; Redis cache দিয়ে stock
    decrement সিদ্ধান্ত নেওয়া যাবে না।
-   `CHECK (available_qty >= 0)` অতিরিক্ত safety net, কিন্তু একে একমাত্র
    concurrency control ধরা যাবে না।
-   Stock update, reservation row এবং order creation একই transaction-এ
    commit হবে।
-   Seller stock adjustment-ও একই inventory concurrency rules মানবে।

------------------------------------------------------------------------

## 10. ১০ মিনিটের Stock Reservation ও Expiry Mechanism

### 10.1 Reservation কেন দরকার?

Customer payment page-এ থাকার সময় stock অন্য customer-এর কাছে বিক্রি হয়ে
যাওয়া ঠেকাতে অল্প সময়ের জন্য stock reserve করা হয়। তবে customer payment না
করলে stock অনির্দিষ্ট সময় আটকে রাখা যাবে না।

### 10.2 Reservation তৈরি

Checkout transaction-এর মধ্যে:

1.  `available_qty >= requested_qty` condition দিয়ে stock reserve।
2.  `stock_reservations` row `ACTIVE` হিসেবে তৈরি।
3.  `expires_at = NOW() + INTERVAL '10 minutes'` সেট।
4.  Reservation item quantity সংরক্ষণ।
5.  Order `PENDING_PAYMENT` হিসেবে তৈরি।
6.  সবকিছু একসঙ্গে commit।

### 10.3 Expiry Worker

Background worker প্রতি কিছু সময় পর expired reservation খুঁজবে। উদাহরণ
query:

``` sql
SELECT id
FROM stock_reservations
WHERE status = 'ACTIVE'
  AND expires_at <= NOW()
ORDER BY expires_at
FOR UPDATE SKIP LOCKED
LIMIT 100;
```

প্রতিটি reservation-এর জন্য transaction-এ:

1.  Reservation row lock/conditional transition করবে।
2.  `ACTIVE → EXPIRED` update সফল হয়েছে কি না নিশ্চিত করবে।
3.  সফল হলে reservation items-এর quantity inventory-তে ফেরত দেবে:
    `available_qty += qty`, `reserved_qty -= qty`।
4.  Order এখনও pending থাকলে policy অনুযায়ী `EXPIRED` করবে।
5.  Coupon reservation থাকলে release করবে।
6.  Transaction commit করবে।

**Idempotency:** Reservation status `ACTIVE` থেকে একবারই transition হতে
পারবে। Worker একই reservation পুনরায় দেখলেও stock দ্বিতীয়বার ফেরত দেবে না।

### 10.4 Expiry race

Expiry worker এবং payment webhook একই reservation update করতে পারে।
দুজনকেই reservation/order row-তে lock নিতে হবে এবং status transition
transactionally যাচাই করতে হবে। শুধু worker-এর আগে `SELECT` করা যথেষ্ট নয়।

------------------------------------------------------------------------

## 11. Payment Failure ও Delayed Webhook

External payment gateway ও PostgreSQL-এর মধ্যে একক ACID transaction নেই।
ফলে system-কে distributed failure মাথায় রেখে design করতে হবে।

### 11.1 Payment Failed

1.  Gateway webhook বা trusted status API দিয়ে failure verify।
2.  `payment: PENDING → FAILED` transition।
3.  Order এখনও `PENDING_PAYMENT` হলে business policy অনুযায়ী
    `PAYMENT_FAILED`।
4.  Reservation `ACTIVE` থাকলে stock release।
5.  Coupon reservation থাকলে release।
6.  একই webhook আবার এলে duplicate event ID/state transition-এর কারণে
    stock দ্বিতীয়বার release হবে না।

### 11.2 Payment সফল, কিন্তু server crash

উদাহরণ: Gateway টাকা কেটে নিয়েছে, কিন্তু webhook processing-এর আগে
application crash করেছে।

-   Gateway webhook retry করবে বলে ধরে নিতে হবে, কিন্তু শুধু তার ওপর নির্ভর
    করা উচিত নয়।
-   `payment_webhook_events`-এ event durable ভাবে record করে processing
    retry করতে হবে।
-   Scheduled reconciliation worker gateway API দিয়ে pending payment-এর
    status যাচাই করবে।
-   Verified success পাওয়া গেলে payment/order state recover করবে।
-   Refund প্রয়োজন হলে idempotent refund operation শুরু করবে।

### 11.3 Webhook processing-এর নিয়ম

1.  Provider signature ও timestamp যাচাই।
2.  `provider + event_id` unique constraint দিয়ে duplicate আটকানো।
3.  Event record durable করা।
4.  Order/payment reference, amount ও currency local record-এর সঙ্গে
    মিলানো।
5.  State transition এবং reservation handling একই DB transaction-এ করা।
6.  Transaction commit হওয়ার পর 2xx response দেওয়া।
7.  Transient failure হলে provider retry করতে পারে; internal worker-ও
    retry করবে।

Webhook response খুব দেরি করানো যাবে না। Payload log করার সময় card data,
secret বা sensitive personal information রাখা যাবে না।

### 11.4 Payment success webhook reservation expiry-এর পরে এলে

এটি একটি business policy decision; নীরবে stock negative করা যাবে না।

**প্রস্তাবিত policy:**

-   Reservation এখনও `ACTIVE` এবং expiry হয়নি: payment verify করে order
    confirm ও reservation confirm।
-   Reservation `EXPIRED` এবং stock অন্য order-এ reserve/sell হয়ে গেছে:
    নতুন stock জোর করে allocate নয়; order-কে reconciliation/manual-review
    state-এ নিন এবং automatic refund/void workflow শুরু করুন।
-   Reservation expire হয়েছে কিন্তু stock এখনও available: payment
    provider-এর verified success ও business policy অনুযায়ী নতুন atomic
    reservation করা যেতে পারে; তা সম্ভব না হলে refund/void।
-   Payment success এবং reservation expiry একসঙ্গে race করলে DB row lock
    এবং explicit state transition winner নির্ধারণ করবে।

Provider-এর success event দেরিতে আসতে পারে, তাই expiry মানেই payment
অবশ্যই ব্যর্থ---এমন ধরে নেওয়া যাবে না।

### 11.5 Payment reconciliation

একটি scheduled job নিয়মিত:

-   অনেকক্ষণ `PENDING` থাকা payment খুঁজবে।
-   Provider API থেকে status যাচাই করবে।
-   Local status update করবে।
-   Provider success কিন্তু local order pending হলে recovery করবে।
-   Local success কিন্তু provider status mismatch হলে alert তৈরি করবে।
-   Unresolved case support/manual review queue-তে পাঠাবে।

------------------------------------------------------------------------

## 12. Database Transaction

### 12.1 Checkout Transaction-এর pseudocode

``` text
function checkout(userId, items, couponCode, idempotencyKey):
    validate request format and positive quantities

    begin database transaction

      existing = find order by (userId, idempotencyKey)
      if existing exists:
          if request hash differs:
              return 409 Conflict
          return existing order

      load current products and prices
      validate products are active
      calculate subtotal using integer minor units
      validate coupon eligibility and redemption capacity

      for each product ordered by productId:
          atomically decrement available_qty only if enough stock
          increment reserved_qty
          if update affected zero rows:
              rollback and return INSUFFICIENT_STOCK

      create order with PENDING_PAYMENT status
      create order_items with price/name snapshots
      create ACTIVE stock reservation with expiry
      reserve coupon capacity if policy requires
      insert transactional outbox event

    commit transaction

    create/reuse payment session using provider idempotency key
    return order and payment-session details
```

**Implementation note:** Idempotency key-এর concurrent duplicate request
আটকাতে unique constraint থাকতে হবে। শুধু transaction শুরু করে key check করলে
race হতে পারে; unique constraint violation handle করে existing result
ফিরিয়ে দিতে হবে।

### 12.2 Atomicity

Stock reserve হয়েছে কিন্তু order create হয়নি---এমন partial result থাকা
উচিত নয়। Stock update, order, items এবং reservation একই transaction-এ
থাকবে। যেকোনো step ব্যর্থ হলে rollback হবে।

### 12.3 Payment-এর ক্ষেত্রে সীমাবদ্ধতা

External payment API database transaction-এর অংশ নয়। তাই checkout
transaction commit হওয়ার পর payment initiation হবে। Provider idempotency
key এবং local payment record দিয়ে repeated request নিরাপদ করা হবে।
Payment session তৈরি করতে ব্যর্থ হলে order pending থাকতে পারে এবং
retry/reconciliation হবে; reservation expire হলে release হবে।

### 12.4 Isolation Level

-   সাধারণ checkout-এ conditional stock update এবং unique constraints
    অনেক ক্ষেত্রে যথেষ্ট।
-   জটিল coupon cap বা cross-row business invariant-এর জন্য row
    lock/serializable transaction বিবেচনা করা যায়।
-   `SERIALIZABLE` ব্যবহার করলে serialization failure হতে পারে; তাই
    bounded retry এবং idempotent logic প্রয়োজন।
-   Transaction ছোট রাখব; এর ভেতরে external HTTP call, email বা SMS
    পাঠাব না।

------------------------------------------------------------------------

## 13. ACID Properties

  -----------------------------------------------------------------------
  Property                অর্থ                     E-commerce-এ প্রয়োগ
  ----------------------- ----------------------- -----------------------
  **Atomicity**           সব operation হবে, নয়তো  Stock reserve, order,
                          কিছুই হবে না             items ও reservation
                                                  একসঙ্গে commit/rollback

  **Consistency**         Transaction-এর আগে ও    Stock negative নয়,
                          পরে invariant ঠিক থাকবে quantity positive, FK
                                                  valid, coupon cap মানা

  **Isolation**           Concurrent transaction  Conditional update, row
                          একে অন্যের মাঝামাঝি      lock, unique
                          অবস্থা ব্যবহার করে ভুল     constraint, প্রয়োজনে
                          সিদ্ধান্ত নেবে না         serializable

  **Durability**          Commit হওয়া data        PostgreSQL durable
                          crash-এর পরও টিকে থাকবে commit, backup ও
                                                  recovery strategy
  -----------------------------------------------------------------------

### ACID নিয়ে গুরুত্বপূর্ণ পার্থক্য

ACID একটি database transaction-এর guarantee। এটি নিজে থেকে payment
gateway-এর সঙ্গে distributed atomic transaction তৈরি করে না। Payment
flow-তে durable state machine, idempotency, webhook retry, outbox এবং
reconciliation দিয়ে failure recovery করতে হবে।

------------------------------------------------------------------------

## 14. Caching, Security ও Error Handling

### 14.1 Caching

-   Product details/category listing-এর জন্য Redis বা CDN cache ব্যবহার
    করা যায়।
-   Cache key-তে product ID, locale বা filter parameters রাখতে হবে।
-   Product update হলে cache invalidation/event-driven refresh করতে হবে।
-   Search result কিছু সময় stale হতে পারে; checkout-এর price ও stock
    অবশ্যই database থেকে পুনরায় যাচাই হবে।
-   **Inventory availability cache কেবল indicative display-এর জন্য;
    reservation decision-এর source of truth নয়।**
-   Cart cache করা গেলেও cart-এর durable persistence ও recovery policy
    স্পষ্ট থাকতে হবে।

### 14.2 Security

-   Password plaintext নয়; Argon2id বা bcrypt-এর মতো password hashing।
-   HTTPS/TLS বাধ্যতামূলক।
-   Role-based access control এবং resource ownership check।
-   Seller শুধু নিজের product ও inventory update করতে পারবে।
-   Server-side input validation এবং parameterized SQL/ORM।
-   Rate limiting: login, coupon apply, checkout ও payment session
    endpoints।
-   Payment webhook signature verification ও replay/duplicate
    protection।
-   Payment card data নিজের database-এ সংরক্ষণ না করে
    gateway-hosted/tokenized flow ব্যবহার।
-   Secrets environment/secret manager-এ; source code বা logs-এ নয়।
-   Sensitive data redact করে structured logging।
-   CSRF protection প্রয়োজন হলে cookie-based auth policy অনুযায়ী প্রয়োগ;
    XSS protection ও secure cookie settings।
-   Admin/seller critical action-এর audit log।

### 14.3 Error Handling ও Retry

  -----------------------------------------------------------------------
  Error                               আচরণ
  ----------------------------------- -----------------------------------
  Insufficient stock                  `409`/business error; order
                                      transaction rollback

  Invalid/expired coupon              `422`; checkout fail বা policy
                                      অনুযায়ী coupon বাদ দিয়ে confirmation

  Same key, same payload              আগের result ফেরত

  Same key, different payload         `409 Conflict`

  Gateway timeout                     Payment status unknown/pending ধরে
                                      provider status reconcile

  DB deadlock/serialization failure   সীমিত exponential backoff-সহ retry

  Webhook duplicate                   আগে processed হলে 2xx; side effect
                                      পুনরায় নয়

  Notification failure                Order rollback নয়; queue retry ও
                                      dead-letter handling
  -----------------------------------------------------------------------

Client-কে internal stack trace বা database details দেখানো যাবে না।
প্রত্যেক response/log-এ `requestId` বা trace ID রাখা উপকারী।

------------------------------------------------------------------------

## 15. Edge Cases ও সমাধান

### Case 1: Stock 10, কিন্তু 100 customer একসঙ্গে কিনতে চায়

Atomic conditional update দিয়ে সর্বোচ্চ ১০টি unit reserve হবে---ধরে নেওয়া
হয়েছে প্রতিটি customer ১টি করে চাইছে এবং অন্য reservation/stock movement
নেই। Stock শেষ হলে বাকি request-গুলো `INSUFFICIENT_STOCK` পাবে। 100
request-এর মধ্যে ঠিক 10টি success হবে---শুধু quantity ও transaction outcome
সেই পরিস্থিতি নিশ্চিত করলে।

### Case 2: Payment সফল, কিন্তু order confirm করার আগে server crash

Webhook event/provider reference durable করে রাখব; provider retry এবং
reconciliation worker দিয়ে payment status আবার যাচাই করব। Verified
success হলে idempotent transaction দিয়ে order confirm করব। Reservation
ইতোমধ্যে expire হয়ে গেলে late-success policy অনুযায়ী fulfill বা
refund/void করব।

### Case 3: Network timeout-এর পর customer checkout retry করে

একই `Idempotency-Key` পুনরায় পাঠাতে হবে। Unique constraint ও request
hash-এর কারণে একই order result ফেরত আসবে, duplicate order তৈরি হবে না।
Client নতুন key দিয়ে একই action করলে backend সেটিকে আলাদা request হিসেবে
দেখতে পারে, তাই client retry-তে key reuse জরুরি।

### Case 4: Coupon limit 1,000; অনেক customer একসঙ্গে apply করে

Coupon capacity reserve করার সময় coupon row lock অথবা atomic conditional
update ব্যবহার করতে হবে:

``` sql
UPDATE coupons
SET reserved_count = reserved_count + 1
WHERE id = :coupon_id
  AND status = 'ACTIVE'
  AND redeemed_count + reserved_count < max_redemptions
RETURNING id;
```

এখানে `reserved_count` column যোগ করলে reservation
create/release/confirm সব একই transaction-এ maintain করতে হবে। Update
কোনো row না ফেরালে coupon capacity নেই। Coupon policy অনুযায়ী payment
confirm-এর সময় reservation `redeemed_count`-এ convert হবে;
expiry/failure-এ `reserved_count` কমবে।

### Case 5: Customer stock reserve করে payment করে না

Expiry worker reservation expire করবে, reserved stock release করবে এবং
pending order-কে `EXPIRED` করবে। Conditional status transition-এর কারণে
duplicate worker run stock দ্বিগুণ ফেরত দেবে না।

### Case 6: Reservation expire হওয়ার পরে payment webhook আসে

Webhook signature ও payment status verify করে current order/reservation
state lock করতে হবে। Stock available হলে policy অনুযায়ী নতুন atomic
reservation/fulfillment বিবেচনা করা যায়; stock না থাকলে order-কে fulfill
করার জন্য oversell নয়---refund/void বা manual review। সব সিদ্ধান্ত audit
log-সহ হবে।

### Case 7: Cart-এ যোগ করার পরে product price বদলে যায়

Cart-এর পুরোনো price authoritative নয়। Checkout-এ current price পড়ব এবং
সেই price snapshot `order_items`-এ রাখব। UI-তে price পরিবর্তন হলে
customer-কে নতুন total দেখিয়ে confirmation নেওয়া যেতে পারে; price lock
guarantee থাকলে সেটির expiry ও policy আলাদা করে design করতে হবে।

### অতিরিক্ত edge cases

-   **একই webhook বারবার আসে:** `UNIQUE(provider, event_id)` এবং
    idempotent transition।
-   **একটি multi-item order-এর একটি product reserve হয়, অন্যটি হয় না:**
    পুরো DB transaction rollback।
-   **Seller একই সময়ে stock update করে:** inventory row-এর concurrency
    control ও audited stock movement।
-   **Refund webhook duplicate:** provider refund ID unique এবং refund
    state transition idempotent।
-   **Queue message duplicate:** consumer idempotency key/event ID দিয়ে
    duplicate side effect আটকাবে।
-   **Deadlock:** consistent lock ordering এবং bounded retry।
-   **Product deactivate হয়ে যায় checkout-এর মাঝখানে:** checkout
    transaction-এ active status যাচাই; প্রয়োজনীয় consistency অনুযায়ী
    lock/version check।

------------------------------------------------------------------------

## 16. Scalability ও Reliability

### 16.1 Read ও Write আলাদা করে ভাবা

Product browsing সাধারণত read-heavy; cache/CDN এবং প্রয়োজন হলে read
replica সাহায্য করতে পারে। Checkout, inventory, coupon ও payment state
write-critical; এগুলোর authoritative write primary transactional
database-এ হবে।

### 16.2 Horizontal Scaling

API stateless রাখলে একাধিক instance চালানো যায়। Session state client
token বা shared store-এ থাকবে। Rate limit ও idempotency record
distributed environment-এ shared storage/DB-তে রাখতে হবে; in-memory map
যথেষ্ট নয়।

### 16.3 Queue কোথায় ব্যবহার করব?

-   Email/SMS notification
-   Search index update
-   Analytics/event processing
-   Invoice generation
-   Retryable non-critical integration

Stock reservation বা payment success-এর authoritative update কেবল
asynchronous queue consumer-এর অনিশ্চিত eventual execution-এর ওপর নির্ভর
করা যাবে না। প্রয়োজনীয় DB state transaction-এ commit করে পরে outbox event
publish করব।

### 16.4 Database growth

-   `orders`, `order_items`, `payments`-এ query-appropriate indexes।
-   Slow query analysis ও connection pooling।
-   Backup, point-in-time recovery এবং restore drill।
-   Workload প্রমাণ করলে historical tables partitioning।
-   Read replicas analytics/reporting-এ ব্যবহার করা যেতে পারে; critical
    stock decision stale replica থেকে নয়।

### 16.5 Reliability pattern

-   Idempotency keys
-   Transactional outbox
-   Retry with exponential backoff and jitter
-   Dead-letter queue
-   Payment reconciliation
-   Health checks and graceful shutdown
-   Backup/restore testing
-   Circuit breaker/timeouts for external providers

------------------------------------------------------------------------

## 17. Monitoring ও Logging

নিম্নলিখিত metric ও alert গুরুত্বপূর্ণ:

-   Checkout success/failure rate
-   `INSUFFICIENT_STOCK` rate
-   Payment pending কতক্ষণ থাকছে
-   Webhook signature failures ও processing lag
-   Expired reservation count ও expiry worker lag
-   Coupon reservation/redeemed counter mismatch
-   DB transaction latency, lock wait, deadlock ও connection pool
    saturation
-   Queue depth, retry count ও dead-letter message
-   Payment provider latency/error rate
-   Order confirmation বনাম gateway settlement mismatch

প্রতিটি checkout, order, payment ও webhook event-এ correlation/request ID
রাখা উচিত। Log-এ card number, password, token বা secret রাখা যাবে না।
Alert threshold production traffic ও baseline অনুযায়ী নির্ধারণ করতে হবে।

------------------------------------------------------------------------

## 18. Interview-এ বলার মতো সংক্ষিপ্ত উত্তর

### 18.1 System Design --- ২--৩ লাইনে

"আমি প্রথমে modular backend, PostgreSQL এবং Redis cache দিয়ে system
design করব। Product browsing-এর মতো read-heavy request cache করতে পারি,
কিন্তু checkout-এর stock ও price PostgreSQL থেকে verify করব। Order, order
items এবং stock reservation একই database transaction-এ তৈরি করব; payment
gateway-এর সঙ্গে idempotency, signed webhook ও reconciliation ব্যবহার
করব।"

### 18.2 Pool/Inventory concurrency কীভাবে handle করব?

"Stock reserve করার সময়
`UPDATE inventory SET available_qty = available_qty - quantity WHERE available_qty >= quantity`
conditional update ব্যবহার করব। একই row-তে concurrent update database
serialize করবে; update কোনো row না পেলে stock insufficient। Reservation
ও order একই transaction-এ থাকবে, তাই overselling এবং partial order এড়ানো
যাবে।"

### 18.3 ACID কীভাবে ব্যবহার হবে?

"Atomicity দিয়ে stock reservation, order ও order items একসঙ্গে commit বা
rollback করব। Consistency-এর জন্য quantity check, foreign key ও unique
constraint থাকবে। Isolation-এর জন্য atomic conditional update বা row lock
ব্যবহার করব, আর Durability PostgreSQL commit ও backup/recovery strategy
দিয়ে নিশ্চিত করব।"

### 18.4 Payment successful হলেও server crash করলে?

"Payment gateway-এর signed webhook event ID-সহ durable ভাবে record করব।
Duplicate webhook idempotently handle করব এবং pending payment
reconciliation worker দিয়ে gateway status আবার যাচাই করব। Reservation
expire হয়ে গেলে stock oversell না করে business policy অনুযায়ী fulfillment
বা refund/void করব।"

### 18.5 Duplicate checkout আটকাব কীভাবে?

"Client `Idempotency-Key` পাঠাবে। `(user_id, idempotency_key)` unique
constraint ও request hash দিয়ে একই request-এর result পুনরায় দেব; একই
key-তে আলাদা payload এলে `409 Conflict` দেব।"

### 18.6 Coupon cap কীভাবে enforce করব?

"Coupon-এর capacity reserve করার সময় atomic conditional update বা row
lock ব্যবহার করব। Reservation, redemption এবং release একই
transaction-safe accounting দিয়ে maintain করব, যাতে concurrent request-এ
maximum redemption অতিক্রম না হয়।"

### 18.7 কেন microservices নয়?

"প্রথমে modular monolith রাখলে transaction ও deployment সহজ হয়। Traffic,
team ownership বা independent scaling-এর প্রয়োজন প্রমাণিত হলে catalog,
order, inventory ও payment আলাদা service করা যায়। Strong consistency
দরকার এমন workflow অকারণে network boundary-তে ভাগ করব না।"

------------------------------------------------------------------------

## 19. Final Interview Checklist

### Requirements ও architecture

-   [ ] Functional এবং non-functional requirements বলতে পারি।
-   [ ] Modular monolith বনাম microservices-এর trade-off ব্যাখ্যা করতে
    পারি।
-   [ ] Product read path এবং checkout write path আলাদা করে দেখাতে পারি।

### Database

-   [ ] Users, products, inventory, carts, orders, order items,
    payments, coupons ও reservations-এর সম্পর্ক বুঝি।
-   [ ] Foreign key, unique constraint, check constraint ও indexes
    ব্যাখ্যা করতে পারি।
-   [ ] Order item-এ price snapshot কেন প্রয়োজন বুঝি।
-   [ ] টাকা integer minor unit-এ রাখার কারণ বলতে পারি।

### Concurrency ও ACID

-   [ ] Conditional inventory update লিখতে পারি।
-   [ ] Concurrent buyer-এর ক্ষেত্রে final stock কীভাবে protect হয় বুঝি।
-   [ ] Multi-item reservation fail হলে transaction rollback হয় ব্যাখ্যা
    করতে পারি।
-   [ ] Lock ordering, deadlock ও bounded retry সম্পর্কে বলতে পারি।
-   [ ] ACID-এর চারটি property checkout-এর সঙ্গে যুক্ত করতে পারি।

### Payment ও recovery

-   [ ] External gateway call DB transaction-এর বাইরে রাখার কারণ জানি।
-   [ ] Idempotency key ও request hash-এর কাজ বুঝি।
-   [ ] Signed webhook, duplicate event handling এবং reconciliation
    ব্যাখ্যা করতে পারি।
-   [ ] Late webhook ও expired reservation-এর policy ব্যাখ্যা করতে পারি।
-   [ ] Refund/void workflow ও audit log-এর প্রয়োজন বুঝি।

### Operations

-   [ ] Reservation expiry worker কীভাবে stock release করে বুঝি।
-   [ ] Cache কখন stale হতে পারে এবং কেন stock source of truth নয় বুঝি।
-   [ ] Queue/outbox, retries, monitoring, backup ও restore strategy
    সম্পর্কে বলতে পারি।

------------------------------------------------------------------------

## উপসংহার

এই design-এর মূল লক্ষ্য শুধু দ্রুত checkout করা নয়; **একই সময়ে বহু customer
request এলেও inventory, order, coupon এবং payment state সঠিক রাখা**।
PostgreSQL transaction ও constraints critical data consistency নিশ্চিত
করবে, idempotency duplicate request আটকাবে, reservation expiry abandoned
checkout-এর stock ফেরত দেবে, আর webhook/reconciliation external payment
failure থেকে recovery করতে সাহায্য করবে।

> **Interview-এর মূল কথা:** "Stock-এর source of truth database; checkout
> transaction-এ atomic reservation করব, payment gateway-কে idempotently
> integrate করব এবং webhook ও reconciliation দিয়ে failure recover করব।"
