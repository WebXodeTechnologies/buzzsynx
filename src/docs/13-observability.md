# Buzzsynx — Observability

**Document:** Observability Architecture & Standards
**Project:** Buzzsynx
**Version:** 0.2
**Status:** Architecture Specification / Foundation
**Primary Concerns:** Logs, Errors, Metrics, Health, Tracing, Auditability, Alerts
**Architecture:** Multi-Tenant Modular Monolith
**Initial Industry Scope:** Supermarket / Grocery
**Future Scope:** Additional industry capabilities through the shared platform architecture

---

# 1. Purpose

Observability defines how Buzzsynx understands, measures, diagnoses, and operates its own behavior.

Buzzsynx must be able to answer operational questions such as:

* Is the application healthy?
* Is the API responding correctly?
* Which requests are failing?
* Which tenant and store are affected?
* Which business operation failed?
* Is PostgreSQL healthy?
* Is Redis healthy?
* Are background jobs processing?
* Are workers healthy?
* Is an AI provider failing?
* Are payment operations failing?
* Are requests becoming slower?
* Did a deployment introduce the problem?
* What happened before and after the failure?
* Was an important business action performed?
* Did the system recover successfully?

The objective is to make Buzzsynx:

> **Observable, diagnosable, measurable, and operationally trustworthy.**

Observability is an architectural capability, not merely a collection of monitoring tools.

---

# 2. Scope

Buzzsynx is currently designed as a:

> **Multi-Tenant Modular Monolith**

The initial complete business implementation is:

> **Supermarket / Grocery**

Future industries such as pharmacy, clothing, restaurant, electronics, and other business types will reuse the same observability architecture through industry capabilities.

Observability therefore applies to:

* Platform
* Tenant
* Store / Branch
* User
* API
* Database
* Cache
* Queues
* Workers
* POS
* Inventory
* Purchasing
* Payments
* Analytics
* AI
* Notifications
* Reports
* File storage
* Infrastructure

Not every component requires the same level of monitoring.

Critical business paths receive stronger observability than non-critical functionality.

---

# 3. Observability Principles

Buzzsynx follows five primary observability principles.

## 3.1 Logs Explain Events

Logs answer:

> **What happened?**

Example:

```text
sale.completed
inventory.movement.created
payment.failed
worker.job.failed
```

---

## 3.2 Metrics Explain Behavior

Metrics answer:

> **How often, how much, and how quickly is something happening?**

Examples:

```text
API request rate
API error rate
P95 latency
Queue backlog
Database connection usage
AI request count
```

---

## 3.3 Traces Explain Flow

Traces answer:

> **Where did an operation spend time or fail?**

Tracing is particularly useful for complex workflows involving multiple application components or external providers.

---

## 3.4 Health Checks Explain Availability

Health checks answer:

> **Can this application instance currently operate?**

Liveness and readiness must remain distinct.

---

## 3.5 Audit Events Explain Business Actions

Audit events answer:

> **Who performed which important business action, when, and on which resource?**

Audit events are not the same as technical logs.

---

# 4. Observability Architecture

High-level architecture:

```text
                         Buzzsynx
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
        Logs             Metrics           Traces
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                    Observability Layer
                            │
               ┌────────────┴────────────┐
               │                         │
           Dashboards                  Alerts
               │                         │
               └────────────┬────────────┘
                            │
                       Operations
```

Business audit events follow a separate logical path:

```text
Business Operation
        ↓
Audit Event
        ↓
Audit Storage
        ↓
Authorized Investigation
```

The two systems may share infrastructure, but they have different purposes and retention/access requirements.

---

# 5. Observability Layers

Buzzsynx should progressively observe the following layers:

```text
Frontend
   ↓
Nginx / Edge
   ↓
Express API
   ↓
Application Services
   ↓
PostgreSQL
   ↓
Redis
   ↓
BullMQ
   ↓
Workers
   ↓
External Providers
   ├── AI
   ├── Payment
   ├── Email / WhatsApp
   └── File Storage

Infrastructure
   ├── Docker
   ├── Host
   └── AWS
```

Observability should be introduced according to operational importance rather than attempting to monitor every internal detail from day one.

---

# 6. Application Logging

Buzzsynx should use structured logging rather than relying primarily on unstructured console output.

The current backend stack uses structured logging through **Pino / pino-http**.

Example:

```json
{
  "level": "info",
  "event": "sale.completed",
  "requestId": "req_123",
  "tenantId": "tenant_456",
  "storeId": "store_001",
  "userId": "user_789",
  "saleId": "sale_001",
  "durationMs": 142,
  "environment": "production",
  "timestamp": "2026-09-25T10:30:00Z"
}
```

Structured logs allow:

* Searching
* Filtering
* Correlation
* Aggregation
* Alerting
* Incident investigation

---

# 7. Log Levels

## DEBUG

Detailed diagnostic information.

Examples:

* Development diagnostics
* Internal execution details
* Cache diagnostics
* Detailed worker information

DEBUG logging should be carefully controlled in production.

---

## INFO

Normal operational events.

Examples:

* Application started
* Database connected
* Worker started
* Sale completed
* Job completed
* Configuration loaded

---

## WARN

Unexpected but recoverable conditions.

Examples:

* Slow request
* Cache fallback
* Retry scheduled
* External provider temporarily unavailable
* Queue backlog increasing
* Non-critical dependency degraded

---

## ERROR

An operation failed and requires investigation.

Examples:

* Database operation failed
* Payment operation failed
* Worker job failed
* External API failure
* AI request failure

---

## FATAL

The process cannot safely continue.

Examples:

* Required configuration missing
* Application initialization failure
* Required database connection unavailable during startup

The application should not generate excessive FATAL logs for ordinary request failures.

---

# 8. Log Context

Important logs should include relevant context.

Typical fields:

```text
timestamp
level
environment
applicationVersion
requestId
correlationId
tenantId
storeId
userId
module
operation
resourceType
resourceId
durationMs
errorCode
message
```

Not every log requires every field.

Only relevant and safe information should be included.

---

# 9. Request ID

Every API request should receive a unique request ID.

Example:

```text
HTTP Request
     ↓
requestId = req_abc123
     ↓
Middleware
     ↓
Controller
     ↓
Service
     ↓
Repository
     ↓
PostgreSQL
     ↓
Response
```

The request ID should be returned to the client where appropriate, allowing support and engineering teams to correlate a reported failure with server-side logs.

Example:

```text
X-Request-Id: req_abc123
```

The exact header convention should remain consistent across the platform.

---

# 10. Correlation Across Async Operations

Request IDs alone are insufficient when work continues asynchronously.

Example:

```text
Sale Request
     ↓
requestId
     ↓
Database Transaction
     ↓
Outbox Event / Async Event
     ↓
jobId
     ↓
Worker
     ↓
Notification / Analytics / AI
```

Relevant identifiers may include:

```text
requestId
correlationId
jobId
tenantId
storeId
saleId
```

A correlation ID can connect the entire business operation while a request ID identifies an individual HTTP request.

---

# 11. Tenant and Store-Aware Observability

Buzzsynx is multi-tenant and supports multiple stores / branches.

Observability must therefore preserve tenant and store context where appropriate.

Example:

```text
tenantId
storeId
userId
requestId
operation
resourceId
```

This allows operational questions such as:

> Which tenant experienced the problem?

> Which store was affected?

> Is the problem isolated to one store or affecting the entire platform?

Tenant and store context must never be used to bypass authorization.

Operational dashboards must enforce appropriate access controls.

---

# 12. Platform vs Tenant Observability

There are two distinct observability perspectives.

## Platform Observability

Authorized platform operators may monitor:

* Overall API health
* Infrastructure health
* Database health
* Redis health
* Queue health
* Worker health
* Aggregate error rates
* Aggregate platform usage
* Security events

---

## Tenant Observability

Authorized tenant users may view appropriate business information for their own tenant and permitted stores.

Examples:

* Sales metrics
* Inventory events
* Payment failures
* Reports
* AI usage
* Business alerts

Tenant users must not access another tenant's operational or business data.

---

# 13. Sensitive Data Protection

Logs are operational data and must be treated as potentially sensitive.

Never log:

* Passwords
* Password hashes
* Access tokens
* Refresh tokens
* API secrets
* Database credentials
* Payment credentials
* Full card information
* Secret keys
* Unnecessary personal information

Avoid:

```text
password=MyPassword123
```

Prefer:

```text
authentication.failed
```

with safe contextual information.

Pino redaction should be configured for known sensitive fields.

---

# 14. Error Tracking

Application errors should be centrally captured.

A tool such as **Sentry** may be introduced for application error tracking.

The architecture should support capturing:

* Error type
* Error message
* Stack trace
* Environment
* Application version
* Release
* Request ID
* Correlation ID
* Relevant tenant/store context
* User context where appropriate
* Frontend browser context
* Backend context

Sentry is a planned observability component, not assumed to be fully implemented unless explicitly configured.

---

# 15. Error Classification

Errors should be classified consistently.

```text
Validation Error
Authentication Error
Authorization Error
Not Found
Conflict
Business Rule Error
Database Error
External Service Error
Queue Error
AI Error
Payment Error
Infrastructure Error
Unknown Error
```

Stable error codes should be preferred over relying only on human-readable messages.

---

# 16. Business Errors vs System Errors

These must be distinguished.

## Business Error

The system is functioning correctly but the requested operation is not allowed.

Example:

```text
INSUFFICIENT_STOCK
```

---

## System Error

The system itself cannot complete the operation normally.

Example:

```text
DATABASE_UNAVAILABLE
```

This distinction should be reflected in:

* API responses
* Logs
* Error tracking
* Metrics
* Alerts

---

# 17. API Metrics

The API should eventually expose metrics such as:

* Request count
* Error count
* Error rate
* Response latency
* Requests by route
* Requests by HTTP status
* Authentication failures
* Authorization failures
* Rate-limit events
* Timeout count

Latency should be monitored using percentiles:

```text
P50
P95
P99
```

Average latency alone can hide poor experiences for a subset of requests.

Metrics should avoid unbounded labels such as raw user IDs, request IDs, or arbitrary resource IDs.

---

# 18. Business Metrics

Technical metrics are not enough for Buzzsynx.

Important business metrics include:

* Completed sales
* Failed sales
* Payment failures
* Inventory adjustments
* Purchase/receiving transactions
* Returns
* Product creation
* Customer creation
* AI operations
* Notification operations
* Reports generated
* Background jobs completed/failed

Example:

```text
Sales
 ├── Completed
 └── Failed
```

Business metrics should be tenant/store scoped where meaningful.

However, tenant IDs should be handled carefully in metric systems because high-cardinality metric labels can become expensive and difficult to operate.

For detailed tenant-level analysis, logs or business analytics storage may be more appropriate.

---

# 19. PostgreSQL Observability

PostgreSQL is the authoritative source of business truth.

It should be monitored for:

* Availability
* Connection pool usage
* Query latency
* Slow queries
* Transaction failures
* Deadlocks
* Lock contention
* Database size
* Table growth
* Index usage
* Connection exhaustion
* Migration failures
* Replication status if introduced later

Particular attention should be given to:

```text
Sales
Inventory
Purchasing
Payments
Returns
Analytics queries
```

---

# 20. Database Performance Observability

Potential indicators include:

```text
Query duration
Transaction duration
Rows returned
Rows scanned
Connection wait time
Lock wait time
Deadlocks
```

Slow-query analysis should be used to identify problems such as:

```text
Product Search
     ↓
Poor Query Plan
     ↓
Large Scan
     ↓
Database Load ↑
     ↓
API Latency ↑
```

The objective is not merely to collect query data but to identify performance regressions before they materially affect users.

---

# 21. Redis Observability

Redis is an operational support component, not the source of business truth.

Monitor:

* Connection health
* Memory usage
* Cache hit/miss behavior
* Evictions
* Command latency
* Connection failures
* Key growth
* Rate-limit behavior
* Lock behavior where applicable
* Queue backend health when BullMQ shares the Redis infrastructure

Example relationship:

```text
Cache Hit Rate ↓
       ↓
More PostgreSQL Reads
       ↓
Database Load ↑
       ↓
API Latency ↑
```

Redis monitoring must distinguish ordinary cache activity from BullMQ queue infrastructure.

---

# 22. Redis Failure Behavior

Redis failure must not corrupt core business state.

Expected behavior:

```text
Redis unavailable
      ↓
Cache reads may fall back to PostgreSQL where safe
      ↓
Critical transactions continue through PostgreSQL
```

However:

* BullMQ processing cannot continue normally while its Redis backend is unavailable.
* Queue reliability for important events should be supported by database-backed mechanisms such as a transactional outbox where required.
* Rate limiting behavior should be explicitly defined per endpoint.
* Critical security controls must not silently become ineffective because Redis is unavailable.

Redis should therefore be treated as an acceleration/coordination layer, not the authoritative business database.

---

# 23. BullMQ Observability

BullMQ queues should expose operational information such as:

* Waiting jobs
* Active jobs
* Completed jobs
* Failed jobs
* Delayed jobs
* Retry counts
* Processing duration
* Queue backlog
* Oldest waiting job
* Worker availability

Example:

```text
AI Queue
───────────────
Waiting:    25
Active:      4
Completed: 12450
Failed:     12
Delayed:     3
```

A continuously increasing backlog may indicate:

* Insufficient worker capacity
* Downstream provider latency
* Repeated job failures
* Low concurrency
* Dependency failure

---

# 24. Worker Observability

Workers should record:

```text
worker.started
worker.stopped
job.started
job.completed
job.failed
job.retry
```

Relevant fields:

```text
queue
jobId
jobType
jobVersion
tenantId
storeId
correlationId
durationMs
attempt
errorCode
```

Worker failures must not silently disappear.

Workers should also expose operational health independently from the API process.

---

# 25. Queue Job Context

Background jobs should carry only the minimum required information.

Typical payload:

```text
jobType
jobVersion
tenantId
storeId
referenceId
idempotencyKey
correlationId
```

Do not place:

* Passwords
* API secrets
* Payment credentials
* Large unnecessary objects
* Unnecessary personal information

Workers must validate job payloads before processing.

---

# 26. Async Business Event Observability

For important business events, the preferred reliability model is:

```text
Business Transaction
       ↓
PostgreSQL Transaction
       ↓
Business Data + Outbox Event
       ↓
COMMIT
       ↓
Outbox Publisher
       ↓
BullMQ
       ↓
Worker
```

This prevents a successful business transaction from losing its associated asynchronous event merely because queue submission temporarily failed.

For non-critical best-effort work, direct post-commit enqueueing may be sufficient initially.

The transactional outbox should be introduced where reliability requirements justify it.

---

# 27. POS Observability

POS is a critical business workflow.

Important operational events include:

```text
Product Search
Product Scanned
Stock Validation
Cart Updated
Sale Started
Payment Started
Payment Recorded
Sale Completed
Sale Failed
Inventory Movement Created
Invoice Created
```

A POS transaction should be traceable without logging sensitive payment information.

The observability model should distinguish:

```text
Sale Status
Payment Status
Inventory Status
Invoice Status
```

rather than treating them as one generic status.

---

# 28. POS Transaction Observability

The core POS transaction follows the business architecture:

```text
POS Request
     ↓
Authentication / Authorization
     ↓
Product / Price / Tax Validation
     ↓
Stock Validation
     ↓
PostgreSQL Transaction
     ├── Sale
     ├── Sale Items
     ├── Payment Allocation
     ├── Inventory Movement
     └── Invoice Record
     ↓
COMMIT
     ↓
Async Events
     ├── Analytics
     ├── Cache Invalidation
     ├── Notifications
     ├── PDF
     └── AI
```

The critical transaction should not depend on AI, email, analytics processing, or PDF generation.

Observability must make it clear whether a failure occurred:

* Before transaction
* During transaction
* After commit
* In asynchronous processing

---

# 29. Payment Observability

Payment operations are business-critical.

Monitor:

* Payment attempts
* Successful payments
* Failed payments
* Payment status transitions
* Provider response failures
* Webhook failures
* Webhook replay/idempotency events
* Refund operations
* Provider latency
* Timeout events

External payment providers must not be called inside the core PostgreSQL transaction.

Payment gateway state should be reconciled through verified provider webhooks and idempotent processing.

Never log raw payment credentials or sensitive card information.

---

# 30. Inventory Observability

Inventory uses an authoritative movement ledger.

Observable movement types include:

```text
PURCHASE
SALE
RETURN
TRANSFER
DAMAGE
EXPIRY
ADJUSTMENT
```

Track:

* Inventory movements
* Stock adjustments
* Purchases/receiving
* Sales
* Returns
* Transfers
* Damage
* Expiry
* Negative-stock attempts
* Transaction failures
* Stock reconciliation issues

Important inventory operations should also produce audit events.

---

# 31. Inventory Integrity Monitoring

Observability should help detect unexpected inventory conditions.

Examples:

```text
Stock balance changed
        ↓
Movement recorded
        ↓
Transaction committed
        ↓
Expected balance verified
```

Potential anomalies:

* Unexpected negative stock
* Duplicate movement
* Missing movement
* Failed transaction
* Unusual adjustment frequency
* Cross-store access attempt

Observability should detect these conditions; the database and application transaction rules remain responsible for enforcing inventory correctness.

---

# 32. AI Observability

AI is an intelligence layer, not an authoritative business transaction engine.

AI operations should be monitored for:

* Request count
* Success/failure rate
* Provider
* Model
* Latency
* Token usage where available
* Estimated cost where available
* Timeout events
* Rate limits
* Retry events
* Output validation failures
* Job failures
* Prompt/configuration version

Example:

```text
AI Job
  ↓
Data Retrieval
  ↓
Model Provider
  ↓
Response
  ↓
Validation
  ↓
Insight Storage
```

Failures should be identifiable at each stage.

---

# 33. AI Cost and Usage Monitoring

AI usage may create variable operational cost.

Where supported, monitor:

```text
Requests
Tokens
Estimated Cost
Provider
Model
Feature
Tenant
Store
```

Tenant-level usage data may later support:

* AI quotas
* Subscription limits
* Usage-based billing
* Cost alerts

Detailed tenant-level usage should generally be stored as business/usage data rather than creating excessive high-cardinality infrastructure metrics.

---

# 34. Analytics and Reports Observability

Analytics should primarily operate as a read/aggregation path.

Monitor:

* Dashboard query latency
* Report generation time
* Failed report jobs
* Large-query behavior
* Export failures
* Queue backlog
* Generated file failures

Long-running reports should be asynchronous.

Example:

```text
Report Request
     ↓
Create Job
     ↓
BullMQ
     ↓
Worker
     ↓
Generate Report
     ↓
Store File
     ↓
Notify User
```

A report failure should not affect POS transactions.

---

# 35. Notification Observability

Notifications may include:

* Email
* WhatsApp
* Other supported channels in the future

Track:

```text
notification.created
notification.sent
notification.failed
notification.retry
```

Monitor:

* Provider failures
* Retry counts
* Delivery status where available
* Queue backlog
* Provider latency

Notification failure must not normally fail the originating business transaction.

---

# 36. Health Checks

Buzzsynx should expose health endpoints.

Recommended:

```text
GET /api/health/live
GET /api/health/ready
```

A basic liveness response:

```json
{
  "status": "ok"
}
```

Readiness may evaluate critical dependencies.

Example:

```text
Application
   ├── PostgreSQL ✓
   ├── Redis ✓
   └── Required Configuration ✓
```

Health checks should not expose sensitive infrastructure information publicly.

---

# 37. Liveness vs Readiness

## Liveness

Answers:

> Is the process alive?

```text
GET /api/health/live
```

Liveness should remain lightweight and should not fail merely because a downstream dependency is temporarily unavailable.

---

## Readiness

Answers:

> Can this instance safely receive the traffic it is responsible for?

```text
GET /api/health/ready
```

Example:

```text
Application Process: Running
PostgreSQL: Unavailable
```

The process may be alive but not ready to receive normal business traffic.

---

# 38. Dependency Health

Critical dependencies should have operational health visibility.

```text
PostgreSQL
Redis
BullMQ
AI Provider
Payment Provider
Email Provider
File Storage
```

Dependencies should be classified as:

```text
Critical
Degraded
Optional
```

For example:

| Dependency       | Core POS Impact                     |
| ---------------- | ----------------------------------- |
| PostgreSQL       | Critical                            |
| Redis            | Should not be authoritative         |
| BullMQ           | Async functionality degraded        |
| AI Provider      | AI functionality degraded           |
| Email Provider   | Notification functionality degraded |
| Payment Provider | Payment method may be unavailable   |
| File Storage     | Reports/files may be degraded       |

This classification should be reflected in alerts and degraded-mode behavior.

---

# 39. Graceful Degradation

Not every dependency failure should bring down Buzzsynx.

Example:

```text
AI Provider Down
       ↓
AI insights unavailable
       ↓
Core POS continues
```

Similarly:

```text
Email Provider Down
       ↓
Email job retries
       ↓
Business transaction remains successful
```

However, degradation must never bypass core business integrity or security controls.

---

# 40. Distributed Tracing

Buzzsynx starts as a modular monolith, so full distributed tracing is not mandatory from day one.

Tracing should be introduced progressively when application complexity or production diagnosis justifies it.

Potential future trace:

```text
Frontend
   ↓
Nginx
   ↓
Express
   ↓
Service
   ↓
PostgreSQL
   ↓
Redis
   ↓
BullMQ
   ↓
Worker
   ↓
External Provider
```

**OpenTelemetry** can be considered as the standardized tracing/instrumentation layer when needed.

---

# 41. Trace Example

A future sale trace might resemble:

```text
Trace: sale_abc123

API Request              12ms
 ├── Authentication       3ms
 ├── Tenant Context       2ms
 ├── Product Validation   8ms
 ├── Stock Validation    12ms
 ├── DB Transaction      25ms
 └── Response             4ms
```

External payment-provider interaction should be represented separately because it is outside the PostgreSQL transaction.

Tracing should help identify slow components without exposing sensitive data.

---

# 42. Frontend Observability

Frontend monitoring should eventually capture:

* JavaScript errors
* Route errors
* Failed API requests
* Client-side crashes
* Important user-flow failures
* Page performance

Important flows include:

```text
Login
Dashboard
Product Search
POS
Payment
Inventory
Reports
AI
```

Frontend observability should include release/version information so regressions can be associated with deployments.

---

# 43. Nginx Observability

Nginx should provide operational visibility into:

* Request count
* HTTP status
* Request latency
* Upstream failures
* Connection errors
* Rate limiting
* TLS/HTTPS problems

Nginx logs should contain enough information to correlate requests with the application request ID where practical.

---

# 44. Container and Infrastructure Observability

Target deployment components may include:

```text
Next.js
Express
Worker
Nginx
PostgreSQL
Redis
```

Depending on deployment architecture, PostgreSQL and Redis may eventually be managed services rather than application containers.

Monitor:

* Container/process health
* Restart count
* CPU
* Memory
* Disk
* Network
* Health status
* Startup failures
* Resource exhaustion

Repeated restarts should be investigated and may trigger alerts.

---

# 45. AWS Observability

AWS is the target deployment environment.

Cloud infrastructure observability may include:

* Compute health
* Load balancer health
* Network errors
* Storage usage
* Database health
* Redis health
* Container/process health
* Deployment events
* Resource utilization

**AWS CloudWatch** can be used for infrastructure-level monitoring where appropriate.

CloudWatch is a target tooling choice, not an assumption that the entire AWS monitoring stack is already implemented.

---

# 46. Alerting Strategy

Alerts should be actionable.

Bad:

> Something happened.

Better:

> Production API 5xx error rate exceeded the configured threshold for the evaluation period.

Potential critical alerts:

* API unavailable
* PostgreSQL unavailable
* Database connection exhaustion
* Severe API error rate
* Critical payment failure pattern
* Cross-tenant security event
* Critical worker failure
* Disk exhaustion
* Queue backlog exceeding operational limits

Potential warnings:

* Increasing queue backlog
* Increased API latency
* Increased database latency
* Redis memory pressure
* AI provider failures
* Worker retry rate increasing
* Resource utilization approaching limits

Actual thresholds should be established from real production baselines.

---

# 47. Alert Fatigue

Not every anomaly should generate an alert.

An alert should generally be:

```text
Actionable
+
Relevant
+
Meaningful
```

If an event does not require immediate investigation, it may be better represented as:

* Dashboard information
* Log
* Metric
* Scheduled report

Alert thresholds should be reviewed as the system matures.

---

# 48. Audit Logs vs Application Logs

These systems serve different purposes.

## Application Log

Technical event:

```text
inventory.service.updateStock failed
```

Purpose:

> Help engineers understand system behavior.

---

## Audit Event

Business event:

```text
User adjusted inventory for Product X
```

Purpose:

> Help authorized users investigate important business actions.

Audit logs should be generated for security-sensitive and business-critical actions.

---

# 49. Audit Event Structure

A business audit event may contain:

```json
{
  "tenantId": "tenant_123",
  "storeId": "store_001",
  "userId": "user_456",
  "action": "INVENTORY_ADJUSTED",
  "resourceType": "PRODUCT",
  "resourceId": "product_789",
  "requestId": "req_123",
  "timestamp": "2026-09-25T10:30:00Z"
}
```

For sensitive business changes, additional information may include:

```text
previousValue
newValue
reason
```

Only necessary information should be retained.

---

# 50. Important Auditable Business Actions

Examples include:

```text
User / Membership Changes
Role / Permission Changes
Product Changes
Price Changes
Inventory Adjustments
Purchasing / Receiving
Sales
Returns
Refunds
Payment State Changes
Store Configuration Changes
Tenant Configuration Changes
Sensitive Administrative Actions
```

Not every read operation requires an audit event.

Audit requirements should be based on business importance, security risk, and compliance requirements.

---

# 51. Performance Baselines

Before production release, establish baseline measurements for critical workflows.

Examples:

```text
Login
Product Search
Dashboard Load
POS Product Search
POS Sale
Inventory Update
Purchase / Receiving
Report Generation
AI Insight Generation
```

Baselines should be based on realistic application behavior and representative workloads.

They should be used to identify regressions rather than treated as permanent universal limits.

---

# 52. Slow Request Detection

Buzzsynx should identify unusually slow requests.

A slow-request log may include:

```text
endpoint
durationMs
requestId
tenantId
storeId
userId
database duration
errorCode
```

Example:

```text
Normal
   ↓
Expected latency

Slow
   ↓
Above operational threshold

Critical
   ↓
Severe or sustained latency
```

Thresholds should be defined from actual production behavior.

---

# 53. Deployment Observability

Every deployment should be identifiable.

Operational signals should include:

```text
applicationVersion
release
commitSha
environment
deploymentTimestamp
```

Example:

```text
Release: v0.5.0
Commit: abc123
Environment: production
```

This allows incident investigation to determine whether a problem appeared after a particular deployment.

---

# 54. Release Correlation

After deployment, monitor:

```text
Previous Baseline
       ↓
Deployment
       ↓
Error Rate
       ↓
Latency
       ↓
Database Errors
       ↓
Queue Health
       ↓
Business Metrics
```

A significant regression after deployment should be detectable through observability signals.

---

# 55. Incident Investigation Flow

When a production issue occurs:

```text
Alert
  ↓
Identify affected component
  ↓
Identify time window
  ↓
Find request / correlation ID
  ↓
Inspect logs
  ↓
Inspect metrics
  ↓
Inspect traces where available
  ↓
Inspect database / queue state
  ↓
Identify root cause
  ↓
Mitigate
  ↓
Verify recovery
  ↓
Document incident
  ↓
Improve system
```

The goal is to move from:

> “Something is broken.”

to:

> “This specific component failed during this specific operation, affecting this scope, after this event.”

---

# 56. Incident Severity

A simple severity model may be used.

## SEV-1

Critical production outage or severe security/data-integrity incident.

## SEV-2

Major business functionality significantly affected.

## SEV-3

Limited functionality affected with an available workaround.

## SEV-4

Minor issue or low-impact defect.

Severity should be determined by actual business and operational impact.

---

# 57. Observability Data Retention

Observability data should have defined retention policies.

Different data categories may require different retention:

```text
Application Logs
Metrics
Traces
Audit Events
Security Events
Payment Events
```

Retention decisions should balance:

* Debugging requirements
* Security
* Privacy
* Storage cost
* Business requirements
* Applicable legal/compliance requirements

Audit data should not automatically use the same retention policy as technical logs.

---

# 58. Observability Security

Observability infrastructure itself must be secured.

Requirements include:

* Restrict access
* Use role-based access
* Protect tenant-level dashboards
* Encrypt data where appropriate
* Mask sensitive values
* Avoid public exposure of monitoring systems
* Protect monitoring credentials
* Define retention
* Review third-party observability access
* Prevent cross-tenant data exposure

Observability must never become an unintended data-leakage channel.

---

# 59. Development Environment

Observability should be implemented progressively during development.

Development should already use:

* Structured logs
* Request IDs
* Error handling
* Database diagnostics
* Queue diagnostics
* Worker logging
* AI operation logging where applicable

Production observability should therefore be an extension of development practices rather than a separate last-minute project.

---

# 60. Environment Separation

Observability must distinguish:

```text
development
staging
production
```

Every event should identify its environment.

Example:

```text
environment=production
```

Production alerts must not be confused with development or staging events.

Separate credentials, dashboards, and alert policies should be used where practical.

---

# 61. Recommended Tooling

The initial tooling should remain intentionally simple.

## Application Logging

**Pino / pino-http**

Already aligned with the backend architecture.

---

## Application Error Monitoring

**Sentry**

Target option for centralized application/frontend error tracking.

---

## Infrastructure Monitoring

**AWS CloudWatch**

Target option for AWS infrastructure and service monitoring.

---

## Metrics

Application and infrastructure metrics.

A dedicated metrics stack can be introduced when operational scale justifies it.

---

## Tracing

**OpenTelemetry**

Target option when tracing complexity justifies standardized instrumentation.

---

## Queue Monitoring

BullMQ-compatible monitoring and operational dashboards.

Tooling may evolve without changing the underlying observability architecture.

---

# 62. Observability and Caching / Queues

Observability must align with the caching and asynchronous architecture.

Core principle:

```text
PostgreSQL
    ↓
Business Truth

Redis
    ↓
Cache / Temporary State / Coordination

BullMQ
    ↓
Asynchronous Execution
```

Therefore:

```text
Redis Failure
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
```

Observability should make these boundaries visible.

---

# 63. Critical Event Flow

For an important business event:

```text
Business Request
      ↓
PostgreSQL Transaction
      ↓
Business Data Updated
      ↓
Audit / Outbox Event
      ↓
COMMIT
      ↓
Async Processing
      ↓
BullMQ
      ↓
Worker
      ↓
Analytics / AI / Notification / Cache Invalidation
```

Observability should allow engineers to determine which stage failed.

---

# 64. Failure Scenario: Redis Unavailable

Expected operational visibility:

```text
Redis
  ↓
Unavailable

Cache
  ↓
Fallback where safe

PostgreSQL
  ↓
Core transactions continue

BullMQ
  ↓
Queue processing degraded
```

The incident should be visible through:

* Redis health
* Cache behavior
* Queue health
* Worker health
* API latency

---

# 65. Failure Scenario: AI Provider Unavailable

Expected behavior:

```text
AI Provider
     ↓
Unavailable
     ↓
AI Job Retry
     ↓
Failure / Delayed Job
```

Meanwhile:

```text
POS
Inventory
Purchasing
Sales
```

continue according to their normal workflows.

AI failure should be observable without creating false POS failure alerts.

---

# 66. Failure Scenario: Payment Provider Failure

Example:

```text
Payment Attempt
      ↓
Provider Failure
      ↓
Payment Status Updated
      ↓
Retry / Reconciliation
```

The system should distinguish:

```text
Business Sale Failure
Payment Failure
Provider Failure
Webhook Failure
```

rather than reporting all of them as one generic application error.

---

# 67. Failure Scenario: Worker Failure

Example:

```text
Business Transaction
      ↓
COMMIT
      ↓
Async Job
      ↓
Worker Failure
```

The business transaction remains committed.

The queue system should provide:

```text
Retry
Failure visibility
Job ID
Attempt count
Error information
```

For important business events, the database-backed outbox provides an additional reliability boundary.

---

# 68. Multi-Store Observability

Because a tenant may contain multiple stores:

```text
Tenant
 ├── Store A
 ├── Store B
 └── Store C
```

Operational and business events should include `storeId` when the event is store-specific.

Example:

```text
tenantId = tenant_001
storeId  = store_002
saleId   = sale_123
```

This allows:

* Store-specific troubleshooting
* Store-specific business metrics
* Store-specific inventory analysis
* Store-specific operational alerts

Cross-store aggregation should remain tenant-scoped.

---

# 69. Observability for Security Events

Security-sensitive events should be observable.

Examples:

```text
Authentication failure
Authorization failure
Cross-store access attempt
Cross-tenant access attempt
Rate-limit violation
Invalid webhook signature
Repeated token failures
Suspicious administrative action
```

Security events should be handled separately from ordinary application noise where appropriate.

Sensitive security data must still be protected.

---

# 70. Observability for Industry Capabilities

The observability architecture should support future industry capabilities without creating separate monitoring architectures.

Example:

```text
Shared Observability
       ↓
Common Business Engine
       ↓
Industry Capability
       ├── Supermarket
       ├── Pharmacy
       ├── Clothing
       └── Restaurant
```

The initial implementation focuses on supermarket/grocery.

Future industries should reuse:

* Request correlation
* Tenant/store context
* Logs
* Metrics
* Audit
* Queue monitoring
* Error handling
* Health checks

Industry-specific events can be added where necessary.

---

# 71. Avoiding Observability Over-Engineering

Buzzsynx should not attempt to implement every observability technology immediately.

Initial priorities:

```text
1. Structured Logs
2. Request IDs
3. Error Handling
4. Health Checks
5. Database Monitoring
6. Redis Monitoring
7. Queue / Worker Monitoring
8. Critical Business Audit
9. Basic Application Metrics
10. Production Alerts
```

Later, when justified:

```text
Sentry
Advanced Metrics Platform
OpenTelemetry
Distributed Tracing
Advanced Dashboards
Advanced Cost Monitoring
```

The objective is useful observability, not maximum tooling.

---

# 72. Testing Observability

Observability itself should be tested.

## Logging

* Structured log generation
* Request ID propagation
* Sensitive-field redaction
* Error logging

## Tenant / Store Isolation

* Correct tenant context
* Correct store context
* No cross-tenant operational exposure
* No cross-store exposure

## Health

* Liveness behavior
* Readiness behavior
* Dependency failure behavior

## Queues

* Job creation
* Job failure
* Retry
* Duplicate job
* Worker restart
* Queue backlog

## Audit

* Important business actions create audit events
* Correct user context
* Correct tenant/store context
* Protected audit access

## Critical Workflows

* POS transaction rollback
* Payment failure
* Inventory failure
* Async event failure
* Outbox replay where implemented

---

# 73. Observability Definition of Done

Observability architecture is considered ready for production when the following requirements are implemented according to the project's current deployment stage:

### Logging

* [ ] Structured application logging
* [ ] Consistent log levels
* [ ] Request IDs
* [ ] Correlation IDs where needed
* [ ] Tenant/store context where appropriate
* [ ] Sensitive-field redaction
* [ ] Release/environment information

### Error Monitoring

* [ ] Centralized backend error handling
* [ ] Frontend error monitoring where deployed
* [ ] Error classification
* [ ] Stable error codes
* [ ] Production error visibility

### Metrics

* [ ] API request metrics
* [ ] API error metrics
* [ ] Latency metrics
* [ ] Database metrics
* [ ] Redis metrics
* [ ] Queue metrics
* [ ] Worker metrics
* [ ] Critical business metrics

### Health

* [ ] Liveness endpoint
* [ ] Readiness endpoint
* [ ] PostgreSQL health
* [ ] Redis health
* [ ] Worker health
* [ ] Critical dependency visibility

### POS / Inventory / Payments

* [ ] POS failures traceable
* [ ] Inventory movements observable
* [ ] Payment failures observable
* [ ] Webhook failures observable
* [ ] Critical transactions auditable

### Async Processing

* [ ] Queue backlog visible
* [ ] Failed jobs visible
* [ ] Retry behavior visible
* [ ] Worker failures visible
* [ ] Critical async events recoverable

### Security / Audit

* [ ] Important business actions audited
* [ ] Tenant context captured
* [ ] Store context captured where relevant
* [ ] Audit access protected
* [ ] Security events observable

### Deployment

* [ ] Release/version identifiable
* [ ] Commit SHA identifiable
* [ ] Environment identifiable
* [ ] Deployment-related regressions detectable

### Operations

* [ ] Critical alerts configured
* [ ] Incident investigation workflow defined
* [ ] Retention policies defined
* [ ] Observability access controlled

This checklist describes production readiness requirements; it does not imply that every item is already implemented.

---

# 74. Final Observability Model

Buzzsynx observability can be represented as:

```text
                         OBSERVABILITY
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
      Logs                 Metrics               Traces
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                       Health Checks
                              │
                       Audit Events
                              │
                        Alerting
                              │
                         Operations
```

Together these provide answers to:

```text
What happened?
      ↓
How often?
      ↓
How long did it take?
      ↓
Where did it fail?
      ↓
Is the system healthy?
      ↓
Which tenant/store was affected?
      ↓
Which business operation was affected?
      ↓
Who performed the business action?
      ↓
Did the system recover?
```

---

# 75. Final Principle

> **PostgreSQL owns business truth. Observability provides visibility into that truth and the systems operating around it.**

Buzzsynx observability is not simply about collecting logs.

It creates an operational feedback loop:

```text
System
  ↓
Generate Signals
  ↓
Collect
  ↓
Correlate
  ↓
Analyze
  ↓
Alert
  ↓
Investigate
  ↓
Mitigate
  ↓
Verify
  ↓
Improve
```

The ultimate goal is:

> **When something goes wrong, we should be able to determine what happened, where it happened, which tenant/store and business operation were affected, why it happened, what recovered successfully, and what still requires action.**

**Buzzsynx — Observable by design, diagnosable in production, and trustworthy at scale.**
