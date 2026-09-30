# Buzzsynx — Development Standards

**Document:** Development Standards & Engineering Guidelines
**Project:** Buzzsynx
**Version:** 0.2
**Status:** Architecture-Aligned Development Standard
**Architecture:** Multi-Tenant Modular Monolith
**Initial Product Scope:** Supermarket / Grocery
**Primary Stack:** Next.js, React, Node.js, Express, PostgreSQL, Prisma, Redis, BullMQ
**Engineering Goal:** Maintainable, secure, reliable, observable, production-oriented software

---

# 1. Purpose

This document defines the engineering standards that govern Buzzsynx development.

Its purpose is to ensure that:

* Architecture remains consistent.
* Business logic remains predictable.
* Tenant and store isolation is enforced.
* Security is built into every feature.
* Database integrity is protected.
* AI-assisted development does not introduce architectural shortcuts.
* Background processing remains reliable.
* Observability is built into important workflows.
* Technical debt remains controlled.
* Documentation stays aligned with implementation.
* The codebase remains understandable as Buzzsynx evolves.

This document acts as the project's engineering rulebook.

---

# 2. Current Scope

Buzzsynx is being developed as a **multi-tenant business operations platform**.

The **supermarket/grocery workflow is the initial complete implementation and canonical MVP vertical slice**.

Future industries such as:

* Pharmacy
* Clothing
* Restaurant
* Electronics
* Clinic
* Hardware
* Poultry

may be introduced later through the capability architecture.

They must not be treated as simultaneously completed MVP modules.

The development standard therefore follows:

> **Build the shared core correctly first, complete the supermarket workflow, then extend through capabilities when justified.**

---

# 3. Core Development Philosophy

Buzzsynx follows these principles:

> **Simple before complex.**

> **Explicit before clever.**

> **Business correctness before UI convenience.**

> **Database integrity before application assumptions.**

> **Secure by default.**

> **Tenant isolation by design.**

> **Deterministic logic for critical business operations.**

> **Observable from the beginning.**

> **AI assists development and intelligence; it does not replace engineering authority.**

> **Do not introduce architecture complexity without a real requirement.**

---

# 4. Architecture Standard

Buzzsynx uses a:

> **Multi-Tenant Modular Monolith**

The system is one deployable application architecture with clear business-module boundaries.

Conceptually:

```text
Next.js Frontend
       ↓
Express API
       ↓
Authentication
       ↓
Tenant / Store Context
       ↓
RBAC + Capability Authorization
       ↓
Business Modules
       ↓
PostgreSQL
```

Supporting infrastructure:

```text
Redis
 ├── Cache
 ├── Temporary Data
 ├── Rate Limiting
 └── BullMQ Backend

BullMQ
 └── Background Workers
```

The architecture should remain modular enough that individual modules can evolve independently and, if genuinely required in the future, be extracted into separate services.

Microservices are not the current default.

---

# 5. Initial Business Modules

The current architecture may contain modules such as:

```text
auth
tenants
users
memberships
roles
stores
products
inventory
purchases
suppliers
pos
sales
customers
payments
analytics
reports
ai
notifications
```

The exact module list may evolve.

A module should exist because it represents a meaningful business boundary, not simply because a folder is required.

---

# 6. Module Boundary Rules

Each module should own its relevant business logic.

Example:

```text
Inventory
├── Controller
├── Service
├── Repository
├── Validation
└── Domain/Application Logic
```

Other modules should communicate through defined application/service boundaries.

For example:

```text
POS
 ↓
Inventory Service
 ↓
Inventory Logic
 ↓
Repository
 ↓
PostgreSQL
```

Avoid arbitrary direct manipulation of another module's tables.

### Rule

> **Modules communicate through defined application boundaries, not through uncontrolled database access.**

---

# 7. Layered Responsibility

Where appropriate, backend modules should follow:

```text
Route
 ↓
Middleware
 ↓
Controller
 ↓
Service / Use Case
 ↓
Repository / Data Access
 ↓
PostgreSQL
```

### Route / Middleware

Responsible for:

* Routing
* Authentication
* Tenant/store context
* Permission checks
* Capability checks
* Request-level controls

### Controller

Responsible for:

* Reading request data
* Calling application services
* Mapping results to API responses
* Passing errors to centralized handling

Controllers should remain thin.

### Service / Use Case

Responsible for:

* Business rules
* Workflow orchestration
* Transaction boundaries
* Business validations
* Cross-module coordination

### Repository / Data Access

Responsible for:

* Prisma queries
* Persistence
* Tenant/store-scoped retrieval
* Data access patterns

Repositories should not become a second business-rule engine.

---

# 8. Transaction Ownership

Transaction boundaries should be owned by the application service/use-case that coordinates the business operation.

Example:

```text
createSale()
    ↓
DB Transaction
    ├── Create Sale
    ├── Create Sale Items
    ├── Record Payment Allocations
    ├── Create Stock Movements
    ├── Update Stock Balance
    ├── Create Invoice Record
    └── Create Outbox Event
    ↓
Commit
    ↓
Async Processing
```

Do not scatter unrelated transaction boundaries across controllers and repositories.

External services must not be called from inside critical database transactions.

---

# 9. Business Logic Rules

Critical business logic must have an authoritative implementation.

Avoid duplicating rules across:

* React components
* Controllers
* Services
* Reports
* Workers
* Utility functions

For example, sale totals should have one authoritative business calculation.

```text
Authoritative Sale Calculation
          ↓
     Sale Workflow
       /      \
    Invoice   Reports
```

Reports should consume recorded business data rather than independently inventing a different calculation.

---

# 10. Deterministic Business Logic

Critical business operations must remain deterministic.

AI must not become the authority for:

* Product prices
* Sale totals
* Tax calculations
* Discounts
* Stock quantities
* Payment state
* Invoice numbering
* Authorization
* Financial calculations
* Inventory movements
* Transaction integrity

Example:

```text
Price
 +
Quantity
 -
Discount
 +
Tax
 =
Deterministic Total
```

AI may explain, analyze, predict, summarize, or recommend based on the resulting data.

---

# 11. Multi-Tenancy Standard

Buzzsynx follows:

```text
Super Admin
     ↓
Tenant / Business
     ↓
Store / Branch
     ↓
Membership
     ↓
User
```

A user belongs to a tenant through a membership.

Store access must also be validated where applicable.

The trusted request context should conceptually contain:

```text
userId
tenantId
membershipId
activeStoreId
roles
permissions
capabilities
```

### Critical Rule

> **Never trust tenantId supplied by the client as the source of authorization.**

Tenant context must be derived from authenticated membership and server-side context.

---

# 12. Store Scope

Multi-store support is part of the architecture even though the initial MVP may begin with one store.

Store-owned data must be appropriately scoped.

Examples:

```text
Store-scoped
├── Stock
├── Inventory Movements
├── Sales
├── Purchases / Receiving
├── Payments
├── Registers
└── Store-specific pricing/availability
```

Tenant-wide data may include:

```text
Products
Categories
Brands
Suppliers
Tenant Settings
Memberships
```

The exact scope of each entity must follow the database design.

### Rule

> **Tenant scope and store scope are separate authorization boundaries.**

---

# 13. Tenant-Scoped Data Access

Tenant-owned queries must be tenant-safe.

Conceptually:

```js
where: {
  id: resourceId,
  tenantId
}
```

For store-owned records:

```js
where: {
  id: resourceId,
  tenantId,
  storeId
}
```

Avoid generic unrestricted access such as:

```js
findById(id)
```

for tenant-owned resources unless the calling layer has already established equivalent scope through a trusted, enforceable mechanism.

---

# 14. Cross-Tenant and Cross-Store Security

Every feature must be evaluated for isolation.

Verify:

```text
Can Tenant A read Tenant B data?
Can Tenant A modify Tenant B data?
Can Tenant A delete Tenant B data?

Can Store A access Store B stock?
Can Store A access Store B sales?
Can Store A access Store B reports?

Can a user access a store they are not assigned to?
Can a tenant access another tenant's files?
Can AI receive another tenant's data?
```

Unless explicitly authorized as a platform operation, the answer must be **no**.

---

# 15. Platform Administration

Super Admin is a platform-level role.

Super Admin access must not be treated as an ordinary tenant membership role.

Platform operations should use explicit platform authorization and should be auditable.

Examples:

```text
Tenant suspension
Tenant activation
Capability enablement
Platform configuration
Platform-level support operations
```

Platform access must not accidentally become a mechanism for bypassing audit or security controls.

---

# 16. RBAC Standard

Current business roles are:

```text
Super Admin
Owner
Admin / Manager
Cashier
Accountant
Store Staff
```

Authorization should be permission-based rather than relying only on role names.

Conceptually:

```text
User
 ↓
Membership
 ↓
Role
 ↓
Permissions
```

Frontend role checks are for user experience.

Backend authorization is the actual security boundary.

---

# 17. Capability Standard

Industry capabilities and authorization are different concerns.

A successful operation may require:

```text
Tenant Capability
        +
User Permission
        +
Resource Scope
```

For example:

```text
Tenant Capability
       ↓
EXPIRY_TRACKING

User Permission
       ↓
INVENTORY_VIEW

Store Scope
       ↓
Store A
```

Feature flags, subscription entitlements, capabilities, and permissions should not be treated as interchangeable concepts.

---

# 18. Validation Standard

All external input must be validated.

Sources include:

* Request body
* Query parameters
* Route parameters
* Headers where applicable
* Uploaded files
* Webhooks
* External provider responses

Use the project's validation layer, currently based on **Zod** where appropriate.

Validation should happen before business logic processes untrusted input.

---

# 19. Mass Assignment Protection

Never blindly pass client-provided objects into database updates.

Avoid:

```js
prisma.product.update({
  data: req.body
});
```

Prefer explicit field selection.

The client must not be able to modify protected fields such as:

```text
tenantId
storeId
createdBy
system status
ownership
audit fields
```

unless the operation explicitly allows it.

---

# 20. Error Handling

API errors must be controlled and consistent.

Never expose:

* Prisma internals
* Database credentials
* Stack traces
* Internal service details
* Secrets
* Provider credentials

Example:

```json
{
  "success": false,
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found",
    "requestId": "req_123"
  }
}
```

Internal technical details belong in protected logs.

---

# 21. Error Categories

Use stable error categories/codes where useful:

```text
VALIDATION_ERROR
AUTHENTICATION_ERROR
AUTHORIZATION_ERROR
NOT_FOUND
CONFLICT
BUSINESS_RULE_ERROR
DATABASE_ERROR
EXTERNAL_SERVICE_ERROR
PAYMENT_ERROR
AI_ERROR
QUEUE_ERROR
RATE_LIMIT_ERROR
INTERNAL_ERROR
```

The exact code set may evolve.

Error codes should remain stable enough for frontend behavior and operational monitoring.

---

# 22. API Standards

API contracts should remain consistent.

Preferred versioning:

```text
/api/v1/...
```

Examples:

```text
GET    /api/v1/products
GET    /api/v1/products/:id
POST   /api/v1/products
PATCH  /api/v1/products/:id
```

Store and membership operations should follow the established tenant/store architecture.

Avoid duplicate endpoints representing the same business operation.

For example, POS should not create a separate competing sale implementation if `/sales` is the canonical sale domain.

---

# 23. API Response Standards

Successful responses may follow:

```json
{
  "success": true,
  "data": {},
  "meta": {}
}
```

Errors should follow a consistent structure:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid product data",
    "requestId": "req_123",
    "details": {}
  }
}
```

Do not expose internal implementation details.

---

# 24. Database Standards

PostgreSQL is the authoritative business data store.

Database design must prioritize:

* Referential integrity
* Constraints
* Appropriate indexes
* Transaction safety
* Tenant isolation
* Store isolation
* Data consistency
* Auditability where required

Application logic should not be the only protection for critical invariants when database constraints can enforce them.

---

# 25. Prisma Standards

Prisma is the primary PostgreSQL data-access layer.

Use Prisma consistently with the project's established architecture.

Avoid:

* Scattered raw SQL without justification
* Duplicate data-access patterns
* Unscoped tenant queries
* Generic unrestricted repository methods for tenant-owned data

Raw SQL may be used when there is a documented technical reason, such as a performance-sensitive query or PostgreSQL-specific capability.

---

# 26. Database Naming

Use predictable, descriptive naming.

Examples:

```text
users
tenants
stores
tenant_memberships
membership_stores
products
categories
brands
suppliers
stock_balances
stock_movements
sales
sale_items
purchases
purchase_items
customers
payments
invoices
audit_logs
```

Names should remain consistent with the actual Prisma schema.

---

# 27. IDs

Use one documented ID strategy consistently.

IDs should:

* Be unique.
* Be safe to expose where appropriate.
* Avoid unnecessary predictability for externally exposed resources.
* Remain consistent across APIs and database models.

Do not introduce multiple ID strategies without a clear reason.

---

# 28. Database Transactions

Use transactions when multiple changes must succeed or fail together.

For example, a completed sale may transactionally coordinate:

```text
Sale
+
Sale Items
+
Payment Allocations
+
Stock Movements
+
Stock Balance Updates
+
Invoice Record
+
Outbox Event
```

The exact transaction boundary must remain short and deterministic.

Do not perform external API calls inside the transaction.

---

# 29. Inventory Standards

Inventory is authoritative through the stock movement model.

Canonical movement types include:

```text
PURCHASE
SALE
RETURN
TRANSFER
DAMAGE
EXPIRY
ADJUSTMENT
```

Inventory changes should be traceable to a business event.

Do not silently change stock quantities.

A stock adjustment should record the reason and relevant actor/context.

---

# 30. Inventory Concurrency

Inventory operations must account for concurrent changes.

For example:

```text
Cashier A → attempts sale
Cashier B → attempts sale
        ↓
Same stock
        ↓
Database-controlled validation/update
```

Do not rely on cached stock values for final inventory authorization.

Redis may improve lookup performance, but PostgreSQL remains authoritative for critical stock validation and mutation.

---

# 31. Financial Standards

Financial calculations must be deterministic.

Avoid careless floating-point arithmetic for monetary values.

Define one application-wide monetary representation and use it consistently across:

* Products
* Sales
* Purchases
* Payments
* Discounts
* Taxes
* Invoices
* Reports

The chosen representation must align with PostgreSQL, Prisma, JavaScript, and provider integration behavior.

---

# 32. Payment Standards

Payment operations require additional safeguards.

* Never trust frontend payment success alone.
* Verify payment status server-side.
* Verify provider webhooks.
* Validate webhook signatures.
* Prevent duplicate processing.
* Use idempotency.
* Record payment state transitions.
* Never store raw card credentials.
* Keep external payment calls outside database transactions.

External gateway status should be reconciled using provider events/webhooks where applicable.

---

# 33. Idempotency Standard

Operations that may be retried must be designed to prevent duplicate business effects.

Important examples:

```text
Payments
Webhooks
Sale creation where retry is possible
Inventory operations
Queue jobs
Notifications
External API operations
```

Conceptually:

```text
Request
  ↓
Idempotency Key
  ↓
Check Existing Operation
  ↓
Process Once
  ↓
Retry
  ↓
Return Existing Result
```

Idempotency must be enforced at the appropriate application/database boundary, not merely stored in frontend state.

---

# 34. Redis Standards

Redis is supporting infrastructure.

It may be used for:

* Caching
* Temporary data
* Rate limiting
* Queue infrastructure
* Distributed coordination where genuinely required

Redis must not become the source of truth for:

* Inventory
* Sales
* Payments
* Invoices
* Financial records
* Tenant ownership

> **PostgreSQL owns business truth.**

---

# 35. Cache Standards

Use cache-aside where appropriate:

```text
Request
 ↓
Redis
 ↓
Hit → Return
 ↓
Miss
 ↓
PostgreSQL
 ↓
Cache Result
 ↓
Return
```

Cache keys must include appropriate scope.

Examples:

```text
tenant:{tenantId}:products:{productId}

tenant:{tenantId}:store:{storeId}:dashboard

tenant:{tenantId}:store:{storeId}:analytics
```

Caching must define:

* TTL
* Invalidation behavior
* Failure fallback
* Scope
* Staleness tolerance

Never use stale cache data as final authority for critical POS stock validation.

---

# 36. Cache Invalidation

Business data should be invalidated after successful database commit.

For important asynchronous propagation, use reliable event mechanisms such as a transactional outbox where required.

Do not assume:

```text
DB Commit
 ↓
Redis Invalidate
```

is automatically reliable.

If invalidation fails, TTL/rebuild mechanisms and retryable event processing should allow the system to recover.

---

# 37. BullMQ Standards

BullMQ is used for asynchronous work that should not block critical business requests.

Initial suitable workloads include:

* AI processing
* Notifications
* Email
* Reports
* Analytics processing
* PDF generation
* Scheduled maintenance
* Other non-critical post-commit work

Do not queue work simply because a queue exists.

---

# 38. Queue Reliability

Background jobs should support:

* Retry
* Backoff
* Idempotency
* Failure handling
* Monitoring
* Appropriate concurrency
* Controlled payload size

Workers must assume that jobs can be:

* Retried
* Duplicated
* Delayed
* Interrupted
* Processed after application restart

Exactly-once execution should not be assumed.

---

# 39. Async Event Standard

For important business events, the preferred architecture is:

```text
Business Transaction
       ↓
Database Transaction
       ↓
Outbox Event
       ↓
Commit
       ↓
Publisher
       ↓
BullMQ
       ↓
Worker
```

For lower-risk best-effort work, direct post-commit enqueueing may be acceptable.

The choice should be based on business reliability requirements.

---

# 40. Worker Context

Jobs should contain only the information required for processing.

Where relevant:

```js
{
  tenantId,
  storeId,
  resourceId,
  jobType,
  jobVersion,
  idempotencyKey
}
```

Avoid unnecessary:

* Secrets
* Large objects
* Sensitive personal data
* Full database records

Workers must perform tenant/store-scoped database access using trusted job context.

If a job originated from a user action, actor information may be retained for audit/correlation where appropriate.

---

# 41. AI Development Standards

AI must remain behind a controlled application boundary.

Conceptually:

```text
AI Business Module
       ↓
AI Service
       ↓
Provider Adapter
       ↓
External AI Provider
```

Provider-specific code should not spread throughout unrelated business modules.

The AI layer should support changing providers without rewriting core business logic where practical.

---

# 42. AI Authority Boundary

AI is an intelligence layer, not the authority for critical business state.

AI must not independently determine:

```text
Payment completion
Stock mutation
Invoice numbering
Tax calculation
Financial totals
Authorization
Tenant ownership
Critical database mutations
```

AI recommendations must pass through normal application/business rules before any permitted action occurs.

---

# 43. AI Data Access

AI must not receive unrestricted database access.

Preferred flow:

```text
AI
 ↓
Approved Application Tool / Service
 ↓
Authorization
 ↓
Tenant + Store Scope
 ↓
Approved Business Data
```

AI queries should retrieve only the information required for the specific task.

Cross-tenant AI context must never occur accidentally.

---

# 44. AI Output Validation

AI output must be treated as untrusted external input.

Where structured output is required:

* Validate schema.
* Validate allowed values.
* Apply business rules.
* Sanitize content.
* Handle malformed output.
* Handle provider failures.
* Apply timeouts and limits.

Successful AI generation does not mean the output is valid.

---

# 45. Frontend Architecture

The frontend uses Next.js App Router and React.

Components should have clear responsibilities.

Avoid components that simultaneously handle:

```text
UI
API communication
Complex business logic
Authorization rules
Global state
Database assumptions
```

Prefer:

```text
UI Component
    ↓
Hook / Client Logic
    ↓
API
    ↓
Backend Service
```

---

# 46. Server and Client Components

Use Next.js Server Components by default where appropriate.

Use Client Components when interactivity requires them.

Typical Client Component use cases include:

* POS interactions
* Complex forms
* Interactive tables
* Filters
* Browser APIs
* Zustand state
* Real-time interfaces

Do not make entire pages client-rendered simply because one section is interactive.

---

# 47. State Management

Use the simplest state mechanism that fits the requirement.

Prefer:

```text
Local React State
```

for local UI state.

Use:

```text
Zustand
```

when shared client-side state is genuinely required.

Server state should not be duplicated unnecessarily into Zustand.

Business truth remains on the server.

---

# 48. Server State vs Client State

The following must never become authoritative sources of business truth:

```text
React State
Zustand
localStorage
sessionStorage
Browser Cache
```

They must not determine:

* Tenant ownership
* Permissions
* Inventory truth
* Payment status
* Financial totals
* Final sale status

The server and database remain authoritative.

---

# 49. Data Fetching

Use the project's established data-fetching pattern consistently.

Where appropriate:

* Server-side fetching for server-rendered data.
* TanStack Query for client-side server state.
* Local state for temporary UI state.
* Zustand only for genuine shared client state.

Avoid creating multiple competing data-fetching patterns without justification.

---

# 50. UI Standards

Every major workflow should consider:

* Loading state
* Empty state
* Error state
* Success feedback
* Validation feedback
* Permission visibility
* Responsive behavior
* Accessibility

For example:

```text
Loading
   ↓
Data
   ↓
Empty
   ↓
Error
```

The UI must not assume that data always exists.

---

# 51. POS UI Standard

POS is a high-frequency workflow.

The interface should prioritize:

* Fast product search
* Barcode scanning
* Clear cart state
* Fast quantity changes
* Clear pricing
* Clear discounts
* Clear tax
* Payment state
* Sale completion feedback
* Error recovery

However, UI speed must not bypass backend validation.

The client cart is a convenience layer.

The server remains authoritative.

---

# 52. Forms

Forms should:

* Validate input.
* Provide useful validation messages.
* Prevent accidental duplicate submission.
* Show loading states.
* Handle server errors.
* Preserve input where appropriate.
* Provide confirmation for destructive/critical actions where necessary.

Critical operations should support safe retry behavior.

---

# 53. Naming Standards

Names should communicate intent.

Prefer:

```text
createSale()
calculateInvoiceTotal()
validateStock()
getTenantProducts()
generateDemandForecast()
resolveStoreContext()
```

Avoid vague names:

```text
doStuff()
handleData()
process()
run()
temp()
```

Names should reduce the need for explanatory comments.

---

# 54. File Naming

Maintain consistent project conventions.

Examples:

```text
ProductForm.jsx
InventoryTable.jsx
PosCart.jsx

formatCurrency.js
validatePagination.js

product.service.js
inventory.service.js
sale.service.js
```

Do not switch naming conventions randomly between modules.

---

# 55. Comments

Comments should explain **why**, not simply repeat **what** the code does.

Avoid:

```js
// Add product
addProduct(product);
```

Prefer:

```js
// Stock is revalidated during checkout because cached availability
// may have changed since the product was added to the cart.
```

Comments should be updated when the underlying behavior changes.

---

# 56. Code Reuse and Abstraction

Reuse stable concepts.

Do not create abstractions merely because two pieces of code currently look similar.

Good abstraction:

```text
Shared sale calculation
Shared authorization utility
Shared tenant context
Shared validation pattern
```

Potentially premature abstraction:

```text
GenericUniversalBusinessProcessor
```

created only for hypothetical future industries.

### Rule

> **Reuse stable concepts; do not abstract hypothetical requirements.**

---

# 57. Dependency Management

New dependencies require justification.

Before adding a package, ask:

* Do we already have this capability?
* Does the existing stack support it?
* Is the package maintained?
* Does it add meaningful value?
* Does it increase security risk?
* Does it increase bundle size?
* Does it introduce licensing concerns?
* Can the requirement reasonably be implemented internally?

Avoid dependency accumulation.

---

# 58. Environment Variables

Secrets and environment-specific configuration must never be hardcoded.

Examples:

```text
DATABASE_URL
REDIS_URL
AUTH_SECRET
AI_API_KEY
PAYMENT_SECRET
```

Maintain:

```text
.env.local
.env.example
```

`.env.example` must contain placeholders only.

Production secrets should be managed through appropriate deployment secret management.

---

# 59. Secret Management

Never commit:

* API keys
* Passwords
* Tokens
* Private keys
* Database credentials
* Payment secrets
* Production environment files

If a secret is accidentally committed:

1. Revoke/rotate it.
2. Remove it from active configuration.
3. Assess Git history exposure.
4. Update the affected systems.

Deleting the file alone does not make a leaked secret safe.

---

# 60. Git Standards

Git should provide a reliable development history.

Use meaningful commits:

```text
feat: add tenant onboarding
feat: implement stock movement service
fix: prevent cross-tenant product access
fix: handle duplicate payment webhook
refactor: simplify inventory repository
docs: update API standards
```

Avoid meaningless commits such as:

```text
update
changes
final
final2
test
asdf
```

---

# 61. Branching Standard

Keep branching simple.

Example:

```text
main
 ├── feature/*
 ├── fix/*
 └── refactor/*
```

The exact workflow may evolve as the team grows.

For a solo founder/team, avoid unnecessary Git process overhead.

---

# 62. Pull Request / Review Standard

Even when working alone, significant changes should be reviewable.

Before merging a meaningful change, review:

```text
What changed?
Why?
Which modules changed?
Does tenant isolation remain safe?
Does store isolation remain safe?
Are permissions correct?
Are transactions correct?
Are migrations required?
Are environment variables required?
Does documentation require updating?
```

AI-generated code receives the same review standard as manually written code.

---

# 63. AI-Assisted Development

AI tools are permitted and encouraged where they improve development speed and quality.

However:

> **Generated code is not automatically trusted code.**

Before accepting AI-generated code:

* Understand it.
* Verify architecture.
* Verify current framework APIs.
* Check tenant isolation.
* Check store isolation.
* Check authorization.
* Check database behavior.
* Check transactions.
* Check error handling.
* Check performance.
* Check dependencies.
* Check security.
* Remove unnecessary abstractions.

AI should implement within the established architecture rather than redesigning the architecture independently.

---

# 64. AI Coding Workflow

Preferred workflow:

```text
Requirement
    ↓
Architecture Check
    ↓
Implementation Plan
    ↓
AI-Assisted Implementation
    ↓
Human Review
    ↓
Lint / Build
    ↓
Functional Verification
    ↓
Security Review
    ↓
Observability Review
    ↓
Commit
```

AI accelerates implementation.

Engineering judgment remains human-controlled.

---

# 65. Documentation Standards

Important architectural decisions must be documented.

Examples:

```text
Database change
→ database-design.md

Security change
→ security.md

AI architecture change
→ ai-architecture.md

Caching/queue change
→ caching-and-queues.md

Observability change
→ observability.md

Deployment change
→ devops.md
```

Documentation should describe the actual architecture, not an idealized future system.

---

# 66. Documentation Synchronization

A significant architecture change should update:

```text
Implementation
+
Documentation
+
Architecture
```

Documentation drift is technical debt.

A feature should not be considered fully complete if the implementation significantly contradicts the documented architecture.

---

# 67. Feature Definition of Done

A feature is not complete merely because the UI works.

Applicable layers should be verified:

```text
Requirement
    ↓
Data Model
    ↓
API Contract
    ↓
Validation
    ↓
Business Logic
    ↓
Authorization
    ↓
Tenant Scope
    ↓
Store Scope
    ↓
Database Integrity
    ↓
Error Handling
    ↓
Observability
    ↓
Testing / Verification
    ↓
Documentation
```

Not every feature requires every layer, but each applicable concern must be considered.

---

# 68. Critical Feature Review

## Authentication

* [ ] Session/authentication security
* [ ] Authorization
* [ ] Tenant resolution
* [ ] Store context
* [ ] Error handling
* [ ] Rate limiting where required

## Inventory

* [ ] Tenant isolation
* [ ] Store isolation
* [ ] Stock movement
* [ ] Transaction integrity
* [ ] Concurrency handling
* [ ] Auditability

## POS

* [ ] Product validation
* [ ] Stock validation
* [ ] Price/tax/discount calculation
* [ ] Payment state
* [ ] Sale transaction
* [ ] Inventory update
* [ ] Invoice record
* [ ] Idempotency where required

## Payments

* [ ] Server-side verification
* [ ] Webhook verification
* [ ] Idempotency
* [ ] Duplicate protection
* [ ] Failure handling
* [ ] Reconciliation where applicable

## AI

* [ ] Tenant/store context
* [ ] Approved data access
* [ ] Output validation
* [ ] Provider failure handling
* [ ] Cost/usage visibility
* [ ] No unauthorized mutations

---

# 69. Performance Standards

Performance optimization should be evidence-driven.

Measure before optimizing.

Investigate:

* Slow database queries
* Excessive API calls
* Large payloads
* Unnecessary React renders
* Expensive calculations
* Cache opportunities
* Queue bottlenecks
* External provider latency

Do not introduce caching, queues, workers, or complex optimization solely because they are technically interesting.

---

# 70. Security-by-Default

Before implementing a feature, ask:

```text
Who can access it?

Which tenant owns the data?

Which store owns the data?

What can the user modify?

What validation is required?

Can the request be replayed?

Can another tenant access the resource?

What sensitive data is involved?

What must be audited?

What must be logged?
```

Security should be considered during design and implementation, not only before production.

---

# 71. Observability Standards

Important backend features should provide appropriate observability.

Consider:

* Structured logs
* Request ID
* Tenant ID where appropriate
* Store ID where appropriate
* Error codes
* Duration
* Business events
* Audit events
* Queue/job identifiers
* Metrics where useful

Never log sensitive information unnecessarily.

---

# 72. Database Migration Standards

All Prisma schema changes must be handled through version-controlled Prisma migrations.

Do not casually modify production schema manually.

Migrations must be:

* Reviewable
* Reproducible
* Tested
* Version controlled
* Compatible with deployment sequencing

Destructive migrations require additional planning.

For high-risk migrations, consider:

```text
Expand
 ↓
Migrate
 ↓
Verify
 ↓
Contract
```

rather than immediately removing data structures.

---

# 73. Backward Compatibility

When changing APIs, database structures, or job payloads, consider existing consumers.

Potential consumers include:

```text
Frontend
API Clients
Workers
Scheduled Jobs
Reports
Webhooks
External Integrations
```

Do not make breaking changes without intentionally coordinating the affected systems.

Job payloads should also support versioning when long-lived compatibility is required.

---

# 74. Refactoring Standards

Refactor when complexity creates a real problem.

Good reasons include:

* Duplicate business logic
* Poor module boundaries
* Difficult testing
* Security problems
* Performance problems
* Maintainability problems
* Repeated production defects

Do not refactor merely because another coding style looks more elegant.

---

# 75. Technical Debt

Technical debt should be recorded rather than forgotten.

For deferred work, document:

```text
Problem
Impact
Reason Deferred
Suggested Solution
Priority
```

Temporary shortcuts must not silently become permanent architecture.

---

# 76. Avoid Premature Complexity

Buzzsynx should not introduce architecture simply because it is technically interesting.

Do not prematurely introduce:

* Microservices
* Kubernetes
* Service mesh
* Multi-region infrastructure
* Multiple databases
* Distributed transactions
* Event-driven architecture everywhere
* Complex AI agents
* Custom ML infrastructure

unless real requirements justify them.

The modular monolith remains the default.

---

# 77. Testing Philosophy

Testing should prioritize business risk.

Critical areas include:

```text
Authentication
Tenant Isolation
Store Isolation
RBAC
Products
Inventory
Purchasing
POS
Payments
Returns
Critical Business Rules
AI Boundaries
Queue Processing
```

Important tests should cover both success and failure paths.

The detailed testing strategy remains defined in the separate testing document.

---

# 78. Testing Expectations for Critical Workflows

For critical workflows, verify:

```text
Correct Result
+
Authorization
+
Tenant Isolation
+
Store Isolation
+
Transaction Integrity
+
Idempotency
+
Failure Recovery
```

Examples include:

```text
Sale
Payment
Inventory Adjustment
Purchase Receiving
Return
Webhook Processing
Tenant Onboarding
```

---

# 79. Production Readiness

Before production release, consider:

* Security
* Tenant isolation
* Store isolation
* Data integrity
* Error handling
* Observability
* Performance
* Recovery
* Backup implications
* Migration impact
* Deployment impact
* External dependency failure
* Queue failure
* Redis failure

A feature that works locally is not automatically production-ready.

---

# 80. Local Development Standard

The local development environment should reproduce the important application dependencies.

Where applicable:

```text
Next.js
Express
PostgreSQL
Redis
BullMQ Worker
```

Docker Compose should provide a reproducible environment where practical.

The local environment should remain simple enough for fast development.

---

# 81. Environment Separation

Maintain clear separation between:

```text
Development
Staging
Production
```

Each environment must have its own configuration and appropriate secrets.

Never assume development configuration is safe for production.

---

# 82. Feature Flags and Configuration

Controlled configuration may be used for:

```text
Industry Capabilities
Feature Rollouts
AI Features
Subscription Entitlements
Operational Settings
Experimental Features
```

Do not scatter hardcoded conditions throughout the application.

Capability checks and authorization must still be enforced server-side.

---

# 83. File and Media Standards

Uploaded files are untrusted input.

Validate:

* File type
* File size
* File name
* File content where appropriate
* Storage location
* Access permissions

Private tenant files must not become publicly accessible merely because a URL exists.

Storage paths should preserve appropriate tenant/store boundaries.

---

# 84. External Service Standards

External services should be isolated behind provider/service boundaries where meaningful.

Examples:

```text
Payment Provider
AI Provider
Email Provider
Object Storage
```

Business modules should depend on application-level interfaces rather than spreading provider-specific logic throughout the codebase.

---

# 85. Retry Standards

Retries are appropriate for temporary failures such as:

* Network failures
* AI provider failures
* Email delivery
* Queue jobs
* External service failures

Do not blindly retry operations that may create duplicate financial or inventory effects.

Use idempotency and provider-specific semantics where required.

---

# 86. Graceful Failure

Optional services should fail without unnecessarily stopping core business operations.

Example:

```text
AI Provider Down
      ↓
AI Insight Unavailable
      ↓
POS Continues
```

Another example:

```text
Email Provider Down
      ↓
Notification Job Retries
      ↓
Business Transaction Remains Successful
```

The architecture should prefer graceful degradation.

---

# 87. Logging Standards

Important system boundaries should produce useful structured events.

Examples:

```text
auth.login.success
auth.login.failed

tenant.created
store.created

product.created
inventory.adjusted

sale.created
sale.failed

payment.failed
payment.webhook.processed

ai.request.failed

queue.job.completed
queue.job.failed
```

Event names should remain consistent.

Logging must not replace business audit records.

---

# 88. Audit Standards

Business-critical actions should produce audit events where required.

Examples:

```text
USER_ROLE_CHANGED
PRODUCT_UPDATED
INVENTORY_ADJUSTED
SALE_CREATED
PAYMENT_REFUNDED
TENANT_SETTINGS_CHANGED
STORE_CREATED
MEMBER_INVITED
TENANT_SUSPENDED
```

Audit records should capture appropriate:

```text
tenantId
storeId
userId
action
resource
resourceId
requestId
timestamp
reason / relevant metadata
```

Audit logs must be protected from unauthorized modification and access.

---

# 89. Audit vs Technical Logs

These are different systems.

### Technical Log

```text
inventory.service.updateStock.failed
```

Purpose:

> Help engineers diagnose system behavior.

### Audit Event

```text
INVENTORY_ADJUSTED
```

Purpose:

> Record an important business action.

Neither should be used as a substitute for the other.

---

# 90. Dependency Health

Critical infrastructure should have appropriate health monitoring.

Examples:

```text
PostgreSQL
Redis
BullMQ / Workers
AI Provider
Payment Provider
Email Provider
Object Storage
```

Dependencies should be classified as:

```text
Critical
Degraded
Optional
```

A failure in an optional service should not automatically make the entire application unavailable.

---

# 91. Business Workflow Integrity

The canonical supermarket workflow is:

```text
Tenant Onboarding
      ↓
Store
      ↓
Products
      ↓
Suppliers
      ↓
Purchase / Receive
      ↓
Inventory
      ↓
POS
      ↓
Sale / Payment
      ↓
Invoice
      ↓
Analytics
      ↓
AI Insights
```

The implementation should preserve the distinction between:

```text
Transactional Core
```

and:

```text
Derived / Asynchronous Intelligence
```

---

# 92. Core Transaction vs Async Work

Critical business operations should complete their authoritative database transaction before non-critical asynchronous processing.

Example:

```text
POS Sale
   ↓
DB Transaction
   ├── Sale
   ├── Payment Allocation
   ├── Stock Movement
   ├── Stock Balance
   └── Invoice Record
   ↓
Commit
   ↓
Async
   ├── Analytics
   ├── AI
   ├── Notifications
   ├── PDF
   └── Cache Invalidation
```

AI, email, PDF generation, and analytics processing must not unnecessarily block the core sale transaction.

---

# 93. Code Quality Checks

Before significant changes are committed, run applicable checks such as:

```text
Lint
Build
Unit Tests
Integration Tests
Migration Validation
API Verification
Security Checks
```

The exact checks depend on the affected module.

A feature should not be considered verified merely because the development server starts.

---

# 94. Engineering Decision Rule

When multiple solutions are possible, evaluate:

```text
Correctness
+
Security
+
Maintainability
+
Simplicity
+
Performance
+
Operational Reliability
```

Choose the simplest solution that satisfies the actual requirement.

Do not choose a more advanced architecture simply because it appears more scalable.

---

# 95. Architecture Change Rule

A developer or AI tool must not independently introduce a major architectural pattern without deliberate review.

Major changes include:

```text
New Database
New Service Boundary
New Queue Architecture
New Authentication Strategy
New Tenant Isolation Model
New AI Architecture
Microservice Extraction
Infrastructure Redesign
```

Such changes require explicit architectural evaluation.

---

# 96. Development Workflow

Preferred workflow:

```text
1. Understand Requirement
          ↓
2. Check Existing Architecture
          ↓
3. Identify Affected Module
          ↓
4. Define Data Changes
          ↓
5. Define API Contract
          ↓
6. Define Security / Scope
          ↓
7. Implement Backend
          ↓
8. Implement Frontend
          ↓
9. Add Validation
          ↓
10. Add Authorization
          ↓
11. Verify Tenant + Store Isolation
          ↓
12. Add Observability
          ↓
13. Test Critical Workflow
          ↓
14. Run Quality Checks
          ↓
15. Update Documentation
          ↓
16. Review
          ↓
17. Commit
```

---

# 97. Feature Review Checklist

Before considering a feature complete:

* [ ] Requirement understood.
* [ ] Correct module selected.
* [ ] Existing patterns reviewed.
* [ ] Data model reviewed.
* [ ] API contract defined.
* [ ] Input validation implemented.
* [ ] Authentication considered.
* [ ] Authorization implemented.
* [ ] Tenant isolation verified.
* [ ] Store isolation verified where applicable.
* [ ] Transaction boundary reviewed.
* [ ] Idempotency considered.
* [ ] Error handling implemented.
* [ ] Observability added.
* [ ] Sensitive data protected.
* [ ] Tests/verification completed.
* [ ] Documentation updated.
* [ ] No unnecessary dependency introduced.

---

# 98. AI Code Review Checklist

When AI significantly assists implementation:

* [ ] Code behavior is understood.
* [ ] Current framework APIs are verified.
* [ ] No hallucinated package/API is used.
* [ ] Existing project patterns are followed.
* [ ] Architecture is preserved.
* [ ] Tenant isolation is reviewed.
* [ ] Store isolation is reviewed.
* [ ] Authorization is reviewed.
* [ ] Database queries are reviewed.
* [ ] Transaction boundaries are reviewed.
* [ ] Error handling is reviewed.
* [ ] Security is reviewed.
* [ ] Performance is considered.
* [ ] Unnecessary abstractions are removed.
* [ ] Documentation is updated.

---

# 99. Development Standards Compliance

A module follows Buzzsynx engineering standards when applicable requirements are satisfied for:

```text
Architecture
Security
Tenant Isolation
Store Isolation
Authorization
Validation
Business Logic
Database Integrity
Transactions
Idempotency
Caching
Queues
AI Boundaries
Observability
Testing
Documentation
Operational Reliability
```

Compliance does not mean every module must implement every technology.

It means every relevant engineering concern has been deliberately addressed.

---

# 100. Final Engineering Rules

### Rule 1

> **Never trust the client with security decisions.**

### Rule 2

> **Never trust the client to determine tenant ownership.**

### Rule 3

> **Always validate store access against trusted membership context.**

### Rule 4

> **Never use AI as the authority for critical business calculations or state.**

### Rule 5

> **Never use Redis as the source of truth for business data.**

### Rule 6

> **Never assume background jobs execute exactly once.**

### Rule 7

> **Never commit secrets.**

### Rule 8

> **Never duplicate authoritative business rules unnecessarily.**

### Rule 9

> **Never allow frontend state to become business authority.**

### Rule 10

> **Never introduce architectural complexity without a real requirement.**

### Rule 11

> **Never treat AI-generated code as automatically correct.**

### Rule 12

> **Never call a feature complete until its applicable workflow has been verified.**

---

# 101. Final Development Principle

Buzzsynx should be developed with the mindset:

```text
Think Before Coding
        ↓
Understand the Architecture
        ↓
Design Before Implementing
        ↓
Define Security and Scope
        ↓
Build Deterministic Business Logic
        ↓
Use AI Intelligently
        ↓
Validate Before Trusting
        ↓
Observe Before Operating
        ↓
Test Before Declaring Complete
        ↓
Measure Before Optimizing
        ↓
Document Before Forgetting
```

The objective is not to write the maximum amount of code.

The objective is to build a system where every important piece of code has a clear reason to exist.

---

# 102. Final Statement

Buzzsynx development should remain disciplined as the product evolves.

The codebase should be:

* **Simple enough to understand**
* **Modular enough to evolve**
* **Secure enough to protect tenant businesses**
* **Reliable enough for critical workflows**
* **Observable enough to diagnose failures**
* **Deterministic enough for financial and inventory operations**
* **Flexible enough to support future industry capabilities**
* **Structured enough to support future scale**

The development philosophy is:

> **Build deliberately.**

> **Keep boundaries clear.**

> **Protect the data.**

> **Keep critical business rules deterministic.**

> **Use AI intelligently, not blindly.**

> **Prefer simple architecture until complexity is justified.**

> **Verify before declaring complete.**

> **Leave the codebase better than you found it.**

---

# Buzzsynx Engineering Motto

> **Understand → Design → Build → Verify → Harden → Deliver.**

> **Buzzsynx — First Brick, Not the Whole Building.**
