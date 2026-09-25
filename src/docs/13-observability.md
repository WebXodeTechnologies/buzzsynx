# Buzzsynx — Observability

**Document:** Observability Architecture & Standards
**Project:** Buzzsynx
**Version:** 1.0
**Status:** Architecture & Development
**Primary Concerns:** Logs, Errors, Metrics, Health, Tracing, Auditability, Alerts
**Architecture:** Multi-Tenant Modular Monolith

---

# 1. Purpose

Observability defines how Buzzsynx understands its own behavior.

A production system should not only execute business operations. It should also provide enough information to answer:

* Is the application healthy?
* Is the API responding correctly?
* Which requests are failing?
* Which tenant is affected?
* Which business operation failed?
* Is PostgreSQL healthy?
* Is Redis healthy?
* Are background jobs processing?
* Is the AI provider failing?
* Are payments failing?
* Are requests becoming slower?
* Did a deployment introduce an issue?
* What happened before a failure occurred?

The goal is to make Buzzsynx:

> **Observable, diagnosable, measurable, and operationally trustworthy.**

---

# 2. Observability Principles

Buzzsynx follows five primary observability principles.

### 1. Logs explain events

Logs answer:

> **What happened?**

### 2. Metrics explain behavior

Metrics answer:

> **How often and how much is happening?**

### 3. Traces explain request flow

Traces answer:

> **Where did the request spend time or fail?**

### 4. Health checks explain availability

Health checks answer:

> **Is the system currently capable of operating?**

### 5. Audit logs explain business actions

Audit logs answer:

> **Who performed which important business action?**

These concerns should remain related but should not be treated as the same system.

---

# 3. Observability Architecture

The high-level architecture is:

```text
                    Buzzsynx
                       │
          ┌────────────┼────────────┐
          │            │            │
        Logs        Metrics       Traces
          │            │            │
          └────────────┼────────────┘
                       │
                Observability
                   Platform
                       │
             ┌─────────┴─────────┐
             │                   │
          Dashboard            Alerts
             │                   │
             └─────────┬─────────┘
                       │
                  Operations
```

Business audit events follow a separate path:

```text
Business Operation
       ↓
Audit Event
       ↓
Audit Storage
       ↓
Business Investigation
```

---

# 4. Observability Components

Buzzsynx should eventually observe the following layers:

```text
Frontend
Backend API
Database
Redis
BullMQ
Workers
AI Providers
Payment Providers
File Storage
Nginx
Docker
Host / Cloud Infrastructure
```

Each layer should expose appropriate operational information.

---

# 5. Application Logging

Buzzsynx should use **structured logging** instead of relying primarily on unstructured console messages.

Example:

```json
{
  "level": "info",
  "event": "sale.created",
  "requestId": "req_123",
  "tenantId": "tenant_456",
  "userId": "user_789",
  "saleId": "sale_001",
  "durationMs": 142,
  "timestamp": "2026-09-25T10:30:00Z"
}
```

Structured logs make searching, filtering, aggregation, and alerting significantly easier.

---

# 6. Log Levels

Recommended levels:

## DEBUG

Detailed development information.

Examples:

* Internal state
* Development diagnostics
* Cache operations

Should generally be reduced or disabled in production.

## INFO

Normal operational events.

Examples:

* Server started
* User authenticated
* Sale completed
* Background job completed
* Application connected to database

## WARN

Unexpected but recoverable situations.

Examples:

* Cache miss
* AI provider temporarily unavailable
* Retry scheduled
* Slow request
* Low queue capacity

## ERROR

Operation failed and requires investigation.

Examples:

* Database operation failed
* Payment processing failed
* Worker failed
* External API failure

## FATAL

Application or infrastructure condition that prevents normal operation.

Examples:

* Application cannot initialize
* Required configuration missing
* Database unavailable during startup

---

# 7. Log Context

Important application logs should include appropriate context.

Recommended fields:

```text
timestamp
level
service
environment
requestId
userId
tenantId
role
module
operation
resourceId
durationMs
errorCode
message
```

Not every log needs every field.

Only relevant context should be included.

---

# 8. Request ID

Every API request should have a unique request identifier.

Example:

```text
Request
   ↓
requestId = req_abc123
   ↓
Controller
   ↓
Service
   ↓
Repository
   ↓
Database
```

If the request fails, the same ID should help correlate related logs.

Example:

```text
req_abc123

API Request
   ↓
Inventory Service
   ↓
PostgreSQL
   ↓
Stock Movement
   ↓
Response
```

This makes debugging significantly easier.

---

# 9. Correlation IDs

For asynchronous operations, request IDs alone are not sufficient.

Example:

```text
Sale Request
   ↓
requestId
   ↓
SALE_CREATED
   ↓
jobId
   ↓
Worker
```

The job should retain appropriate correlation information.

Example:

```text
requestId
tenantId
saleId
jobId
```

This allows an engineer to follow an operation across synchronous and asynchronous systems.

---

# 10. Tenant-Aware Observability

Because Buzzsynx is multi-tenant, observability must preserve tenant context where appropriate.

Example:

```text
tenantId
userId
requestId
operation
resourceId
```

This allows operational questions such as:

> Which tenant experienced this failure?

or:

> Are errors isolated to one tenant or affecting the entire platform?

However, tenant information must not become a mechanism for exposing sensitive tenant data to unauthorized users.

Operational dashboards must follow appropriate access controls.

---

# 11. Sensitive Data Protection

Logs must not contain unnecessary sensitive information.

Never log:

* Passwords
* Password hashes
* Authentication secrets
* API keys
* Access tokens
* Refresh tokens
* Payment credentials
* Full card information
* Sensitive personal information unnecessarily
* Database credentials

Instead of:

```text
password=MyPassword123
```

the log should contain:

```text
authentication.failed
```

with appropriate non-sensitive context.

---

# 12. Error Tracking

Application errors should be captured centrally.

A tool such as **Sentry** can be used for application-level error monitoring.

The error system should capture:

* Error message
* Stack trace
* Environment
* Release/version
* Request context
* Relevant tenant context
* User context where appropriate
* Browser information for frontend errors
* Server context for backend errors

---

# 13. Error Classification

Errors should be categorized.

Example:

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

This makes operational analysis easier.

---

# 14. Business Errors vs System Errors

These should not be treated identically.

### Business Error

Example:

```text
Insufficient stock
```

The system is working correctly but the requested operation is not allowed.

### System Error

Example:

```text
PostgreSQL connection failed
```

The system itself is experiencing a failure.

This distinction should exist in:

* API responses
* Logs
* Error tracking
* Metrics
* Alerts

---

# 15. API Metrics

The API should expose metrics such as:

* Request count
* Error count
* Error rate
* Response latency
* Requests by endpoint
* Requests by HTTP status
* Requests by method
* Authentication failures
* Authorization failures
* Rate-limit events

Important latency measurements include:

```text
Average
P50
P95
P99
```

Percentiles are more useful than averages for identifying slow requests experienced by a subset of users.

---

# 16. Business Metrics

Technical metrics alone are insufficient for Buzzsynx.

Important business metrics include:

* Sales completed
* Sales failed
* Payment failures
* Inventory adjustments
* Purchase transactions
* Product creation
* Customer creation
* AI requests
* AI failures
* Background jobs completed
* Background jobs failed
* Notifications sent
* Reports generated

Example:

```text
Sales
  ↓
Completed: 1,245
Failed: 8
Failure Rate: monitored
```

Business metrics should be tenant-aware where appropriate.

---

# 17. Database Observability

PostgreSQL should be monitored for:

* Connection availability
* Connection pool usage
* Query latency
* Slow queries
* Transaction failures
* Deadlocks
* Lock contention
* Database size
* Table growth
* Index usage
* Failed migrations
* Connection exhaustion

Particular attention should be given to:

```text
Sales
Inventory
Payments
Orders
Analytics
```

because these are business-critical areas.

---

# 18. Database Query Performance

Slow database queries should be detectable.

Potential indicators:

```text
query duration
rows scanned
rows returned
index usage
connection wait time
transaction duration
```

Frequently executed queries should be reviewed periodically.

Potential problem:

```text
GET /products
      ↓
Full table scan
      ↓
100,000 products
      ↓
Slow response
```

The observability system should help identify this before it becomes a serious production problem.

---

# 19. Redis Observability

Redis should be monitored for:

* Connection status
* Memory usage
* Cache hit rate
* Cache miss rate
* Evictions
* Command latency
* Connection failures
* Key growth
* Rate-limit behavior
* Lock behavior where applicable

Example:

```text
Cache Hit Rate ↓
       ↓
More PostgreSQL Queries
       ↓
Database Load ↑
       ↓
API Latency ↑
```

Observability should allow this relationship to be diagnosed.

---

# 20. BullMQ Observability

Queues should expose:

* Waiting jobs
* Active jobs
* Completed jobs
* Failed jobs
* Delayed jobs
* Retry counts
* Processing duration
* Queue backlog
* Worker availability

Example:

```text
AI Queue
───────────────
Waiting: 25
Active: 4
Completed: 12,450
Failed: 12
Delayed: 3
```

A continuously increasing queue backlog may indicate:

* Worker capacity is insufficient.
* External provider is slow.
* Jobs are failing repeatedly.
* Concurrency is too low.
* A downstream dependency is unavailable.

---

# 21. Worker Observability

Workers should expose:

* Worker startup
* Worker shutdown
* Job started
* Job completed
* Job failed
* Retry
* Processing duration
* Queue name
* Job ID
* Tenant ID where appropriate

Example:

```text
job.started
job.completed
job.failed
job.retry
```

Worker failures must not silently disappear.

---

# 22. AI Observability

AI operations require dedicated monitoring.

Track:

* AI requests
* AI success rate
* AI failure rate
* Provider latency
* Token usage where available
* Estimated cost where available
* Model/provider
* Prompt version
* Output validation failures
* Rate limits
* Timeout events
* Retry events

Example:

```text
AI Request
    ↓
Provider
    ↓
Response
    ↓
Validation
    ↓
Business Insight
```

Failures at each stage should be distinguishable.

---

# 23. AI Cost Monitoring

Because AI usage may generate variable costs, Buzzsynx should track AI consumption.

Potential metrics:

```text
Requests / tenant
Tokens / tenant
Estimated cost / tenant
Requests / feature
Cost / feature
Failures / provider
```

This can eventually support:

* Usage limits
* Subscription plans
* AI quotas
* Cost alerts
* Tenant-level billing

---

# 24. Payment Observability

Payment operations are business-critical.

Monitor:

* Payment attempts
* Successful payments
* Failed payments
* Payment verification failures
* Webhook failures
* Refunds
* Duplicate requests
* Provider latency

Payment logs must never expose sensitive payment credentials.

---

# 25. POS Observability

POS should have dedicated operational visibility.

Important events:

```text
Cart Created
Product Added
Stock Validation
Sale Started
Payment Started
Payment Completed
Sale Completed
Sale Failed
Inventory Updated
Invoice Generated
```

A failed POS transaction should be traceable from start to finish.

---

# 26. Inventory Observability

Inventory events should be measurable.

Track:

* Stock adjustments
* Purchases
* Sales
* Returns
* Transfers
* Damage
* Expiry
* Negative-stock attempts
* Inventory transaction failures

Unexpected inventory changes should be detectable and auditable.

---

# 27. Health Checks

Buzzsynx should expose health endpoints.

Example:

```text
GET /api/health
```

A basic health response might indicate:

```json
{
  "status": "ok"
}
```

A deeper readiness check can verify dependencies.

```text
Application
   ├── PostgreSQL ✓
   ├── Redis ✓
   ├── Queue ✓
   └── External Dependencies
```

---

# 28. Liveness vs Readiness

These checks should have different purposes.

## Liveness

Answers:

> Is the process alive?

Example:

```text
GET /health/live
```

## Readiness

Answers:

> Can this instance safely receive traffic?

Example:

```text
GET /health/ready
```

A process may be alive but not ready.

For example:

```text
Application Process: Running
PostgreSQL: Unavailable
```

The process is alive, but it may not be ready to serve business requests.

---

# 29. Dependency Health

Buzzsynx should monitor critical dependencies:

```text
PostgreSQL
Redis
BullMQ
AI Provider
Payment Provider
Email Provider
Object/File Storage
```

Dependency failures should be visible.

The system should also define whether a dependency is:

* Critical
* Degraded
* Optional

---

# 30. Degraded Mode

Not every dependency failure should bring down the entire platform.

Example:

```text
AI Provider Down
       ↓
Core POS continues
       ↓
Sales continue
       ↓
AI insights temporarily unavailable
```

Similarly:

```text
Email Provider Down
       ↓
Business transaction continues
       ↓
Email job retries
```

The architecture should prefer graceful degradation where possible.

---

# 31. Distributed Tracing

As Buzzsynx grows, distributed tracing can help understand request flow across:

```text
Frontend
   ↓
Express
   ↓
Service
   ↓
PostgreSQL
   ↓
Redis
   ↓
Queue
   ↓
Worker
   ↓
External Provider
```

Tracing should be introduced when the system's complexity makes it useful.

OpenTelemetry can be considered for standardized instrumentation.

---

# 32. Trace Example

A sale request could appear as:

```text
Trace: sale_abc123

API Request                  10ms
 ├── Authentication           3ms
 ├── Tenant Resolution        2ms
 ├── Product Validation       8ms
 ├── Stock Validation         12ms
 ├── Payment                 180ms
 ├── Database Transaction     25ms
 └── Response                 4ms
```

This immediately identifies the slowest portion of the workflow.

---

# 33. Frontend Observability

Frontend monitoring should capture:

* JavaScript errors
* Failed API requests
* Page performance
* Route errors
* Client-side crashes
* Important user-flow failures

Particular attention should be given to:

* Login
* Dashboard
* Product search
* POS
* Payment
* Reports
* AI interface

---

# 34. Nginx & Infrastructure Observability

Nginx should expose or log:

* Request count
* HTTP status
* Request latency
* Upstream failures
* Connection errors
* Rate limiting
* TLS/HTTPS issues

Infrastructure should eventually monitor:

* CPU
* Memory
* Disk
* Network
* Container health
* Container restarts

---

# 35. Container Observability

Each container should have predictable behavior.

Example:

```text
next
express
worker
postgres
redis
nginx
```

Monitor:

* Container status
* Restart count
* Resource usage
* Health check status
* Logs
* Startup failures

Repeated container restarts should generate an alert.

---

# 36. Alerting Strategy

Alerts should be actionable.

Bad alert:

> Something happened.

Good alert:

> Production API 5xx rate has exceeded the defined threshold for the configured evaluation period.

Potential alerts:

### Critical

* Application unavailable
* Database unavailable
* High API error rate
* Payment failures above threshold
* Cross-tenant security event
* Critical worker failure
* Disk exhaustion
* Database connection exhaustion

### Warning

* Increasing queue backlog
* Slow API response
* High Redis memory
* AI provider failure rate increasing
* Increased database latency
* High resource utilization

---

# 37. Alert Fatigue

Buzzsynx should avoid generating alerts for every small anomaly.

Alerts should generally satisfy:

```text
Actionable
+
Relevant
+
Meaningful
```

If an alert does not require investigation or action, it may be better represented as a dashboard metric.

---

# 38. Audit Logs vs Application Logs

These systems have different purposes.

### Application Log

Technical event:

```text
inventory.service.updateStock failed
```

### Audit Log

Business event:

```text
User X adjusted inventory for Product Y
```

Application logs help engineers.

Audit logs help businesses and administrators investigate important actions.

Both should exist where required.

---

# 39. Audit Event Structure

A business audit event may contain:

```json
{
  "tenantId": "tenant_123",
  "userId": "user_456",
  "action": "INVENTORY_ADJUSTED",
  "resource": "PRODUCT",
  "resourceId": "product_789",
  "timestamp": "2026-09-25T10:30:00Z"
}
```

Only appropriate business information should be stored.

---

# 40. Observability for Multi-Tenancy

Operational monitoring should support two levels.

## Platform Level

Used by authorized platform operators.

Examples:

* Total API traffic
* Overall error rate
* Infrastructure health
* Queue health
* Database health
* Aggregate system metrics

## Tenant Level

Used by authorized tenant administrators.

Examples:

* Their sales
* Their inventory events
* Their operational errors
* Their AI usage
* Their business metrics

Platform and tenant observability data must remain appropriately isolated.

---

# 41. Performance Baselines

Before production release, establish baseline measurements for important workflows.

Examples:

```text
Login
Product Search
Dashboard Load
POS Product Search
POS Sale
Inventory Update
Purchase Creation
Report Generation
AI Request
```

Baseline measurements provide a reference for detecting regressions.

---

# 42. Slow Request Detection

Buzzsynx should identify requests exceeding defined latency thresholds.

Example:

```text
Normal
< threshold

Slow
threshold → higher threshold

Critical
> higher threshold
```

The actual thresholds should be based on real application behavior rather than arbitrary numbers.

Slow requests should include:

* Endpoint
* Duration
* Request ID
* Tenant ID where appropriate
* Database/query information where available
* Error information if applicable

---

# 43. Deployment Observability

Every deployment should be identifiable.

Logs and errors should include:

```text
application version
release
commit SHA
environment
deployment timestamp
```

This allows questions such as:

> Did this issue start after the latest deployment?

Example:

```text
Release: v1.4.0
Commit: abc123
Environment: production
```

---

# 44. Release Correlation

When a new release is deployed, monitor:

```text
Before Deployment
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

A sudden increase in errors after deployment should be detectable.

---

# 45. Incident Investigation Flow

When a production issue occurs:

```text
Alert
 ↓
Identify affected component
 ↓
Check timestamp
 ↓
Find request / correlation ID
 ↓
Inspect logs
 ↓
Inspect metrics
 ↓
Inspect traces
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
```

---

# 46. Incident Severity

A simple severity model can be used.

### SEV-1

Critical production outage or severe security/data-integrity issue.

### SEV-2

Major business functionality significantly affected.

### SEV-3

Limited functionality affected with available workaround.

### SEV-4

Minor issue or low-impact defect.

Severity should be based on actual business impact.

---

# 47. Observability Data Retention

Observability data should have defined retention policies.

Different data types may require different retention:

```text
Application Logs
Metrics
Traces
Audit Logs
Security Events
Payment Events
```

Retention should balance:

* Debugging needs
* Compliance requirements
* Storage cost
* Privacy
* Business requirements

---

# 48. Privacy

Observability systems themselves contain potentially sensitive information.

Therefore:

* Restrict access.
* Avoid unnecessary personal data.
* Mask sensitive values.
* Encrypt data where appropriate.
* Define retention.
* Review third-party observability access.
* Ensure tenant data is not accidentally exposed through operational dashboards.

---

# 49. Observability During Development

Observability should not be postponed until production.

During development:

* Use structured logs.
* Use request IDs.
* Log important business events.
* Monitor database queries.
* Monitor queue jobs.
* Monitor AI operations.
* Verify error handling.

The goal is to make production observability an extension of development practices rather than a separate project.

---

# 50. Recommended Tooling

Initial tooling can remain simple.

### Application Errors

**Sentry**

### Cloud/Infrastructure Monitoring

**AWS CloudWatch**

### Logs

Structured application logs.

### Metrics

Application and infrastructure metrics.

### Tracing

**OpenTelemetry** when tracing complexity justifies it.

### Queue Monitoring

BullMQ-compatible monitoring where required.

The exact tooling may evolve without changing the observability principles.

---

# 51. Observability Environment Strategy

Different environments should be distinguishable.

```text
development
staging
production
```

Each event should identify its environment.

Example:

```text
environment=production
```

Production alerts must not be confused with development or staging events.

---

# 52. Observability Checklist

## Logging

* [ ] Structured logging
* [ ] Log levels
* [ ] Request IDs
* [ ] Correlation IDs
* [ ] Tenant context
* [ ] Error context
* [ ] Sensitive data filtering

## Error Tracking

* [ ] Backend error tracking
* [ ] Frontend error tracking
* [ ] Release tracking
* [ ] Error classification

## Metrics

* [ ] API metrics
* [ ] Database metrics
* [ ] Redis metrics
* [ ] Queue metrics
* [ ] Worker metrics
* [ ] AI metrics
* [ ] Payment metrics
* [ ] Business metrics

## Health

* [ ] Liveness endpoint
* [ ] Readiness endpoint
* [ ] Database health
* [ ] Redis health
* [ ] Worker health
* [ ] Dependency health

## Tracing

* [ ] Request correlation
* [ ] Async correlation
* [ ] Trace support when required

## Audit

* [ ] Important business actions logged
* [ ] Tenant context captured
* [ ] User context captured
* [ ] Audit access protected

## Alerting

* [ ] Critical alerts
* [ ] Warning alerts
* [ ] Queue alerts
* [ ] Database alerts
* [ ] Application alerts
* [ ] Infrastructure alerts

---

# 53. Definition of Done

Observability is considered complete when:

* [ ] Important application events are logged.
* [ ] Logs are structured and searchable.
* [ ] Requests can be correlated using request IDs.
* [ ] Errors are centrally captured.
* [ ] Critical infrastructure dependencies are monitored.
* [ ] API performance can be measured.
* [ ] Database health can be monitored.
* [ ] Redis health can be monitored.
* [ ] Queue and worker health can be monitored.
* [ ] AI operations can be measured.
* [ ] Payment failures can be identified.
* [ ] Business-critical events can be audited.
* [ ] Health endpoints exist.
* [ ] Critical alerts are configured.
* [ ] Sensitive information is protected.
* [ ] Tenant context is handled safely.
* [ ] Deployment versions can be correlated with incidents.
* [ ] Production issues can be investigated systematically.

---

# 54. Final Observability Model

Buzzsynx should provide visibility across five dimensions:

```text
                    OBSERVABILITY
                         │
        ┌────────────────┼────────────────┐
        │                │                │
       Logs           Metrics           Traces
        │                │                │
        └────────────────┼────────────────┘
                         │
                    Health Checks
                         │
                    Audit Events
```

Together they answer:

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
Who performed the business action?
```

---

# 55. Final Principle

> **If we cannot see it, we cannot reliably operate it.**

Buzzsynx observability is therefore not just about collecting logs.

It is about creating a complete operational feedback loop:

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
Fix
  ↓
Verify
  ↓
Improve
```

The ultimate goal is:

> **When something goes wrong, we should be able to determine what happened, where it happened, which business operation was affected, which tenant was affected, why it happened, and whether the fix actually resolved the problem.**
