# Buzzsynx — Caching and Queues

**Document:** `docs/09-caching-and-queues.md`
**Project:** Buzzsynx
**Version:** 0.2
**Status:** Architecture Specification
**Scope:** Architecture-aligned caching and asynchronous processing baseline

---

# 1. Purpose

Buzzsynx uses Redis and BullMQ to improve:

* Application performance
* API responsiveness
* Background processing
* Scheduled operations
* AI processing
* Notifications
* Report generation
* Analytics processing
* Rate limiting
* Temporary data handling

The initial implementation is focused on the **supermarket/grocery business domain**.

The caching and queue architecture is shared across the platform and must also support future industry capabilities without changing the fundamental data-consistency model.

The most important separation is:

```text
PostgreSQL → Source of Truth

Redis      → Cache / Temporary / Coordination Layer

BullMQ     → Background Job Processing
```

The core principle is:

> **Redis and queues improve the system; they do not replace PostgreSQL or transactional business logic.**

---

# 2. Core Architecture

```text
                         BUZZSYNX
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
        PostgreSQL                       Redis
      Source of Truth            Cache / Temporary / Limits
             │                             │
             │                             │
             └──────────────┬──────────────┘
                            │
                            ▼
                          BullMQ
                            │
                     Background Workers
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
         AI            Notifications       Reports
                            │
                            ▼
                       Analytics /
                       Maintenance
```

PostgreSQL owns business truth.

Redis provides speed and temporary infrastructure capabilities.

BullMQ provides asynchronous execution.

---

# 3. Technology Responsibilities

## PostgreSQL

PostgreSQL is responsible for authoritative business data, including:

* Tenants
* Stores
* Users
* Memberships
* Roles and permissions
* Products
* Categories
* Suppliers
* Purchases
* Inventory
* Stock movements
* Sales
* Payments
* Invoices
* Customers
* Business configuration
* Audit records
* AI insight records where persistence is required
* Other transactional business records

PostgreSQL is the final authority when cached or derived data conflicts with database state.

---

## Redis

Redis may be used for:

* Cache
* Rate limiting
* Short-lived temporary state
* Distributed coordination where required
* Queue infrastructure for BullMQ
* Frequently accessed derived data
* Short-lived session data **only if the selected authentication architecture requires it**

Redis must not become the source of truth for business transactions.

---

## BullMQ

BullMQ is responsible for:

* Background jobs
* Scheduled jobs
* Retryable jobs
* AI processing
* Notifications
* Emails
* Reports
* Analytics processing
* Maintenance tasks
* Other workloads that do not need to block the request

BullMQ uses Redis as its underlying queue infrastructure.

---

# 4. Source-of-Truth Rule

The architecture follows:

```text
PostgreSQL
     ↓
Authoritative Business State

Redis
     ↓
Derived / Temporary / Cached State

BullMQ
     ↓
Asynchronous Work
```

If Redis disagrees with PostgreSQL:

> **PostgreSQL wins.**

If a background job fails:

> **The core business transaction remains authoritative.**

---

# 5. What Must NOT Exist Only in Redis

Redis must never be the only source of truth for:

```text
Inventory quantity
Stock movements
Sales
Payments
Invoices
Products
Purchases
Customers
Tenant configuration
User memberships
Permissions
Financial records
Audit records
```

If Redis becomes unavailable, these records must remain safely stored in PostgreSQL.

---

# 6. Cache-Aside Pattern

Buzzsynx should primarily use the cache-aside pattern.

```text
Application
     │
     ▼
Check Redis
     │
     ├── HIT
     │    ↓
     │  Return Cached Data
     │
     └── MISS
          ↓
      PostgreSQL
          ↓
      Store Result
          ↓
      Return Data
```

Conceptually:

```js
const cached = await redis.get(key);

if (cached) {
  return JSON.parse(cached);
}

const data = await repository.getData();

await redis.set(key, JSON.stringify(data), {
  EX: 300
});

return data;
```

The actual implementation should be centralized through the Redis/cache infrastructure rather than duplicating cache logic throughout business modules.

---

# 7. Cache Key Design

Cache keys must be:

* Predictable
* Scoped
* Versionable where required
* Collision-resistant
* Appropriate to the data ownership level

For tenant-scoped data:

```text
tenant:{tenantId}:products:list
tenant:{tenantId}:product:{productId}
tenant:{tenantId}:dashboard:summary
```

For store-scoped data:

```text
tenant:{tenantId}:store:{storeId}:inventory:summary
tenant:{tenantId}:store:{storeId}:analytics:sales:{period}
```

Avoid unsafe global keys for tenant-owned data.

Unsafe:

```text
products:list
```

Safer:

```text
tenant:tenant_123:products:list
```

---

# 8. Tenant and Store Isolation

Redis must follow the same isolation model as PostgreSQL.

Example:

```text
Tenant A
  ├── Store A1
  └── Store A2
```

Cache keys must reflect the actual ownership scope.

For tenant-level data:

```text
tenant:A:products:list
```

For store-level data:

```text
tenant:A:store:A1:inventory:summary
```

A cache key must never allow data belonging to:

```text
Tenant B
```

or:

```text
Store A2
```

to be returned to an unauthorized request for:

```text
Store A1
```

Application authorization and scope validation remain mandatory even when cached data exists.

---

# 9. Cache Scope Classification

Before introducing a cache, determine its ownership.

## Tenant-scoped

Examples:

```text
Product catalog
Categories
Brands
Tenant configuration
```

## Store-scoped

Examples:

```text
Stock summary
Store sales summary
Store inventory dashboard
Store-specific pricing
Store analytics
```

## User-scoped

Examples:

```text
Temporary UI preferences
Short-lived user-specific state
```

## System-scoped

Examples:

```text
Non-sensitive application configuration
Infrastructure metadata
```

The scope determines the cache-key structure.

---

# 10. Cache TTL

Every cache should have an intentional TTL.

Typical examples:

```text
Product reference data       → Short / Medium
Dashboard summaries          → Short
Analytics summaries          → Medium
Static configuration         → Longer
AI insights                  → Feature-specific
Rate-limit counters          → Short
Temporary verification data → Very short
```

TTL should depend on:

* Data volatility
* Business importance
* Query cost
* Acceptable staleness
* Invalidation strategy

Do not cache business data indefinitely without a clear reason.

---

# 11. Cache Invalidation

Cache invalidation should be tied to the business event that changes the underlying data.

Example:

```text
Product Updated
      ↓
PostgreSQL Transaction
      ↓
COMMIT
      ↓
Invalidate / Refresh Relevant Cache
```

Example affected keys:

```text
tenant:A:product:123
tenant:A:products:list
tenant:A:dashboard:summary
```

Only invalidate caches that are actually affected.

Avoid indiscriminately deleting large portions of the cache after every mutation.

---

# 12. Database First for Mutations

Critical mutations must write to PostgreSQL first.

Preferred:

```text
Application
     ↓
PostgreSQL Transaction
     ↓
COMMIT
     ↓
Cache Invalidation / Refresh
```

Never:

```text
Application
     ↓
Redis
     ↓
PostgreSQL
```

as the authoritative transaction pattern.

Example:

```text
Sale Created
     ↓
PostgreSQL Transaction
     ↓
COMMIT
     ↓
Invalidate affected cache
```

---

# 13. Cache Invalidation Failure

A cache invalidation failure must not roll back an already committed business transaction.

Example:

```text
PostgreSQL Transaction
        ↓
COMMIT SUCCESS
        ↓
Redis Invalidation
        ↓
FAILURE
```

The sale remains valid.

The system should:

* Log the failure
* Rebuild or invalidate later
* Use TTL as a secondary protection
* Optionally enqueue cache-refresh work where justified

This reinforces the rule:

> **Cache consistency is secondary to database correctness.**

---

# 14. Redis Failure Strategy

Redis failure must be handled according to its specific purpose.

## Cache Failure

Where safe:

```text
Redis unavailable
      ↓
Read PostgreSQL
```

The application may become slower but remains functionally correct.

## Rate-Limit Failure

The application must have an explicitly defined policy.

For high-risk authentication endpoints, silently disabling rate limiting may be unsafe.

Possible policy:

```text
Fail closed
or
Use secondary protection
or
Apply conservative fallback limits
```

The decision should be made per endpoint risk.

## Queue Infrastructure Failure

New background jobs may fail to enqueue.

Critical workflows should use a reliable event-delivery strategy where required, such as a transactional outbox.

---

# 15. Cache Stampede Protection

If a popular cache expires, many requests may simultaneously query PostgreSQL.

Potential controls:

* Request coalescing
* Background refresh
* Stale-while-revalidate
* Randomized TTL
* Short-lived locks where appropriate

Example:

```text
Cache Expires
      ↓
One Request Refreshes
      ↓
Other Requests Reuse Existing / Stale Value
```

Do not introduce distributed locking everywhere.

Use it only when there is a measurable concurrency problem.

---

# 16. What Should Be Cached?

Good candidates:

```text
Product reference data
Categories
Brands
Tenant configuration
Dashboard summaries
Analytics summaries
AI insights
Frequently accessed read-heavy data
```

Potentially poor candidates:

```text
Payment state as source of truth
Inventory balance as source of truth
Financial transactions
Audit records
Critical authorization state without careful invalidation
```

Caching is an optimization, not a replacement for authoritative reads.

---

# 17. Product Cache

Product lookup is a good candidate for caching because POS may perform frequent reads.

Example:

```text
tenant:{tenantId}:product:{productId}
```

Potential cached fields:

```text
Product ID
SKU
Barcode
Name
Category
Display information
Applicable pricing reference
```

The exact cache content should remain small and useful.

Product cache must be invalidated when relevant product data changes.

---

# 18. POS Caching

POS requires low latency.

Potential cache candidates:

```text
Product lookup
Barcode lookup
Category lookup
Read-heavy product information
Store pricing configuration
```

Example:

```text
Barcode
   ↓
Redis
   ↓
Product ID / Product Reference
```

However:

> **POS caching must never bypass authoritative transaction validation.**

---

# 19. POS Inventory Rule

A cached stock value must never be treated as the final authority during checkout.

Example:

```text
Redis:
Stock = 10
```

PostgreSQL may now contain:

```text
Stock = 2
```

The sale transaction must validate the authoritative inventory state using PostgreSQL transaction logic and appropriate concurrency control.

Therefore:

> **Cache accelerates lookup; the database transaction enforces correctness.**

---

# 20. POS Price and Tax Rule

Cached pricing or tax configuration may improve lookup performance.

However, the final sale calculation must be performed by the authoritative application service using current valid configuration.

The server must calculate:

```text
Price
Discount
Tax
Totals
Payment Allocation
```

The frontend must not be the final authority for these values.

---

# 21. Dashboard Caching

Dashboard queries may combine:

```text
Sales
Inventory
Purchasing
Customers
Payments
```

Repeated aggregation can become expensive.

A cached summary may therefore be used:

```text
tenant:{tenantId}:store:{storeId}:dashboard:summary
```

The summary can be:

* Invalidated after relevant events
* Refreshed periodically
* Rebuilt on cache miss

The exact strategy depends on query cost and data freshness requirements.

---

# 22. Analytics Caching

Analytics may become increasingly expensive as tenant data grows.

Typical flow:

```text
PostgreSQL
     ↓
Aggregation
     ↓
Derived Result
     ↓
Redis Cache
     ↓
Dashboard
```

For larger workloads, consider:

* Precomputed analytics tables
* Materialized views
* Incremental aggregation
* Appropriate database indexes

Redis alone should not be used to solve an inefficient analytical query.

---

# 23. AI Caching

AI results can be expensive to generate.

Appropriate results may be cached or persisted.

Example:

```text
tenant:{tenantId}:store:{storeId}:ai:sales-summary:2026-09-25
```

AI results should include freshness metadata such as:

```text
generatedAt
expiresAt
sourcePeriod
```

Cached AI output must not be treated as current business truth.

---

# 24. AI Cache Strategy

Example:

```text
User Requests Insight
        ↓
Check Cache / Persisted Insight
        │
        ├── Fresh
        │     ↓
        │   Return Result
        │
        └── Missing / Stale
                ↓
            Queue Job
                ↓
            AI Worker
                ↓
          Generate Result
                ↓
          Validate Output
                ↓
          Persist Result
                ↓
             Cache
```

Where immediate generation is unnecessary, asynchronous processing is preferred.

---

# 25. Rate Limiting

Redis is suitable for distributed rate limiting.

Authenticated example:

```text
rate-limit:{tenantId}:{userId}:{endpoint}
```

User-specific example:

```text
rate-limit:{userId}:{endpoint}
```

Public endpoint example:

```text
rate-limit:ip:{ip}:{endpoint}
```

Unauthenticated requests cannot rely on `tenantId`.

Rate-limit strategy should consider:

* IP
* User
* Tenant
* Endpoint
* Authentication state
* Risk level
* Usage policy

Avoid storing raw IP information longer than operationally necessary.

---

# 26. Temporary Data

Redis may store short-lived information such as:

```text
Rate-limit counters
OTP state
Temporary verification state
Short-lived tokens where architecture requires
Temporary coordination state
Short-lived UI/session state where appropriate
```

Sensitive temporary data must have:

* Short TTL
* Restricted access
* Safe key naming
* Minimal payload
* No unnecessary logging

Authentication/session storage must follow the final authentication architecture rather than being assumed to use Redis.

---

# 27. Distributed Locks

Redis locks may be useful for selected coordination problems.

Potential examples:

```text
Duplicate report generation
Scheduled AI generation
Certain maintenance jobs
Preventing duplicate non-transactional processing
```

Locks should have:

* Unique ownership
* Expiration
* Safe release
* Bounded duration

Redis locks must not replace PostgreSQL transaction guarantees.

For inventory correctness, use database transactions and appropriate concurrency control rather than relying on a Redis lock.

---

# 28. BullMQ Architecture

BullMQ provides asynchronous job processing.

```text
Application
     ↓
Create Job / Event
     ↓
BullMQ
     ↓
Redis
     ↓
Worker
     ↓
Business Service
     ↓
PostgreSQL / External Provider
```

BullMQ is appropriate when work:

* Takes significant time
* Does not need to block the request
* Can be retried
* Is scheduled
* Depends on external services
* Can be processed independently

---

# 29. Why Use Queues?

Queues are useful for:

```text
AI processing
Email delivery
Notifications
Report generation
Analytics processing
File processing
Scheduled analysis
Maintenance
```

Queues provide:

* Asynchronous execution
* Retry capability
* Controlled concurrency
* Backpressure
* Failure isolation

---

# 30. Jobs That Must Not Block POS

A POS transaction should not wait for:

```text
AI analysis
Email delivery
Analytics processing
PDF rendering
Notification delivery
Non-critical report generation
```

Preferred:

```text
POS Request
     ↓
Critical PostgreSQL Transaction
     ↓
COMMIT
     ↓
Async Processing
```

The checkout path should remain focused on the minimum operations required to complete the sale correctly.

---

# 31. Queue Categories

Initial queue categories:

```text
queues/

├── ai
├── notifications
├── emails
├── reports
├── analytics
└── maintenance
```

These are logical categories.

The physical deployment can evolve later.

Do not create dozens of independent queues before workload requires them.

---

# 32. AI Queue

Potential jobs:

```text
GENERATE_SALES_SUMMARY
GENERATE_DEMAND_FORECAST
GENERATE_REORDER_RECOMMENDATION
DETECT_DEAD_STOCK
DETECT_ANOMALIES
GENERATE_EXPIRY_ANALYSIS
```

Example:

```js
{
  tenantId,
  storeId,
  jobType: "GENERATE_DEMAND_FORECAST",
  payload: {
    productId: "product_123"
  }
}
```

`storeId` should be present whenever the analysis is store-specific.

---

# 33. Notification Queue

Potential jobs:

```text
LOW_STOCK_ALERT
EXPIRY_ALERT
PAYMENT_NOTIFICATION
USER_INVITATION
SYSTEM_ALERT
```

Flow:

```text
Business Event
      ↓
Notification Job
      ↓
BullMQ
      ↓
Notification Worker
      ↓
Email / In-App / Other Provider
```

Notification failure must not normally fail the originating business transaction.

---

# 34. Email Queue

Email should generally be asynchronous.

Examples:

```text
Welcome Email
Password Reset
User Invitation
Invoice Email
Report Delivery
Payment Confirmation
```

Flow:

```text
Application
    ↓
Create Email Job
    ↓
BullMQ
    ↓
Email Worker
    ↓
Email Provider
```

The email provider must not be called inside a critical PostgreSQL transaction.

---

# 35. Report Queue

Large reports should not block normal API requests.

```text
User Requests Report
       ↓
Authorization
       ↓
Create Report Job
       ↓
Return Job ID
       ↓
Worker Generates Report
       ↓
Store File
       ↓
Notify User
```

The report file must have appropriate tenant/store access controls.

---

# 36. Scheduled Jobs

Scheduled processing may include:

```text
Daily Sales Summary
Low-Stock Analysis
Expiry Checks
AI Forecast Refresh
Weekly Reports
Monthly Analytics
Cleanup Tasks
```

Example:

```text
Scheduler
    ↓
Create Tenant/Store Jobs
    ↓
BullMQ
    ↓
Workers
```

Scheduled jobs must not assume that all tenants share the same data scope.

---

# 37. Tenant-Aware Jobs

Tenant-specific jobs must carry sufficient scope.

Example:

```js
{
  tenantId: "tenant_123",
  storeId: "store_01",
  type: "LOW_STOCK_ANALYSIS"
}
```

The worker must validate the payload before processing.

Tenant and store scope must be applied to all database queries.

---

# 38. Worker Security Model

Workers operate outside a normal HTTP request, but they are still part of the trusted application backend.

A worker should:

```text
Receive Job
    ↓
Validate Payload
    ↓
Resolve Trusted Tenant / Store Context
    ↓
Apply Required Service-Level Authorization
    ↓
Execute Business Service
    ↓
Scoped Database Access
```

A worker does not need to recreate a user's browser session or pretend every job is an interactive HTTP request.

However:

> **A job payload must never grant arbitrary access to another tenant.**

The worker's service identity and job scope must be trusted and controlled by the backend.

---

# 39. Job Payload Design

Job payloads should contain only the information required for processing.

Prefer:

```js
{
  tenantId,
  storeId,
  resourceId,
  jobType
}
```

Avoid putting unnecessary:

```text
Passwords
Access tokens
Payment secrets
Large customer records
Full business datasets
```

into queue payloads.

Where practical, workers should fetch current data from PostgreSQL using the scoped identifiers.

This also prevents stale job payloads from becoming a second source of truth.

---

# 40. Job Idempotency

Jobs may be retried or delivered more than once.

Important jobs should therefore be idempotent where practical.

For example:

```text
Generate Daily Report
```

should not create uncontrolled duplicate records every time it is retried.

For financial or transactional workflows, use:

* Idempotency keys
* Unique database constraints
* Event IDs
* Processing records
* Transactional state checks

where appropriate.

---

# 41. Critical API Idempotency

Idempotency is especially important for operations such as:

```text
Sale creation
Purchase receiving
Returns
Payment processing
Webhook handling
Important event processing
```

For example:

```text
Client Request
     ↓
Idempotency Key
     ↓
Business Transaction
```

A retried request should not accidentally create:

```text
Two sales
Two payments
Two stock movements
Two refunds
```

when only one operation was intended.

---

# 42. Retry Strategy

Not every failure should be retried.

## Usually retryable

```text
Temporary network failure
Provider timeout
Temporary external API failure
Transient database error
External API rate limit
```

## Usually non-retryable

```text
Invalid input
Authorization failure
Missing permanent resource
Malformed business data
Unsupported operation
```

Retry decisions should be defined per job type.

---

# 43. Exponential Backoff

Retryable jobs should use controlled backoff.

```text
Attempt 1
    ↓
Wait
    ↓
Attempt 2
    ↓
Longer Wait
    ↓
Attempt 3
```

Avoid immediate repeated retries that can overload:

* PostgreSQL
* Redis
* AI providers
* Email providers
* External APIs

---

# 44. Failed Jobs

Repeatedly failing jobs must become observable.

Example:

```text
Job
 ↓
Retry
 ↓
Retry
 ↓
Retry
 ↓
Failed
 ↓
Failed Job / Monitoring
```

Operators should be able to identify:

* Job type
* Tenant/store scope where appropriate
* Failure reason
* Attempt count
* Timestamp
* Request/event reference

Sensitive payload data should not be unnecessarily exposed.

---

# 45. Dead-Letter Handling

A dead-letter or equivalent failed-job mechanism may be introduced when required.

It should support:

* Inspection
* Controlled retry
* Failure analysis
* Operational alerting

Do not build an elaborate dead-letter system before there is a real operational need, but do not allow permanently failing jobs to disappear silently.

---

# 46. Queue Priorities

Potential priority classes:

```text
High
  Time-sensitive notifications
  Important operational jobs

Medium
  Standard business processing

Low
  Historical analytics
  Large reports
  Non-urgent AI analysis
```

Actual priority design should follow measured workload and business importance.

---

# 47. Queue Concurrency

Workers must have controlled concurrency.

Example:

```text
AI Worker
Controlled concurrency

Email Worker
Higher concurrency where provider permits

Report Worker
Controlled concurrency

Analytics Worker
Controlled concurrency
```

Unlimited concurrency can overwhelm:

```text
PostgreSQL
Redis
AI Providers
Email Providers
External APIs
```

Concurrency should therefore be configurable.

---

# 48. Backpressure

Queues provide a buffer when workload temporarily increases.

```text
Requests
    ↓
Jobs Increase
    ↓
Queue Grows
    ↓
Workers Process at Controlled Rate
```

Monitor queue depth and processing latency.

If queues continuously grow, investigate:

* Worker capacity
* Database bottlenecks
* Provider limits
* Job design
* Concurrency
* Scheduling frequency

Do not simply increase worker count without checking downstream capacity.

---

# 49. Queue Observability

Monitor:

```text
Queue depth
Processing rate
Job latency
Failure rate
Retry count
Worker health
Stuck jobs
Provider failures
```

Important job failures should generate appropriate operational alerts.

---

# 50. Redis Memory Management

Redis memory must be controlled.

Use:

* TTLs
* Bounded cache sizes
* Appropriate eviction policy
* Monitoring
* Key naming conventions
* Removal of obsolete cache entries

Do not allow unlimited application-generated keys.

BullMQ retention settings should also be controlled so completed and failed jobs do not grow indefinitely.

---

# 51. Redis Key Naming Convention

Recommended general pattern:

```text
{scope}:{tenantId}:{storeId}:{domain}:{resource}:{identifier}
```

Only include scope components that are actually relevant.

Examples:

```text
tenant:A:products:list

tenant:A:product:123

tenant:A:store:A1:inventory:summary

tenant:A:store:A1:analytics:sales:monthly

tenant:A:store:A1:ai:forecast:product-123
```

System-level keys:

```text
system:health
system:config
```

Rate limits:

```text
rate-limit:user:user-123:login
rate-limit:ip:203.x.x.x:login
```

The exact format can evolve, but naming must remain consistent.

---

# 52. Cache Versioning

Cache formats may change as application code evolves.

Use versioned keys when necessary.

Example:

```text
tenant:A:dashboard:v2
```

This prevents incompatible cached structures from being interpreted by newer application code.

---

# 53. Cache Serialization

Use a predictable serialization strategy.

For simple application data:

```text
JSON
```

is generally sufficient.

Avoid unnecessarily caching:

* Huge objects
* Entire database records
* Large nested datasets
* Data that can be cheaply queried

Cache only what provides meaningful performance benefit.

---

# 54. Cache Security

Do not store unnecessary secrets in Redis.

If sensitive temporary information must be stored:

* Use short TTLs
* Restrict Redis network access
* Protect Redis credentials
* Avoid logging values
* Minimize payload
* Scope keys correctly
* Apply infrastructure encryption where appropriate

Redis should not be directly exposed to the public internet.

---

# 55. PostgreSQL + Redis Consistency

The relationship is:

```text
PostgreSQL
    ↓
Authoritative State

Redis
    ↓
Derived / Cached State
```

If they disagree:

```text
PostgreSQL wins.
```

The application should:

* Invalidate stale cache
* Rebuild cache
* Continue from authoritative data

Cache corruption must never become business-data corruption.

---

# 56. PostgreSQL + Queue Consistency

A common failure scenario is:

```text
Database Transaction
      ↓
COMMIT SUCCESS
      ↓
Queue Creation
      ↓
FAILURE
```

The business transaction succeeded, but asynchronous work was not scheduled.

For important events, Buzzsynx should support the **transactional outbox pattern** where reliability requires it.

---

# 57. Transactional Outbox

Conceptually:

```text
Database Transaction
      │
      ├── Business Record
      │
      └── Outbox Event
              ↓
            COMMIT
              ↓
        Outbox Worker
              ↓
           BullMQ
              ↓
          Background Job
```

Example:

```text
Sale Created
+
SALE_CREATED Event
```

are committed atomically.

A worker can then publish/process the event after commit.

This prevents important asynchronous work from being lost because queue submission happened outside the transaction.

---

# 58. Outbox Scope

The outbox pattern should be used selectively.

Good candidates:

```text
Sale-created downstream processing
Payment state events
Inventory-related notifications
Important business notifications
Critical integration events
```

It does not need to be used for every cache refresh or low-value background task.

For simple non-critical cache refreshes, direct post-commit invalidation may be sufficient.

---

# 59. Event-Driven Processing

Business events can trigger background work.

Example:

```text
SALE_CREATED
      │
      ├── Analytics Processing
      ├── Notification
      ├── AI Analysis
      └── Cache Refresh
```

The core sale transaction remains deterministic.

Background consumers must not modify authoritative business state without passing through the appropriate business service and transaction rules.

---

# 60. Sale Workflow

Preferred architecture:

```text
Customer Checkout
       ↓
POS
       ↓
Validate Product / Price / Stock
       ↓
PostgreSQL Transaction
       │
       ├── Sale
       ├── Sale Items
       ├── Payment Allocation
       ├── Stock Movement
       ├── Invoice Record
       └── Outbox Event where required
       ↓
COMMIT
       ↓
Async Processing
       │
       ├── Analytics
       ├── Notifications
       ├── Cache Invalidation
       ├── Invoice PDF
       └── AI Analysis
```

External providers must not be called inside the critical PostgreSQL transaction.

---

# 61. Low-Stock Workflow

Example:

```text
Sale
 ↓
Stock Updated
 ↓
Transaction Commit
 ↓
Business Event
 ↓
Queue
 ↓
Low-Stock Worker
 ↓
Evaluate Threshold
 ↓
Create Alert / Insight
 ↓
Notification Queue
 ↓
Email / In-App Notification
```

The alert must not block checkout.

---

# 62. AI Forecast Workflow

```text
Scheduled Job
      ↓
BullMQ
      ↓
Forecast Worker
      ↓
Tenant / Store Context
      ↓
Fetch Historical Sales
      ↓
Prepare Dataset
      ↓
Forecast / AI Processing
      ↓
Validate Output
      ↓
Store Forecast
      ↓
Cache Result
      ↓
Dashboard
```

AI processing remains independent of POS execution.

---

# 63. Report Generation Workflow

```text
User
 ↓
Request Report
 ↓
Authentication
 ↓
Authorization
 ↓
Create Report Job
 ↓
Return Job ID
 ↓
BullMQ
 ↓
Report Worker
 ↓
Scoped PostgreSQL Query
 ↓
Generate Report
 ↓
Store File
 ↓
Notify User
```

Generated files must have tenant/store-aware access controls.

---

# 64. Notification Failure Isolation

Example:

```text
Sale
 ↓
SUCCESS
 ↓
Notification Job
 ↓
Email Provider
 ↓
FAILURE
```

The sale remains successful.

The notification job may retry independently.

This principle applies to:

```text
Email
AI
Reports
Analytics
Non-critical notifications
```

---

# 65. When NOT to Use a Queue

Do not queue operations that require immediate synchronous results unless the product explicitly supports asynchronous behavior.

Examples:

```text
Login
Product lookup
Barcode lookup
POS cart interaction
Stock validation during checkout
Payment verification
Simple CRUD
Authorization checks
```

These should normally execute synchronously.

---

# 66. When NOT to Use Redis

Do not introduce caching merely because Redis is available.

Avoid caching when:

```text
Data is rarely accessed
Query is already inexpensive
Data changes too frequently
Staleness is unacceptable
Invalidation is unnecessarily complex
Memory cost exceeds performance benefit
```

Every cache should have a measurable or architectural reason.

---

# 67. Initial Queue Architecture

For the initial Buzzsynx implementation:

```text
Redis
 │
 └── BullMQ
      │
      ├── ai
      ├── notifications
      ├── emails
      ├── reports
      ├── analytics
      └── maintenance
```

Workers may initially run in the same deployment environment while maintaining clear module boundaries.

They can later be separated when workload, scaling, reliability, or operational requirements justify it.

---

# 68. Initial Caching Scope

The initial supermarket implementation should prioritize a small number of useful caches:

```text
1. Product / barcode lookup
2. Read-heavy reference data
3. Dashboard summaries
4. Analytics summaries where expensive
5. AI insights
6. Rate limiting
```

Do not attempt to cache every database query.

---

# 69. Future Industry Support

The caching and queue architecture is industry-agnostic.

Future capabilities may introduce additional jobs such as:

```text
Pharmacy
  Expiry analysis
  Batch alerts

Clothing
  Variant analysis
  Size demand analysis

Restaurant
  Ingredient analysis
  Waste analysis
```

These should reuse the same:

```text
Redis
BullMQ
Tenant Scope
Store Scope
Worker Security
Idempotency
Observability
```

architecture.

---

# 70. Failure Scenarios

## Redis Cache Down

```text
Redis unavailable
      ↓
Cache miss / bypass
      ↓
PostgreSQL
```

Where safe, business operations continue with potentially higher latency.

---

## Redis Rate-Limit Infrastructure Down

Behavior depends on endpoint risk.

```text
Authentication endpoint
      ↓
Conservative protection / fail-closed policy
```

versus a low-risk internal cache operation that may simply bypass Redis.

The policy must be explicit.

---

## Worker Down

```text
API continues
      ↓
Jobs remain queued
      ↓
Worker Restarts
      ↓
Processing Resumes
```

Job retention and retry configuration must support this behavior.

---

## AI Provider Down

```text
AI Job
  ↓
Timeout / Failure
  ↓
Retry
  ↓
Failed Job if necessary
  ↓
Monitoring
```

Core business operations continue.

---

## Email Provider Down

```text
Email Job
   ↓
Retry
   ↓
Failed Job if necessary
```

The originating business transaction remains unaffected.

---

# 71. Security Requirements

## Redis

* [ ] Private network access
* [ ] Authentication/configured access controls
* [ ] Encryption where appropriate
* [ ] Tenant/store-aware keys
* [ ] Sensitive-data minimization
* [ ] TTLs
* [ ] Memory limits
* [ ] No business source of truth
* [ ] No public internet exposure

## BullMQ

* [ ] Protected Redis connection
* [ ] Tenant/store-aware jobs
* [ ] Job payload validation
* [ ] Worker service authorization
* [ ] Idempotency
* [ ] Retry controls
* [ ] Failure monitoring
* [ ] Sensitive payload minimization
* [ ] Controlled concurrency

---

# 72. Testing Strategy

## Cache Tests

Test:

* Cache hit
* Cache miss
* Cache expiration
* Cache invalidation
* Cache rebuild
* Redis unavailable
* Tenant isolation
* Store isolation
* Cache-key correctness
* Stale-data handling

---

## Queue Tests

Test:

* Job creation
* Job validation
* Job processing
* Retry behavior
* Failure behavior
* Idempotency
* Worker restart
* Queue recovery
* Tenant isolation
* Store isolation
* Concurrency limits

---

## Transaction / Event Tests

Test:

```text
Business Transaction
+
Outbox Event
```

including:

* Successful commit
* Transaction rollback
* Duplicate event
* Worker retry
* Event processing failure
* Event recovery

---

## Integration Tests

Test:

```text
PostgreSQL
+
Redis
+
BullMQ
+
Application
+
Workers
```

as an integrated system.

---

# 73. Observability Requirements

Monitor:

```text
Redis
  Memory
  Hit rate
  Miss rate
  Errors
  Connection health

BullMQ
  Queue depth
  Job latency
  Processing rate
  Failure rate
  Retry count
  Worker health

Application
  Cache errors
  Queue errors
  Event failures
  Slow queries
```

Do not use tenant IDs as unnecessarily high-cardinality metric labels.

Tenant/store context can instead be captured in structured logs and traces where appropriate.

---

# 74. Definition of Done

The caching and queue architecture is ready when:

* [ ] PostgreSQL is clearly defined as source of truth
* [ ] Redis responsibilities are defined
* [ ] BullMQ responsibilities are defined
* [ ] Tenant-aware cache keys exist
* [ ] Store-aware cache keys exist where required
* [ ] TTL strategy exists
* [ ] Cache invalidation strategy exists
* [ ] Redis failure behavior is defined
* [ ] Rate-limit failure behavior is defined
* [ ] Initial queues are defined
* [ ] Worker responsibilities are defined
* [ ] Job payloads are validated
* [ ] Tenant/store scope is enforced in workers
* [ ] Idempotency strategy exists
* [ ] Retry strategy exists
* [ ] Failed jobs are observable
* [ ] Worker concurrency is controlled
* [ ] Critical workflows do not depend on background jobs
* [ ] Transactional outbox is available for workflows that require reliable event delivery
* [ ] Cache and queue security is tested
* [ ] Redis memory usage is monitored
* [ ] Queue health is monitored
* [ ] Integration tests cover PostgreSQL + Redis + BullMQ

---

# 75. Final Architecture

Buzzsynx follows this separation:

```text
                         BUSINESS SYSTEM
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
                 ▼                             ▼
            PostgreSQL                       Redis
          Source of Truth             Cache / Temporary
                 │                   Rate Limits / Queue
                 │                             │
                 │                             │
                 └──────────────┬──────────────┘
                                │
                                ▼
                              BullMQ
                                │
                        Background Workers
                                │
             ┌──────────────────┼──────────────────┐
             ▼                  ▼                  ▼
            AI             Notifications         Reports
             │                  │                  │
             └──────────────────┼──────────────────┘
                                ▼
                         Analytics /
                         Maintenance
                                │
                                ▼
                         Observability
```

---

# 76. Core Architectural Boundaries

Buzzsynx maintains these boundaries:

```text
PostgreSQL
    ↓
Business Truth

Redis
    ↓
Performance / Temporary State

BullMQ
    ↓
Asynchronous Execution

Workers
    ↓
Scoped Backend Processing

AI / External Providers
    ↓
Non-authoritative Intelligence / Integration
```

No supporting infrastructure should silently become a second business database.

---

# 77. Final Principle

> **PostgreSQL owns truth. Redis provides speed. BullMQ provides asynchronous execution.**

The system should be designed so that:

```text
Redis Cache Failure
        ≠
Business Data Loss

Worker Failure
        ≠
Business Transaction Failure

AI Failure
        ≠
POS Failure

Email Failure
        ≠
Sale Failure

Analytics Failure
        ≠
Inventory Failure
```

For the initial Buzzsynx supermarket implementation:

```text
                    PostgreSQL
                        │
                 Source of Truth
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
        Redis                       BullMQ
     Fast Reads                  Async Work
          │                           │
          │              ┌────────────┼────────────┐
          │              ▼            ▼            ▼
          │             AI       Notifications   Reports
          │
          ▼
       Frontend
```

The architecture should remain simple enough for the current modular monolith while providing a clean path toward larger workloads.

> **Use Redis when speed matters.**

> **Use BullMQ when work does not need to block the user.**

> **Use PostgreSQL whenever business truth matters.**

> **Keep the critical path deterministic; move expensive and non-critical work outside it.**
