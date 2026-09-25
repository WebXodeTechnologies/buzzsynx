# Buzzsynx — Development Standards

**Document:** Development Standards & Engineering Guidelines
**Project:** Buzzsynx
**Version:** 1.0
**Status:** Development Standard
**Architecture:** Multi-Tenant Modular Monolith
**Primary Stack:** Next.js, React, Node.js, Express, PostgreSQL, Prisma, Redis, BullMQ
**Engineering Goal:** Maintainable, secure, scalable, observable, production-oriented software

---

# 1. Purpose

This document defines the engineering standards that must be followed while developing Buzzsynx.

The purpose is to ensure that:

* Code remains maintainable.
* Architecture remains consistent.
* Business logic remains predictable.
* Security is built into every feature.
* Multi-tenancy is enforced consistently.
* AI-assisted development does not introduce architectural shortcuts.
* New features follow established patterns.
* Technical debt is controlled.
* The codebase remains understandable as the project grows.

This document acts as the project's **engineering rulebook**.

---

# 2. Core Development Philosophy

Buzzsynx should be developed according to the following principles:

> **Simple before complex.**

> **Explicit before clever.**

> **Reusable before duplicated.**

> **Secure by default.**

> **Business logic before UI convenience.**

> **Database integrity before application assumptions.**

> **Observability from the beginning.**

> **AI assists development; engineering decisions remain deliberate.**

---

# 3. Architecture Standard

Buzzsynx uses a:

> **Multi-Tenant Modular Monolith**

The codebase should be organized around business domains.

Examples:

```text id="2m1h8v"
auth
tenants
users
roles
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

Each module should have clear responsibilities.

---

# 4. Module Boundary Rules

A module should own its business logic.

Example:

```text id="xw22o0"
Inventory Module
    ├── Controller
    ├── Service
    ├── Repository
    ├── Validation
    └── Domain Logic
```

The POS module may request inventory operations through an appropriate application/service boundary.

It should not directly manipulate inventory tables everywhere in the codebase.

### Rule

> **Modules communicate through defined application boundaries, not through arbitrary database manipulation.**

---

# 5. Separation of Responsibilities

The following separation should be maintained:

```text id="8at8jj"
Controller
   ↓
Service
   ↓
Repository
   ↓
Database
```

### Controller

Responsible for:

* Request handling
* Authentication context access
* Input parsing
* Calling services
* Returning responses

Controllers should remain thin.

### Service

Responsible for:

* Business rules
* Workflow orchestration
* Authorization checks where applicable
* Transaction coordination
* Calling repositories and domain services

### Repository

Responsible for:

* Database queries
* Persistence
* Data retrieval
* Tenant-scoped database operations

### Utility

Responsible for:

* Generic reusable functionality
* Formatting
* Small deterministic helpers

Utilities should not contain hidden business rules.

---

# 6. Business Logic Rules

Business logic must not be duplicated across:

* React components
* API controllers
* Random utility files
* Database queries
* Background workers

For example, sale total calculation should have one authoritative implementation.

Bad:

```text id="x3gb6c"
POS calculates total
Invoice calculates total differently
Report calculates total differently
```

Good:

```text id="kq8x3j"
Shared business calculation
       ↓
POS
Invoice
Reports
```

---

# 7. Deterministic Business Logic

Critical business operations must remain deterministic.

AI should not control:

* Payment totals
* Tax calculations
* Stock quantities
* Authorization
* Invoice numbering
* Financial calculations
* Transaction integrity

Example:

```text id="w1g3l0"
Product Price
+
Quantity
+
Discount
+
Tax
=
Deterministic Total
```

AI may analyze the resulting data but should not become the authority for the calculation.

---

# 8. Multi-Tenancy Standard

Every tenant-owned operation must be tenant-aware.

The backend must resolve tenant context from trusted authentication/membership information.

Never trust:

```text id="h3t0k2"
tenantId from request body
tenantId from query
tenantId from frontend state
tenantId from URL alone
```

as the source of authorization.

The expected context is conceptually:

```js
{
  userId,
  tenantId,
  role,
  permissions,
  capabilities
}
```

---

# 9. Tenant-Scoped Data Access

Tenant-owned queries must always include tenant scope.

Example:

```js
await prisma.product.findFirst({
  where: {
    id: productId,
    tenantId
  }
});
```

Avoid:

```js
await prisma.product.findUnique({
  where: {
    id: productId
  }
});
```

when the operation requires tenant isolation and the schema/query pattern does not otherwise guarantee it.

### Rule

> **Never rely on the frontend to provide tenant isolation.**

---

# 10. Cross-Tenant Security

Every feature must be evaluated for cross-tenant access.

Before considering a feature complete, verify:

```text id="o1u5ip"
Can Tenant A read Tenant B data?
Can Tenant A update Tenant B data?
Can Tenant A delete Tenant B data?
Can Tenant A access Tenant B files?
Can Tenant A access Tenant B analytics?
Can Tenant A trigger Tenant B operations?
Can Tenant A access Tenant B AI context?
```

The answer must be **no**, unless the operation is explicitly authorized as a platform-level operation.

---

# 11. RBAC Standard

Authorization must be enforced on the backend.

Frontend role checks are for user experience only.

Example:

```text id="8m5p0j"
Frontend
   ↓
Hide button

Backend
   ↓
Actually enforce permission
```

Never consider a feature secure merely because a button is hidden.

---

# 12. Capability Standard

Industry capabilities and permissions are separate concepts.

For example:

```text id="x4s6r3"
Tenant Industry
      ↓
PHARMACY

Capability
      ↓
EXPIRY_TRACKING

User Permission
      ↓
INVENTORY_VIEW
```

A user needs both the appropriate business capability and authorization to perform a restricted operation.

---

# 13. Validation Standard

All external input must be validated.

Sources include:

* Request body
* Query parameters
* Route parameters
* Headers where applicable
* Uploaded files
* Webhooks
* External API responses

Validation should occur before business logic executes.

---

# 14. Error Handling

Errors should be handled consistently.

Avoid returning raw internal errors to clients.

Bad:

```text id="v0cv4v"
PrismaClientKnownRequestError...
```

Prefer a controlled response such as:

```json id="2d50av"
{
  "success": false,
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found"
  }
}
```

Internal logs may contain additional technical information.

---

# 15. Error Categories

Use consistent error categories such as:

```text id="5zy3x6"
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
INTERNAL_ERROR
```

Error codes should be stable enough for frontend handling and operational monitoring.

---

# 16. API Standards

API design should remain consistent across modules.

Example:

```text id="0q5p4k"
GET    /api/products
GET    /api/products/:id
POST   /api/products
PATCH  /api/products/:id
DELETE /api/products/:id
```

Use HTTP methods according to their intended purpose.

Responses should follow a consistent structure.

Example:

```json id="o7m10c"
{
  "success": true,
  "data": {}
}
```

For errors:

```json id="5h0e4c"
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid product data"
  }
}
```

---

# 17. Database Standards

PostgreSQL is the authoritative business data store.

Database design should prioritize:

* Data integrity
* Referential integrity
* Appropriate constraints
* Appropriate indexes
* Transaction safety
* Tenant isolation

Do not use application code alone to enforce rules that should also be protected by database constraints where practical.

---

# 18. Database Naming

Use predictable naming conventions.

Examples:

```text id="z12w7o"
users
tenants
tenant_memberships
products
inventory
stock_movements
sales
sale_items
purchases
purchase_items
customers
payments
invoices
```

Names should be descriptive and consistent.

---

# 19. IDs

Use a consistent ID strategy across the application.

IDs should:

* Be unique.
* Be difficult to guess where appropriate.
* Remain consistent across APIs and database models.

Do not introduce multiple ID strategies without a documented reason.

---

# 20. Database Transactions

Transactions must be used for operations where multiple database changes must succeed or fail together.

Critical examples:

```text id="n5qzgj"
Sale
+
Sale Items
+
Payment
+
Stock Movement
+
Inventory Update
+
Invoice
```

These operations should be designed carefully so that partial success does not leave the system inconsistent.

---

# 21. Inventory Standards

Inventory must use the stock movement model.

Example:

```text id="n6t8cm"
PURCHASE
SALE
RETURN
TRANSFER
DAMAGE
EXPIRY
ADJUSTMENT
```

Inventory changes should be traceable.

Do not silently modify stock without an appropriate business event or adjustment record.

---

# 22. Financial Standards

Financial calculations must be deterministic.

Do not use floating-point arithmetic carelessly for monetary values.

The chosen representation must be consistent across:

* Products
* Sales
* Purchases
* Payments
* Taxes
* Discounts
* Invoices
* Reports

All monetary calculations should use one defined application-wide approach.

---

# 23. Payment Standards

Payment operations require additional safeguards.

* Verify payment status server-side.
* Verify provider webhooks.
* Prevent duplicate processing.
* Use idempotency where appropriate.
* Never trust client-side payment success alone.
* Never expose sensitive payment credentials.
* Record payment state transitions.

---

# 24. Idempotency

Operations that may be retried must be designed to avoid duplicate effects.

Important examples:

* Payments
* Webhooks
* Queue jobs
* Notifications
* Inventory operations
* External API requests

Example:

```text id="z4x9d7"
Payment Request
      ↓
Idempotency Key
      ↓
Process Once
      ↓
Retry
      ↓
Return Existing Result
```

---

# 25. Redis Standards

Redis is used for:

* Caching
* Temporary data
* Rate limiting
* Distributed coordination where required

Redis must not become the source of truth for:

* Inventory
* Payments
* Sales
* Invoices
* Financial records

PostgreSQL remains authoritative.

---

# 26. Cache Standards

Use cache-aside where appropriate:

```text id="3t1xwd"
Request
  ↓
Check Cache
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

Cache keys must be tenant-aware.

Example:

```text id="k17c3m"
tenant:{tenantId}:products:{productId}
```

Caching should have:

* Appropriate TTL
* Clear invalidation rules
* Failure fallback

---

# 27. BullMQ Standards

Background jobs should be used for operations that do not need to block the main request.

Examples:

* AI processing
* Emails
* Notifications
* Reports
* Analytics
* Scheduled processing

Jobs should include appropriate context:

```js
{
  tenantId,
  userId,
  jobType,
  resourceId
}
```

---

# 28. Queue Reliability

Jobs should support:

* Retry
* Backoff
* Idempotency
* Failure handling
* Monitoring
* Appropriate concurrency

Workers must never assume that a job will execute exactly once.

---

# 29. AI Development Standards

AI must remain behind a clear application boundary.

Recommended separation:

```text id="0n9lqe"
src/lib/ai
      ↓
AI Provider / Infrastructure

src/server/modules/ai
      ↓
AI Business Logic
```

AI provider-specific code should not spread throughout business modules.

---

# 30. AI Safety

AI must not have unrestricted access to:

* PostgreSQL
* Redis
* Files
* Internal APIs
* Administrative operations

AI tool calling must use explicitly approved application-level tools.

Example:

```text id="x6u7pr"
AI
 ↓
Approved Tool
 ↓
Application Service
 ↓
Authorization
 ↓
Tenant-Scoped Data
```

---

# 31. AI Output Validation

Never assume AI output is valid simply because it is returned successfully.

AI outputs should be:

* Structured where possible.
* Schema validated.
* Sanitized.
* Interpreted by application logic.
* Logged appropriately.

AI-generated recommendations must not automatically become critical business mutations.

---

# 32. Frontend Standards

React components should have clear responsibilities.

Avoid creating components that simultaneously handle:

* UI
* API calls
* Business rules
* Database assumptions
* Complex state management
* Authorization logic

Prefer:

```text id="f3yy8w"
UI Component
     ↓
Hook / Client Logic
     ↓
API
     ↓
Backend Service
```

---

# 33. Server vs Client Components

Next.js App Router should use Server Components by default where appropriate.

Use Client Components when interactive behavior requires them.

Client Components should not be introduced unnecessarily.

Examples that typically require client behavior:

* POS interaction
* Complex forms
* Interactive filters
* Real-time UI
* Zustand state
* Browser APIs

---

# 34. State Management

Use local React state when state is local.

Use Zustand only when state genuinely needs broader client-side sharing.

Do not place all application data into a global store.

Avoid creating a global state architecture that duplicates server state unnecessarily.

---

# 35. Data Fetching

Server-owned business data should remain server-authoritative.

Avoid treating:

```text id="p7m7fc"
Zustand
localStorage
React state
```

as sources of truth for:

* Permissions
* Inventory
* Payment status
* Tenant identity
* Financial totals

---

# 36. UI Standards

Every major UI flow should consider:

* Loading state
* Empty state
* Error state
* Success feedback
* Validation feedback
* Responsive behavior
* Accessibility
* Permission visibility

Example:

```text id="9fj3jq"
Loading
   ↓
Data
   ↓
Empty
   ↓
Error
```

A page should not assume that data always exists.

---

# 37. Forms

Forms should:

* Validate input.
* Display useful errors.
* Prevent accidental duplicate submission.
* Provide loading states.
* Handle server errors.
* Preserve user input where appropriate.

Critical actions should require appropriate confirmation when necessary.

---

# 38. Naming Standards

Names should communicate intent.

Prefer:

```text id="9byu6q"
createSale()
calculateInvoiceTotal()
validateStock()
getTenantProducts()
generateDemandForecast()
```

Avoid vague names:

```text id="1v6ypb"
doStuff()
handleData()
process()
run()
temp()
```

Names should make the code understandable without requiring excessive comments.

---

# 39. File Naming

Use consistent naming conventions across the project.

React components:

```text id="8jpxl6"
ProductForm.jsx
InventoryTable.jsx
PosCart.jsx
```

Utilities:

```text id="k0e1ks"
formatCurrency.js
validatePagination.js
```

Services:

```text id="m7xvdr"
product.service.js
inventory.service.js
sale.service.js
```

Maintain the chosen project convention consistently.

---

# 40. Comments

Comments should explain **why**, not simply repeat **what** the code does.

Bad:

```js
// Add product to cart
addProduct(product);
```

Useful:

```js
// Stock is revalidated during checkout because cached availability
// may have changed since the product was added to the cart.
```

Avoid excessive comments that make the code harder to read.

---

# 41. Code Duplication

Before creating new logic, check whether an existing implementation can be reused.

Avoid:

```text id="5z9w6m"
calculateTotalA()
calculateTotalB()
calculateTotalC()
```

when the underlying business rule is the same.

However, do not create premature abstractions merely to eliminate tiny similarities.

### Rule

> **Reuse stable concepts; do not abstract hypothetical future requirements.**

---

# 42. Dependency Management

New dependencies should be introduced only when they provide meaningful value.

Before adding a package, consider:

* Do we already have this capability?
* Is the package actively maintained?
* Is it compatible with the current stack?
* Does it introduce unnecessary complexity?
* Does it create security or licensing concerns?
* Can the functionality reasonably be implemented internally?

Avoid dependency accumulation.

---

# 43. Environment Variables

Secrets and environment-specific configuration must never be hardcoded.

Examples:

```text id="0aw5ah"
DATABASE_URL
REDIS_URL
AUTH_SECRET
AI_API_KEY
PAYMENT_SECRET
```

Maintain:

```text id="2gkr0q"
.env.local
.env.example
```

`.env.example` should document required variables without exposing real secrets.

---

# 44. Security Secrets

Never commit:

* API keys
* Passwords
* Tokens
* Private keys
* Database credentials
* Payment secrets
* Production environment files

Git history should also be considered when a secret is accidentally committed.

---

# 45. Git Standards

Git should provide a reliable history of development.

Use meaningful commits.

Prefer:

```text id="7h5n0c"
feat: add tenant onboarding
feat: implement stock movement service
fix: prevent cross-tenant product access
refactor: simplify inventory repository
docs: update API standards
```

Avoid:

```text id="q7y2h3"
update
changes
final
final2
test
asdf
```

---

# 46. Branching

A simple branching model is preferred.

Example:

```text id="2b5s6h"
main
  │
  ├── feature/*
  ├── fix/*
  └── refactor/*
```

The exact workflow can evolve as the team grows.

Do not create unnecessary branching complexity for a small team.

---

# 47. Pull Requests

Even when working alone, significant changes should be reviewable.

A pull request or equivalent review should consider:

* What changed?
* Why?
* Which modules are affected?
* Are tenant boundaries preserved?
* Are security implications understood?
* Are migrations required?
* Are environment variables required?
* Does documentation need updating?

AI-generated code should receive the same review as manually written code.

---

# 48. AI-Assisted Development Standards

AI tools may be used extensively during development.

However:

> **Generated code is not automatically trusted code.**

Before accepting AI-generated code:

* Understand what it does.
* Verify architecture.
* Check tenant isolation.
* Check authorization.
* Check error handling.
* Check database behavior.
* Check performance.
* Check security.
* Check dependencies.
* Check whether it duplicates existing logic.

Do not blindly paste generated code into production paths.

---

# 49. AI Coding Workflow

Recommended workflow:

```text id="o0jbrv"
Requirement
    ↓
Architecture Check
    ↓
Implementation Plan
    ↓
AI-Assisted Code
    ↓
Human Review
    ↓
Lint / Build
    ↓
Functional Verification
    ↓
Security Review
    ↓
Commit
```

AI should accelerate implementation, not replace engineering judgment.

---

# 50. Documentation Standards

Architectural decisions should be documented.

When implementation changes an important architectural decision, update the relevant documentation.

Examples:

```text id="9grm44"
Database change
→ database-design.md

Security change
→ security.md

AI architecture change
→ ai-architecture.md

Deployment change
→ devops.md

Observability change
→ observability.md
```

Avoid allowing documentation and implementation to drift significantly.

---

# 51. Definition of Done for a Feature

A feature is not complete merely because the UI exists.

A feature should generally satisfy:

```text id="jvpsx9"
Requirement
   ↓
UI
   ↓
API
   ↓
Validation
   ↓
Business Logic
   ↓
Database
   ↓
Authorization
   ↓
Tenant Isolation
   ↓
Error Handling
   ↓
Logging
   ↓
Documentation
   ↓
Verification
```

Not every feature requires every layer, but applicable layers must be considered.

---

# 52. Critical Feature Review

For business-critical features, explicitly review:

### Authentication

* [ ] Session security
* [ ] Authorization
* [ ] Error handling

### Inventory

* [ ] Transaction integrity
* [ ] Tenant isolation
* [ ] Stock movement
* [ ] Concurrency

### POS

* [ ] Stock validation
* [ ] Price calculation
* [ ] Payment
* [ ] Transaction atomicity

### Payments

* [ ] Server-side verification
* [ ] Webhooks
* [ ] Idempotency
* [ ] Failure handling

### AI

* [ ] Tenant context
* [ ] Output validation
* [ ] Cost control
* [ ] Provider failure handling

---

# 53. Performance Standards

Performance optimization should be evidence-driven.

Do not optimize based only on assumptions.

Look for:

* Slow queries
* Excessive API requests
* Large payloads
* Unnecessary renders
* Expensive calculations
* Cache opportunities
* Queue bottlenecks

Prefer measuring before optimizing.

---

# 54. Security-by-Default

New features should begin secure rather than becoming secure later.

Before implementing a feature, ask:

```text id="yq5g47"
Who can access it?
Which tenant owns the data?
What can the user modify?
What validation is required?
What happens if the request is replayed?
What sensitive data is involved?
What should be logged?
```

Security review should happen during implementation, not only before deployment.

---

# 55. Observability Standards

New backend features should provide appropriate observability.

Consider:

* Structured logs
* Request ID
* Tenant context
* Error codes
* Duration
* Business events
* Audit events
* Metrics where useful

Do not log sensitive information unnecessarily.

---

# 56. Database Migration Standards

Database schema changes must be handled through Prisma migrations.

Never casually modify production schema manually.

Migration changes should be:

* Version controlled.
* Reviewable.
* Reproducible.
* Tested against expected environments.
* Compatible with deployment sequencing.

Destructive migrations require additional care.

---

# 57. Backward Compatibility

When changing APIs or database structures, consider existing consumers.

Avoid breaking:

```text id="r3s5vc"
Frontend
API
Workers
Scheduled Jobs
Reports
External Integrations
```

without intentionally coordinating the change.

---

# 58. Refactoring Standards

Refactor when complexity is creating real problems.

Good reasons:

* Duplicate business logic
* Difficult testing
* Poor module boundaries
* Performance issues
* Security concerns
* Maintainability problems

Avoid refactoring simply because another coding style looks more elegant.

---

# 59. Technical Debt

Technical debt should be recorded rather than forgotten.

For deferred work, document:

```text id="d8x7xq"
Problem
Impact
Reason Deferred
Suggested Solution
Priority
```

This prevents temporary shortcuts from silently becoming permanent architecture.

---

# 60. Avoid Premature Complexity

Buzzsynx should not introduce architecture simply because it is technically interesting.

Avoid prematurely adding:

* Microservices
* Kubernetes
* Event-driven everything
* Multiple databases
* Multi-region architecture
* Complex AI agents
* Service meshes
* Distributed transactions

The modular monolith should remain the default until real requirements justify additional complexity.

---

# 61. Testing Philosophy

Testing is an important part of the eventual development process.

At minimum, critical business logic should eventually have appropriate automated coverage.

Priority areas include:

```text id="wq0wz3"
Authentication
Multi-Tenancy
RBAC
Inventory
POS
Payments
Critical Business Rules
AI Boundaries
```

The detailed testing strategy remains a separate document and can be completed when the implementation phase reaches that requirement.

---

# 62. Production Readiness

Before a feature reaches production, consider:

* Security
* Tenant isolation
* Data integrity
* Error handling
* Observability
* Performance
* Recovery
* Deployment impact
* Migration impact

A feature that works locally is not automatically production-ready.

---

# 63. Code Review Checklist

Before merging significant code:

* [ ] Requirement understood.
* [ ] Correct module used.
* [ ] No unnecessary duplication.
* [ ] Business logic is in the appropriate layer.
* [ ] Validation exists.
* [ ] Authorization exists.
* [ ] Tenant isolation exists.
* [ ] Errors are handled.
* [ ] Sensitive data is protected.
* [ ] Database operations are safe.
* [ ] Transactions are used where required.
* [ ] Logging is appropriate.
* [ ] Documentation is updated.
* [ ] No unnecessary dependency added.
* [ ] No secrets committed.

---

# 64. AI Code Review Checklist

When AI generated or significantly assisted the implementation:

* [ ] Code behavior understood.
* [ ] No hallucinated APIs or packages.
* [ ] Current framework conventions verified.
* [ ] Existing project patterns followed.
* [ ] Security reviewed.
* [ ] Tenant isolation reviewed.
* [ ] Database queries reviewed.
* [ ] Error handling reviewed.
* [ ] Performance considered.
* [ ] Unnecessary abstraction removed.
* [ ] Documentation updated.

---

# 65. Development Workflow

The preferred workflow for Buzzsynx is:

```text id="3w1j76"
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
6. Implement Backend
        ↓
7. Implement Frontend
        ↓
8. Add Validation
        ↓
9. Add Authorization
        ↓
10. Verify Tenant Isolation
        ↓
11. Add Observability
        ↓
12. Verify Business Workflow
        ↓
13. Run Quality Checks
        ↓
14. Update Documentation
        ↓
15. Commit
```

---

# 66. Local Development Standard

Development should be reproducible.

Where applicable, the local environment should use:

```text id="17p0hv"
Next.js
Express
PostgreSQL
Redis
BullMQ Worker
```

Docker Compose should eventually provide a consistent development environment.

---

# 67. Environment Separation

Maintain clear separation between:

```text id="6yyvvr"
Development
Staging
Production
```

Never assume that configuration safe for development is automatically safe for production.

---

# 68. Feature Flags & Configuration

Features that vary by tenant or environment should use controlled configuration.

Examples:

```text id="f8m5f1"
Industry Capability
Feature Flag
AI Feature
Subscription Feature
Operational Setting
```

Avoid scattering hardcoded feature conditions throughout the application.

---

# 69. File & Media Standards

Uploaded files should be treated as untrusted input.

Validate:

* File type
* File size
* File name
* Storage path
* Access permissions

Do not expose private tenant files publicly without appropriate authorization.

---

# 70. External Service Standards

External services should be isolated behind service/provider abstractions where appropriate.

Examples:

```text id="4q7m32"
Payment Provider
AI Provider
Email Provider
File Storage
```

Business modules should not become tightly coupled to a specific provider when abstraction provides meaningful value.

---

# 71. Retry Standards

Retries should be used carefully.

Good candidates:

* Temporary network failures
* AI provider failures
* Email delivery
* Background jobs
* External APIs

Do not blindly retry operations that may create duplicate financial or inventory effects.

Use idempotency where required.

---

# 72. Graceful Failure

Optional services should fail gracefully.

Example:

```text id="f4d2gz"
AI Provider Down
      ↓
AI Feature Unavailable
      ↓
Core Business Operations Continue
```

Core business functionality should not unnecessarily depend on optional intelligence services.

---

# 73. Logging Standards

Every significant system boundary should provide useful logs.

Examples:

```text id="4p8y1f"
auth.login.success
auth.login.failed
tenant.created
product.created
inventory.adjusted
sale.created
payment.failed
ai.request.failed
queue.job.completed
```

Event names should remain consistent.

---

# 74. Audit Standards

Business-critical actions should produce audit events.

Examples:

```text id="g8l8wt"
USER_ROLE_CHANGED
PRODUCT_UPDATED
INVENTORY_ADJUSTED
SALE_CREATED
PAYMENT_REFUNDED
TENANT_SETTINGS_CHANGED
```

Audit records must be protected from unauthorized modification or access.

---

# 75. Documentation Synchronization

When a significant implementation decision changes:

```text id="z8i6xq"
Code
  +
Documentation
  +
Architecture
```

must remain aligned.

A technically correct implementation with outdated documentation is considered incomplete.

---

# 76. Project Consistency Rule

Before introducing a new pattern, inspect the existing codebase.

Ask:

> **Do we already have a pattern for this?**

If yes, follow the existing pattern unless there is a documented reason to change it.

Consistency is more valuable than constantly switching between approaches.

---

# 77. Engineering Decision Rule

When multiple solutions are possible, prefer the solution that provides the best balance of:

```text id="3r5p9y"
Correctness
+
Security
+
Maintainability
+
Simplicity
+
Performance
```

Do not choose a solution merely because it is more advanced.

---

# 78. Definition of Development Standards Compliance

A feature or module follows Buzzsynx development standards when:

* [ ] It follows the modular architecture.
* [ ] Responsibilities are clearly separated.
* [ ] Business logic is deterministic where required.
* [ ] Tenant isolation is enforced.
* [ ] Authorization is enforced server-side.
* [ ] Input is validated.
* [ ] Errors are handled consistently.
* [ ] Database access is safe.
* [ ] Critical operations use transactions where necessary.
* [ ] Redis is used appropriately.
* [ ] Background jobs are reliable.
* [ ] AI is safely isolated.
* [ ] Observability is provided.
* [ ] Secrets are protected.
* [ ] Documentation is maintained.
* [ ] The implementation avoids unnecessary complexity.

---

# 79. Final Engineering Rules

The following rules should remain visible throughout development.

### Rule 1

> **Never trust the client with security decisions.**

### Rule 2

> **Never trust the client to determine tenant ownership.**

### Rule 3

> **Never use AI as the authority for critical business calculations.**

### Rule 4

> **Never use Redis as the source of truth for business data.**

### Rule 5

> **Never allow background jobs to silently fail.**

### Rule 6

> **Never commit secrets.**

### Rule 7

> **Never duplicate business rules unnecessarily.**

### Rule 8

> **Never introduce architecture complexity without a real requirement.**

### Rule 9

> **Never treat AI-generated code as automatically correct.**

### Rule 10

> **Never call a feature complete until its complete workflow is verified.**

---

# 80. Final Development Principle

Buzzsynx should be built with the mindset:

```text id="t0y2mz"
Think Before Coding
        ↓
Design Before Implementing
        ↓
Understand Before Using AI
        ↓
Secure Before Exposing
        ↓
Validate Before Trusting
        ↓
Observe Before Operating
        ↓
Measure Before Optimizing
        ↓
Document Before Forgetting
```

The objective is not to write the maximum amount of code.

The objective is to build a system where every important piece of code has a clear reason to exist.

---

# 81. Final Statement

Buzzsynx development should remain disciplined as the project grows.

The codebase should be:

* **Simple enough to understand**
* **Modular enough to evolve**
* **Secure enough to protect tenant businesses**
* **Reliable enough for critical workflows**
* **Observable enough to diagnose failures**
* **Flexible enough to support multiple industries**
* **Structured enough to support future scale**

The development standard can be summarized as:

> **Build deliberately.**
>
> **Keep boundaries clear.**
>
> **Protect the data.**
>
> **Keep business rules deterministic.**
>
> **Use AI intelligently, not blindly.**
>
> **Prefer simple architecture until complexity is justified.**
>
> **Leave the codebase better than you found it.**

### Buzzsynx Engineering Motto

> **Understand → Design → Build → Verify → Harden → Deliver.**

**Buzzsynx — First Brick, Not the Whole Building.**
