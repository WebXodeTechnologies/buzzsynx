# Buzzsynx — Caching and Queues

**Document:** `docs/09-caching-and-queues.md`
**Project:** Buzzsynx
**Version:** 1.0
**Status:** Architecture Specification

---

# 1. Purpose

Buzzsynx uses Redis and BullMQ to improve:

* Application performance
* API responsiveness
* Scalability
* Background processing
* Scheduled operations
* AI processing
* Notifications
* Rate limiting
* Temporary data handling

The architecture must clearly distinguish between:

```text
PostgreSQL → Source of Truth
Redis      → Fast/temporary data layer
BullMQ     → Background job processing
```

The most important principle is:

> **Redis and queues improve the system; they do not replace the database or transactional business logic.**

---

# 2. Core Architecture

```text
                    BUZZSYNX
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
     PostgreSQL                   Redis
     Source of Truth           Cache / Temp
          │                         │
          │                         │
          └────────────┬────────────┘
                       │
                       ▼
                    BullMQ
                       │
              Background Workers
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
      AI           Notifications     Reports
```

---

# 3. Technology Responsibilities

## PostgreSQL

Responsible for:

* Products
* Inventory
* Stock movements
* Sales
* Payments
* Customers
* Suppliers
* Users
* Tenants
* Roles
* Permissions
* Invoices
* Audit records
* AI insights
* Business transactions

---

## Redis

Responsible for:

* Cache
* Rate limiting
* Temporary state
* Short-lived sessions where applicable
* Distributed locks where required
* Queue infrastructure
* Frequently accessed derived data

---

## BullMQ

Responsible for:

* Background jobs
* Scheduled jobs
* Retryable jobs
* AI processing
* Notifications
* Reports
* Analytics processing
* Emails
* Maintenance tasks

---

# 4. What Must NOT Be Stored Only in Redis

Redis must never become the only source of truth for:

```text
Inventory quantity
Sales
Payments
Invoices
Customers
Products
Purchase records
Stock movements
Tenant configuration
User permissions
Financial records
```

If Redis is unavailable, these business records must remain safe in PostgreSQL.

---

# 5. Cache-Aside Pattern

Buzzsynx should primarily use the cache-aside pattern.

Flow:

```text
Application
     ↓
Check Redis
     │
     ├── HIT → Return cached data
     │
     └── MISS
           ↓
       PostgreSQL
           ↓
       Store in Redis
           ↓
       Return data
```

Example:

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

The exact implementation will be centralized in the Redis/cache infrastructure.

---

# 6. Cache Key Design

Cache keys must be predictable and tenant-aware.

Example:

```text
tenant:{tenantId}:products:list
tenant:{tenantId}:product:{productId}
tenant:{tenantId}:inventory:summary
tenant:{tenantId}:dashboard:summary
tenant:{tenantId}:analytics:sales:{period}
```

Avoid global keys for tenant-owned data.

Unsafe:

```text
products:list
```

Safer:

```text
tenant:tenant_123:products:list
```

---

# 7. Tenant Isolation in Redis

Redis must follow the same tenant isolation model as PostgreSQL.

Example:

```text
Tenant A

tenant:A:products:list
tenant:A:inventory:summary
tenant:A:dashboard:summary
```

```text
Tenant B

tenant:B:products:list
tenant:B:inventory:summary
tenant:B:dashboard:summary
```

Application code must never accidentally return Tenant B's cached data to Tenant A.

---

# 8. Cache TTL

Every cache should have an intentional TTL.

Examples:

```text
Product list          → short/medium TTL
Dashboard summary     → short TTL
Analytics             → medium TTL
Static configuration  → longer TTL
AI insight            → feature-specific TTL
Rate limits           → short TTL
```

TTL should be selected according to how quickly the underlying data can change.

Never cache business data indefinitely without an explicit invalidation strategy.

---

# 9. Cache Invalidation

Cache invalidation is one of the most important parts of the caching architecture.

Example:

```text
Product Updated
      ↓
Database Updated
      ↓
Invalidate Product Cache
      ↓
Invalidate Related Lists
```

For example:

```text
tenant:A:product:123
tenant:A:products:list
tenant:A:dashboard:summary
```

may need invalidation after a product change.

---

# 10. Database First

For important mutations:

```text
Application
     ↓
PostgreSQL Transaction
     ↓
Commit
     ↓
Cache Invalidation
```

Do not make Redis the first authority for critical business mutations.

Example:

```text
Sale Created
     ↓
PostgreSQL transaction
     ↓
Commit
     ↓
Invalidate affected cache
```

---

# 11. Cache Failure Strategy

Redis should be considered a performance dependency, not a business-data dependency.

If Redis fails:

```text
Redis unavailable
      ↓
Application continues
      ↓
Read from PostgreSQL
```

For rate limiting or other Redis-dependent security features, the failure behavior should be explicitly defined rather than accidentally bypassing protection.

---

# 12. Avoid Cache Stampede

If a popular cache expires and hundreds of requests simultaneously query PostgreSQL, the database can become overloaded.

Possible controls:

* Request coalescing
* Short randomized TTLs
* Distributed locks
* Background refresh
* Stale-while-revalidate

Example:

```text
Cache expires
     ↓
One request refreshes
     ↓
Other requests wait/use stale value
```

---

# 13. Cache What Makes Sense

Good cache candidates:

```text
Product catalog
Categories
Tenant configuration
Dashboard summaries
Analytics summaries
Frequently accessed reports
AI insights
Read-heavy reference data
```

Poor cache candidates:

```text
Payment state as source of truth
Inventory as source of truth
Financial transactions
Audit records
Critical authorization state without careful invalidation
```

---

# 14. Product Cache

Example:

```text
tenant:{tenantId}:product:{productId}
```

Useful for:

* Product details
* POS product lookup
* Frequently accessed product information

The cache must be invalidated when relevant product data changes.

---

# 15. POS Caching

POS requires low latency.

Potential cache candidates:

```text
Product lookup
Barcode lookup
Category lookup
Pricing configuration
```

Example:

```text
Barcode
   ↓
Redis
   ↓
Product ID
```

However, final sale validation must still use authoritative database state.

---

# 16. POS and Inventory Rule

A cached stock value must never be trusted as the final authority during a critical sale.

Example:

```text
Redis says:
Stock = 10
```

But PostgreSQL may now contain:

```text
Stock = 2
```

The transactional sale process must validate authoritative inventory state.

Therefore:

> **Cache can accelerate lookup; database transactions enforce correctness.**

---

# 17. Dashboard Caching

Dashboard data is often expensive to calculate.

Instead of repeatedly executing:

```text
Sales aggregation
+
Inventory aggregation
+
Customer aggregation
+
Purchase aggregation
```

the system can cache a summary.

Example:

```text
tenant:{tenantId}:dashboard:summary
```

The cache can be refreshed after relevant events or periodically.

---

# 18. Analytics Caching

Analytics queries can become expensive as tenant data grows.

Use:

```text
PostgreSQL
    ↓
Aggregation
    ↓
Cached Result
    ↓
Dashboard
```

Example:

```text
tenant:{tenantId}:analytics:sales:monthly
```

For larger workloads, precomputed analytics tables/materialized views may be more appropriate than relying only on Redis.

---

# 19. AI Caching

AI results can be expensive to generate.

Cache appropriate results.

Example:

```text
tenant:{tenantId}:ai:sales-summary:2026-09-25
```

However, AI results can become stale.

Therefore store:

```text
generatedAt
expiresAt
```

or equivalent freshness metadata.

---

# 20. AI Cache Strategy

Example:

```text
User requests insight
        ↓
Check Redis
        │
        ├── Fresh → Return
        │
        └── Missing/Stale
                 ↓
             Queue Job
                 ↓
             AI Worker
                 ↓
             Generate
                 ↓
             Store Result
                 ↓
             Cache
```

This avoids making every user request wait for an AI provider.

---

# 21. Rate Limiting

Redis is suitable for distributed rate limiting.

Example:

```text
rate-limit:{userId}:{endpoint}
```

Tenant-aware example:

```text
rate-limit:{tenantId}:{userId}:{endpoint}
```

For public endpoints:

```text
rate-limit:ip:{ip}:{endpoint}
```

The exact limits will be defined per endpoint and risk level.

---

# 22. Temporary Data

Redis can store short-lived data such as:

```text
OTP state
Password-reset state
Temporary verification state
Rate-limit counters
Short-lived UI/session state where appropriate
```

Sensitive temporary data must have:

* Short TTL
* Restricted access
* Secure key naming
* No unnecessary logging

---

# 23. Distributed Locks

Redis can provide distributed locking for selected operations.

Potential use cases:

```text
Scheduled AI generation
Duplicate report generation
Certain maintenance jobs
Preventing duplicate processing
```

Locks must have:

* Expiration
* Unique ownership
* Safe release behavior

Do not use Redis locks as a substitute for database transaction guarantees.

---

# 24. BullMQ Architecture

BullMQ provides asynchronous processing.

Architecture:

```text
Application
     ↓
Create Job
     ↓
Redis
     ↓
BullMQ Queue
     ↓
Worker
     ↓
Business Service
     ↓
PostgreSQL / External Provider
```

---

# 25. Why Use Queues?

Queues are useful when work:

* Takes significant time
* Does not need to block the request
* Can be retried
* Is scheduled
* Depends on external services
* Can run independently

Examples:

```text
AI processing
Email
Reports
Notifications
Analytics
Image processing
Scheduled tasks
```

---

# 26. Jobs That Should NOT Block POS

A POS transaction should not wait for:

```text
AI analysis
Email delivery
Analytics processing
PDF generation
Notification delivery
```

Instead:

```text
POS Transaction
      ↓
Complete critical DB transaction
      ↓
Queue background jobs
```

This keeps checkout fast and reliable.

---

# 27. Queue Categories

Recommended initial queues:

```text
queues/
├── ai
├── notifications
├── emails
├── reports
├── analytics
└── maintenance
```

Additional queues can be introduced when workload justifies them.

Avoid creating dozens of queues prematurely.

---

# 28. AI Queue

Jobs may include:

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
  jobType: "GENERATE_DEMAND_FORECAST",
  payload: {
    productId
  }
}
```

---

# 29. Notification Queue

Jobs:

```text
LOW_STOCK_ALERT
EXPIRY_ALERT
PAYMENT_NOTIFICATION
USER_INVITATION
SYSTEM_ALERT
```

Example:

```text
Business Event
      ↓
Notification Job
      ↓
BullMQ
      ↓
Notification Worker
      ↓
Email / Push / In-app
```

---

# 30. Email Queue

Email should generally be asynchronous.

Examples:

```text
Welcome email
Password reset
User invitation
Invoice email
Report delivery
Payment confirmation
```

Flow:

```text
Application
   ↓
Queue Email Job
   ↓
Email Worker
   ↓
Email Provider
```

---

# 31. Report Queue

Large reports should not block API requests.

Instead:

```text
User Requests Report
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

The frontend can poll or receive a notification when the report is ready.

---

# 32. Scheduled Jobs

BullMQ can support recurring tasks such as:

```text
Daily sales summary
Daily low-stock analysis
Expiry checks
Weekly business report
Monthly analytics
AI forecast refresh
Cleanup jobs
```

Example:

```text
Scheduler
   ↓
Daily 02:00
   ↓
Generate tenant jobs
   ↓
Workers process jobs
```

---

# 33. Tenant-Aware Jobs

Every tenant-specific job must contain tenant identity.

Example:

```js
{
  tenantId: "tenant_123",
  type: "LOW_STOCK_ANALYSIS"
}
```

The worker must explicitly initialize tenant context before accessing data.

---

# 34. Worker Security

Workers must follow the same tenant isolation rules as APIs.

Flow:

```text
Job
 ↓
Validate Job Payload
 ↓
Resolve Tenant
 ↓
Tenant Context
 ↓
Capability Check if required
 ↓
Business Service
 ↓
Tenant-scoped Database Access
```

Workers are not exempt from authorization architecture.

---

# 35. Job Idempotency

Jobs can be retried.

Therefore, important jobs should be safe to run more than once where possible.

Example:

```text
Generate daily report
```

If executed twice, it should not:

```text
Charge customer twice
Send duplicate financial transaction
Create duplicate invoice
```

Use:

* Idempotency keys
* Unique database constraints
* Job identifiers
* Processing records

where appropriate.

---

# 36. Retry Strategy

Not every error should be retried.

### Retryable

```text
Temporary network failure
Provider timeout
Temporary Redis failure
Transient database error
External API rate limit
```

### Usually non-retryable

```text
Invalid input
Authorization failure
Malformed permanent data
Missing required resource
```

---

# 37. Exponential Backoff

Retryable jobs should use controlled backoff.

Conceptually:

```text
Attempt 1
   ↓
Wait

Attempt 2
   ↓
Longer wait

Attempt 3
   ↓
Longer wait
```

Avoid aggressive immediate retries that can overload dependencies.

---

# 38. Dead-Letter / Failed Jobs

Jobs that repeatedly fail must become visible.

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
Failed Job Storage / Monitoring
```

Operators should be able to inspect:

* Job type
* Tenant
* Failure reason
* Attempts
* Timestamp

Sensitive payload data should not be unnecessarily exposed.

---

# 39. Queue Priorities

Not all jobs have equal urgency.

Potential priority model:

```text
High
  Critical notifications

Medium
  Standard business processing

Low
  AI analysis
  Historical analytics
  Large reports
```

The exact queue strategy can evolve based on workload.

---

# 40. Queue Concurrency

Workers should limit concurrency based on workload.

Example:

```text
AI Worker
Concurrency: controlled

Email Worker
Concurrency: higher

Report Worker
Concurrency: controlled
```

Do not allow unlimited concurrent jobs to overwhelm:

* PostgreSQL
* Redis
* AI providers
* Email providers
* External APIs

---

# 41. Backpressure

When demand increases:

```text
Requests
   ↓
Jobs increase
   ↓
Queue grows
```

The system should allow workers to process jobs at a controlled rate.

This is one of the major benefits of queues.

---

# 42. Queue Observability

Monitor:

```text
Queue depth
Processing rate
Job latency
Failure rate
Retry count
Worker health
Stuck jobs
Provider errors
```

These metrics should integrate with the observability architecture.

---

# 43. Redis Memory Management

Redis memory must be controlled.

Use:

* TTLs
* Appropriate eviction policy
* Bounded cache sizes
* Monitoring
* Key naming conventions

Do not allow unlimited application-generated keys.

---

# 44. Redis Key Naming Convention

Recommended:

```text
{scope}:{tenantId}:{domain}:{resource}:{identifier}
```

Examples:

```text
tenant:A:product:123
tenant:A:inventory:summary
tenant:A:analytics:sales:monthly
tenant:A:ai:forecast:product-123
```

System-level keys:

```text
system:health
system:config
```

Rate-limit keys:

```text
rate-limit:A:user-123:login
```

---

# 45. Cache Serialization

Cached objects should have a predictable serialization strategy.

Prefer:

```text
JSON
```

for simple application data.

Large or complex cached objects should be reviewed carefully to avoid unnecessary memory usage.

---

# 46. Cache Versioning

Cache formats may change as the application evolves.

Use versioned keys when necessary.

Example:

```text
tenant:A:dashboard:v2
```

This prevents incompatible old cached data from breaking new application code.

---

# 47. Cache Security

Do not store unnecessary secrets in Redis.

If sensitive temporary information must be stored:

* Use short TTLs
* Restrict Redis access
* Avoid logging values
* Encrypt at the appropriate infrastructure layer
* Scope keys correctly

---

# 48. PostgreSQL + Redis Consistency

The relationship should be:

```text
PostgreSQL
    ↓
Authoritative State

Redis
    ↓
Derived / Cached State
```

If there is disagreement:

> PostgreSQL wins.

The application should invalidate or rebuild stale Redis data.

---

# 49. PostgreSQL + Queue Consistency

A common problem:

```text
Database transaction succeeds
       ↓
Queue job creation fails
```

This can result in missed background processing.

For important workflows, consider the **transactional outbox pattern**.

---

# 50. Transactional Outbox

Conceptually:

```text
Database Transaction
 ├── Business Record
 └── Outbox Event
          ↓
       COMMIT
          ↓
     Outbox Worker
          ↓
       BullMQ
```

Example:

```text
Sale Created
+
SALE_CREATED event
```

are committed together.

A worker later publishes/processes the event.

This reduces the risk of losing important asynchronous work.

---

# 51. Event-Driven Processing

Buzzsynx can use business events to trigger background work.

Example:

```text
SALE_CREATED
     │
     ├── Update analytics
     ├── Queue notification
     ├── Queue AI analysis
     └── Update dashboard cache
```

The core sale transaction remains deterministic.

---

# 52. Example — Sale Workflow

```text
Customer Checkout
       ↓
POS
       ↓
Validate Stock
       ↓
PostgreSQL Transaction
       │
       ├── Sale
       ├── Sale Items
       ├── Payment
       ├── Stock Movement
       └── Outbox Event
       ↓
     COMMIT
       ↓
Queue Processing
       │
       ├── Analytics
       ├── Notifications
       ├── Cache Invalidation
       └── AI Analysis
```

This is the preferred architecture for asynchronous post-sale processing.

---

# 53. Example — Low Stock Alert

```text
Sale
 ↓
Stock Updated
 ↓
Business Event
 ↓
Queue
 ↓
Low Stock Worker
 ↓
Check Threshold
 ↓
Create Alert
 ↓
Notification Queue
 ↓
Email / In-app Notification
```

The alert does not need to block checkout.

---

# 54. Example — AI Forecast

```text
Scheduled Job
      ↓
BullMQ
      ↓
Forecast Worker
      ↓
Tenant Context
      ↓
Fetch Historical Sales
      ↓
Prepare Dataset
      ↓
AI / Forecast Model
      ↓
Validate Output
      ↓
Store Forecast
      ↓
Cache Result
      ↓
Dashboard
```

---

# 55. Example — Report Generation

```text
User
 ↓
Request Report
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
Query PostgreSQL
 ↓
Generate Report
 ↓
Store File
 ↓
Notification
```

---

# 56. When NOT to Use a Queue

Do not queue operations that require an immediate response unless asynchronous behavior is acceptable.

Examples:

```text
Login
Simple product lookup
POS cart interaction
Stock validation during checkout
Payment confirmation validation
Simple CRUD operations
```

These should normally execute synchronously.

---

# 57. When NOT to Use Redis

Do not introduce Redis simply because the application has Redis available.

Avoid caching:

```text
Rarely accessed data
Highly volatile data with no performance benefit
Critical state without invalidation strategy
Large datasets that exceed reasonable memory budgets
```

Every cache should have a clear reason.

---

# 58. Initial Queue Architecture

For the first implementation:

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

Workers can initially run within the same deployment environment while maintaining clear module boundaries.

They can later be separated into independent services if workload requires it.

---

# 59. Future Scaling

Initial architecture:

```text
Application
    │
    ├── PostgreSQL
    ├── Redis
    └── Workers
```

Future architecture:

```text
                    Load Balancer
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
         API Instances        Web Instances
              │
              ├──────── PostgreSQL
              │
              └──────── Redis
                         │
                       BullMQ
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      AI Workers    Report Workers   Notification
                                      Workers
```

The modular monolith can evolve without immediately becoming microservices.

---

# 60. Failure Scenarios

## Redis Down

```text
Redis unavailable
      ↓
Use PostgreSQL for cacheable reads
```

Business operations should continue where possible.

---

## Worker Down

```text
API continues
      ↓
Jobs remain queued
      ↓
Worker restarts
      ↓
Processing resumes
```

---

## AI Provider Down

```text
AI job fails
      ↓
Retry
      ↓
If still failing
      ↓
Failed job
      ↓
Business system remains operational
```

---

## Email Provider Down

```text
Email job
   ↓
Retry
   ↓
Queue remains available
   ↓
Business transaction unaffected
```

---

# 61. Security Requirements

Redis and queues must follow the security architecture.

### Redis

* [ ] Protected network access
* [ ] Authentication where applicable
* [ ] Tenant-aware keys
* [ ] Sensitive data minimization
* [ ] TTLs
* [ ] No business source of truth

### BullMQ

* [ ] Tenant-aware jobs
* [ ] Payload validation
* [ ] Worker isolation
* [ ] Idempotency
* [ ] Retry controls
* [ ] Sensitive payload minimization

---

# 62. Testing Strategy

Test:

### Cache

* Cache hit
* Cache miss
* Cache invalidation
* Expired cache
* Redis unavailable
* Tenant isolation

### Queue

* Job creation
* Job processing
* Retry
* Failure
* Idempotency
* Worker restart
* Tenant isolation

### Integration

Test:

```text
Database
+
Redis
+
BullMQ
+
Application
```

---

# 63. Definition of Done

The caching and queue architecture is complete when:

* [ ] PostgreSQL remains source of truth
* [ ] Redis responsibilities are clearly defined
* [ ] Cache keys are tenant-aware
* [ ] TTL strategy exists
* [ ] Cache invalidation strategy exists
* [ ] Redis failure behavior is defined
* [ ] BullMQ queues are defined
* [ ] Workers are defined
* [ ] Jobs contain tenant context
* [ ] Jobs are validated
* [ ] Retry strategy exists
* [ ] Idempotency strategy exists
* [ ] Failed jobs are observable
* [ ] Critical workflows do not depend on AI/email/background jobs
* [ ] Transactional outbox is considered for important events
* [ ] Redis and queue security is tested

---

# 64. Final Architecture

Buzzsynx follows this separation:

```text
                    BUSINESS SYSTEM
                          │
             ┌────────────┴────────────┐
             │                         │
             ▼                         ▼
        PostgreSQL                   Redis
       Source of Truth          Cache / Temporary
             │                         │
             │                         │
             └────────────┬────────────┘
                          │
                          ▼
                       BullMQ
                          │
                    Background Jobs
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
       AI            Notifications       Reports
        │                 │                 │
        └─────────────────┼─────────────────┘
                          ▼
                     Observability
```

---

# 65. Final Principle

> **PostgreSQL owns truth. Redis provides speed. BullMQ provides asynchronous execution.**

The system should be designed so that:

```text
Redis failure
    ≠
Business data loss

Worker failure
    ≠
Business transaction failure

AI failure
    ≠
POS failure

Email failure
    ≠
Payment failure
```

Buzzsynx should remain operational even when non-critical supporting systems fail.

> **Use Redis when speed matters.
> Use BullMQ when time does not need to block the user.
> Use PostgreSQL whenever business truth matters.**
