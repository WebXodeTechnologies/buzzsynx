# Buzzsynx — API Design

**Version:** v0.2
**Status:** Architecture-Aligned API Baseline
**Product:** Buzzsynx
**Architecture:** Multi-Tenant Modular Monolith
**API Style:** REST
**Backend:** Node.js + Express
**Database:** PostgreSQL + Prisma
**Cache / Queue:** Redis + BullMQ

---

# 1. Purpose

This document defines the API architecture, conventions, security boundaries, request lifecycle, business-operation patterns, and development standards for Buzzsynx.

The API is the application boundary between the Buzzsynx frontend and the business domain.

The API must support:

* Multi-tenant SaaS operations
* Authentication
* Authorization and RBAC
* Tenant and store/branch isolation
* Product management
* Inventory management
* Purchasing
* POS
* Sales
* Payments
* Customers
* Suppliers
* Invoices
* Analytics
* Reports
* AI capabilities
* Notifications
* Industry capabilities
* Audit logging

The API must prioritize:

* Security
* Tenant isolation
* Business correctness
* Transaction integrity
* Maintainability
* Consistent contracts
* Observability
* Future extensibility

The API is part of the **Buzzsynx modular monolith**. It should not introduce microservice complexity unless future requirements justify it.

---

# 2. API Architecture

Buzzsynx uses a REST API inside the modular monolith.

```text
Next.js Frontend
       ↓
HTTP / REST API
       ↓
Express
       ↓
Middleware
       ↓
Controllers
       ↓
Application Services
       ↓
Repositories / Prisma
       ↓
PostgreSQL
```

Supporting infrastructure:

```text
Express Application
       │
       ├── Redis
       ├── BullMQ
       ├── AI Providers
       ├── Storage
       ├── Payment Providers
       └── External Services
```

The API does not communicate directly with PostgreSQL from controllers.

Business operations must pass through the appropriate application/service layer.

---

# 3. API Style

Buzzsynx initially uses **RESTful APIs**.

Examples:

```text
GET    /api/v1/products
GET    /api/v1/products/:id
POST   /api/v1/products
PATCH  /api/v1/products/:id
DELETE /api/v1/products/:id
```

REST is preferred initially because it provides:

* Clear HTTP semantics
* Simple client integration
* Easy debugging
* Straightforward authorization
* Good testing support
* Mobile compatibility
* Easy third-party integration

GraphQL is not required for the initial architecture.

It may be evaluated later only if the product develops a genuine need for highly flexible client-driven data fetching.

---

# 4. API Versioning

Application APIs use a version prefix.

```text
/api/v1
```

Examples:

```text
/api/v1/products
/api/v1/inventory
/api/v1/sales
/api/v1/customers
```

A future breaking API contract may use:

```text
/api/v2
```

Versioning must be introduced deliberately.

Minor backward-compatible changes should not require a new major API version.

---

# 5. Base URL

Development:

```text
http://localhost:4000/api/v1
```

Staging:

```text
https://api-staging.buzzsynx.com/api/v1
```

Production:

```text
https://api.buzzsynx.com/api/v1
```

The exact production domains may change during deployment.

The frontend must obtain environment-specific API URLs through configuration and must not hard-code production URLs.

Example:

```text
NEXT_PUBLIC_API_URL
```

---

# 6. API Module Structure

The Express backend is organized by business module.

```text
src/server/modules/

├── auth/
├── tenants/
├── users/
├── roles/
├── products/
├── inventory/
├── pos/
├── sales/
├── purchases/
├── suppliers/
├── customers/
├── payments/
├── analytics/
├── reports/
├── ai/
└── notifications/
```

A typical module may contain:

```text
products/

├── product.routes.js
├── product.controller.js
├── product.service.js
├── product.repository.js
├── product.validation.js
└── product.constants.js
```

The structure may evolve as implementation grows.

The architecture should preserve domain boundaries even when the codebase remains a single deployable application.

---

# 7. Request Lifecycle

Every authenticated tenant request should conceptually follow:

```text
HTTP Request
     ↓
Request ID
     ↓
Security Middleware
     ↓
Authentication
     ↓
Tenant Resolution
     ↓
Membership Resolution
     ↓
Store / Branch Scope
     ↓
RBAC / Permission Check
     ↓
Capability Check
     ↓
Input Validation
     ↓
Controller
     ↓
Service
     ↓
Repository / Prisma
     ↓
PostgreSQL
     ↓
Response
```

For failures:

```text
Error
 ↓
Error Middleware
 ↓
Structured Error
 ↓
HTTP Response
```

Not every endpoint requires every step.

For example, public authentication endpoints do not require an existing authenticated tenant context.

---

# 8. Request Context

The API should construct a trusted request context after authentication and authorization middleware.

Conceptually:

```text
Request Context

{
  requestId,
  userId,
  tenantId,
  membershipId,
  roleIds,
  permissionSet,
  activeStoreId,
  capabilities
}
```

This context is derived from trusted server-side data.

The application must not blindly accept these values from the request body.

---

# 9. Request ID

Every API request should have a unique request identifier.

Example:

```text
X-Request-ID: req_01JXYZ...
```

If a client provides a request ID, the server should validate it according to the application's security and tracing rules.

Otherwise, the server generates one.

Request IDs should be available in:

* Application logs
* Error responses where appropriate
* Error monitoring
* Background job metadata where useful
* Distributed tracing where implemented

Request IDs must not contain sensitive information.

---

# 10. Authentication

Authentication determines:

> **Who is the user?**

Authentication is separate from authorization.

```text
Authentication
      ↓
Who are you?

Authorization
      ↓
What are you allowed to do?
```

Initial authentication may support:

```text
Email + Password
Google OAuth
```

Future options may include:

```text
Passkeys
Enterprise SSO
```

Authentication implementation must use secure session/token practices appropriate to the chosen architecture.

---

# 11. Authentication Endpoints

Initial endpoints:

```text
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/logout
POST /api/v1/auth/refresh
GET  /api/v1/auth/me
```

Additional endpoints may include:

```text
POST /api/v1/auth/forgot-password
POST /api/v1/auth/reset-password
POST /api/v1/auth/verify-email
GET  /api/v1/auth/google
GET  /api/v1/auth/google/callback
```

Only authentication mechanisms actually implemented should be exposed.

---

# 12. Registration Workflow

Buzzsynx onboarding creates the initial business context.

Conceptually:

```text
Registration Request
        ↓
Validate Input
        ↓
Check User
        ↓
Create User
        ↓
Create Tenant / Business
        ↓
Create Initial Store / Branch
        ↓
Select Industry / Capability Configuration
        ↓
Create Business Settings
        ↓
Create Initial Membership
        ↓
Assign Owner Role
        ↓
Create Default Configuration
        ↓
Create Authenticated Session
        ↓
Return Response
```

The core onboarding operation should use an appropriate database transaction.

The person creating the business receives the **Owner** membership.

Platform review or administrative approval, where required, should not unnecessarily block normal onboarding.

Tenant lifecycle may be:

```text
PENDING
   ↓
ACTIVE
   ↓
SUSPENDED
   ↓
ARCHIVED
```

The exact lifecycle rules belong to the tenant/domain layer.

---

# 13. Login Workflow

```text
POST /api/v1/auth/login
```

Conceptual flow:

```text
Credentials
    ↓
Validate Input
    ↓
Find User
    ↓
Verify Credentials
    ↓
Resolve Membership
    ↓
Resolve Active Tenant
    ↓
Resolve Active Store Scope
    ↓
Create Session / Token
    ↓
Return Authenticated Response
```

Failed authentication should use appropriate generic errors and should not expose sensitive account information.

---

# 14. Session Strategy

The exact implementation is finalized during authentication development.

The architecture must support:

* Secure session handling
* Expiration
* Refresh or re-authentication
* Logout/revocation where applicable
* Secure cookies or equivalent secure token handling
* CSRF protection where cookie-based authentication requires it
* Session invalidation when necessary

Sensitive authentication material must never be logged.

---

# 15. Tenant and Store Resolution

Tenant isolation is a critical API security boundary.

Buzzsynx hierarchy:

```text
Super Admin
     ↓
Tenant / Business
     ↓
Store / Branch
     ↓
Membership / User
```

A request must establish:

```text
currentUser
currentTenant
currentMembership
currentStore
permissions
capabilities
```

where applicable.

The backend must never blindly trust:

```text
tenantId
storeId
```

provided by the browser.

Client-provided IDs may be used as operation inputs only after the server verifies that they belong to the authenticated tenant and that the user is authorized to access them.

---

# 16. Tenant-Scoped APIs

Most business endpoints operate within the authenticated tenant context.

Example:

```text
GET /api/v1/products
```

The server derives:

```text
currentTenantId
```

from trusted authentication and membership context.

Queries must enforce tenant scope:

```javascript
where: {
  tenantId: currentTenantId
}
```

For store-scoped resources, the query must additionally enforce the appropriate store scope.

Example:

```javascript
where: {
  tenantId: currentTenantId,
  storeId: currentStoreId
}
```

Tenant isolation must exist at the application/service/data-access boundary, not merely in the frontend.

---

# 17. System-Level APIs

Some endpoints are platform-level.

Examples:

```text
GET /api/v1/system/health
GET /api/v1/system/version
```

These do not use ordinary tenant authorization.

Platform endpoints must have their own security rules.

For example:

```text
Super Admin
```

may access administrative platform APIs that ordinary tenant users cannot.

Public health checks should expose only the minimum information necessary.

---

# 18. Authorization

Authorization combines:

```text
Authentication
+
Tenant Membership
+
Role / Permissions
+
Store Scope
+
Capability
```

Conceptually:

```text
User
 ↓
Membership
 ↓
Role
 ↓
Permissions
 ↓
Tenant Scope
 ↓
Store Scope
 ↓
Capability
 ↓
Endpoint / Operation
```

RBAC determines what the user is allowed to perform.

Capability configuration determines whether the tenant has the relevant business functionality enabled.

---

# 19. Current Buzzsynx Roles

The initial role model is:

```text
Super Admin
Owner
Admin / Manager
Cashier
Accountant
Store Staff
```

### Super Admin

Platform-level management.

### Owner

Business owner and highest tenant-level operational role.

### Admin / Manager

Operational management according to assigned permissions.

### Cashier

POS and payment-related operations according to assigned permissions.

### Accountant

Financial, payment, reporting, and accounting-related operations according to assigned permissions.

### Store Staff

Operational store functions according to assigned permissions.

Permissions, rather than role names alone, should determine actual access.

---

# 20. Permission Naming

Permissions should use a predictable convention.

Recommended:

```text
resource.action
```

Examples:

```text
product.view
product.create
product.update
product.delete

inventory.view
inventory.adjust
inventory.transfer

sale.view
sale.create
sale.cancel
sale.refund

purchase.view
purchase.create
purchase.receive

customer.view
customer.create
customer.update

payment.view
payment.create
payment.refund

report.view
settings.view
settings.update
```

Permission names should remain stable once exposed to application logic.

---

# 21. Capability Authorization

RBAC and industry capability are separate concepts.

```text
User Permission
      +
Tenant Capability
      +
Store Scope
      ↓
Allowed Operation
```

For example:

```text
medicine.batch.manage
```

should only be available when the appropriate pharmacy capability is enabled.

Similarly:

```text
restaurant.recipe.manage
```

belongs to the restaurant capability.

Capability checks must be enforced by the backend.

They must not exist only as frontend feature flags.

---

# 22. Product APIs

Core endpoints:

```text
GET    /api/v1/products
POST   /api/v1/products
GET    /api/v1/products/:id
PATCH  /api/v1/products/:id
DELETE /api/v1/products/:id
```

Additional endpoints:

```text
GET  /api/v1/products/:id/stock
GET  /api/v1/products/:id/movements
GET  /api/v1/products/:id/variants
POST /api/v1/products/:id/variants
```

Product APIs must respect:

* Tenant scope
* Store scope where applicable
* Permissions
* Capability requirements

Product master data should be distinguished from store-specific stock and pricing data where the domain model requires it.

---

# 23. Product Search

POS and inventory workflows require fast product lookup.

Example:

```text
GET /api/v1/products/search?q=milk
```

Barcode lookup:

```text
GET /api/v1/products/barcode/:barcode
```

Search may support:

```text
Name
SKU
Barcode
Variant SKU
```

PostgreSQL indexes should be used before introducing a dedicated search engine.

The POS search path should be optimized for frequent lookups.

---

# 24. Category APIs

```text
GET    /api/v1/categories
POST   /api/v1/categories
GET    /api/v1/categories/:id
PATCH  /api/v1/categories/:id
DELETE /api/v1/categories/:id
```

All category operations are tenant-scoped.

---

# 25. Brand APIs

```text
GET    /api/v1/brands
POST   /api/v1/brands
GET    /api/v1/brands/:id
PATCH  /api/v1/brands/:id
DELETE /api/v1/brands/:id
```

All brand operations are tenant-scoped.

---

# 26. Inventory APIs

Core endpoints:

```text
GET  /api/v1/inventory
GET  /api/v1/inventory/:productId
GET  /api/v1/inventory/movements
POST /api/v1/inventory/adjustments
POST /api/v1/inventory/transfers
```

Inventory quantity must not be freely mutated through a generic update endpoint.

Avoid:

```text
PATCH /api/v1/inventory/:id

{
  "quantity": 500
}
```

Preferred:

```text
POST /api/v1/inventory/adjustments
```

The system records the business reason and corresponding stock movement.

PostgreSQL remains the authoritative source of inventory state.

---

# 27. Inventory Movement Model

Inventory-affecting business operations should create appropriate stock movements.

Examples:

```text
PURCHASE
SALE
RETURN
TRANSFER
DAMAGE
EXPIRY
ADJUSTMENT
```

The exact movement enum is defined by the database/domain model.

Conceptually:

```text
Business Event
      ↓
Stock Movement
      ↓
Inventory Balance Update
```

The movement and balance update must occur atomically where the operation is transactional.

---

# 28. Stock Adjustment API

```text
POST /api/v1/inventory/adjustments
```

Example:

```json
{
  "productId": "product_id",
  "storeId": "store_id",
  "countedQuantity": 95,
  "reason": "Physical stock count correction"
}
```

The backend should:

```text
Validate Permission
       ↓
Resolve Tenant
       ↓
Validate Store Scope
       ↓
Validate Product
       ↓
Read Current Stock
       ↓
Calculate Adjustment Delta
       ↓
Create Adjustment / Movement
       ↓
Update Inventory Balance
       ↓
Create Audit Record
       ↓
COMMIT
```

The operation must be transactional.

The client should not be trusted to provide the final adjustment delta without server-side verification.

---

# 29. Stock Transfer API

```text
POST /api/v1/inventory/transfers
GET  /api/v1/inventory/transfers
GET  /api/v1/inventory/transfers/:id
```

Conceptual flow:

```text
Source Store
     ↓
Transfer
     ↓
Destination Store
```

A completed transfer should produce corresponding inventory movements, such as:

```text
TRANSFER_OUT
TRANSFER_IN
```

The transfer must validate:

* Source scope
* Destination scope
* Product
* Available quantity
* Authorization
* Transfer state

The exact transaction model is defined in the inventory design.

---

# 30. Supplier APIs

```text
GET    /api/v1/suppliers
POST   /api/v1/suppliers
GET    /api/v1/suppliers/:id
PATCH  /api/v1/suppliers/:id
DELETE /api/v1/suppliers/:id
```

Supplier operations are tenant-scoped.

---

# 31. Purchase APIs

Purchase APIs may support:

```text
GET  /api/v1/purchases
POST /api/v1/purchases
GET  /api/v1/purchases/:id
PATCH /api/v1/purchases/:id
POST /api/v1/purchases/:id/receive
POST /api/v1/purchases/:id/cancel
```

The exact purchase lifecycle depends on the implemented purchasing model.

A purchase order is not required to be a mandatory step if the MVP uses direct purchase/receiving entry.

Possible workflow:

```text
Supplier
   ↓
Purchase / Purchase Order
   ↓
Receive Goods
   ↓
Verify Quantities / Cost
   ↓
Inventory Movement
   ↓
Inventory Balance
```

---

# 32. Purchase Receiving

```text
POST /api/v1/purchases/:id/receive
```

Conceptual flow:

```text
Validate Purchase
       ↓
Validate Tenant / Store Scope
       ↓
Validate Receiving Quantities
       ↓
BEGIN TRANSACTION
       ↓
Create Receiving Record
       ↓
Create Inventory Movements
       ↓
Update Inventory Balance
       ↓
Update Purchase Status
       ↓
Create Audit Record
       ↓
COMMIT
```

Receiving must be idempotent where duplicate requests could create duplicate stock.

---

# 33. Customer APIs

```text
GET    /api/v1/customers
POST   /api/v1/customers
GET    /api/v1/customers/:id
PATCH  /api/v1/customers/:id
DELETE /api/v1/customers/:id
```

Additional:

```text
GET /api/v1/customers/:id/sales
GET /api/v1/customers/:id/payments
GET /api/v1/customers/:id/summary
```

A customer is optional for ordinary walk-in POS sales unless a specific business rule requires customer identification.

Therefore:

```text
Walk-in Sale
```

is valid without creating a customer record.

---

# 34. POS APIs

POS APIs must prioritize speed and transaction correctness.

The POS cart can primarily remain client-side until checkout.

Possible endpoints:

```text
GET  /api/v1/pos/products/search
POST /api/v1/sales
GET  /api/v1/sales/:id
```

A server-side cart/session API may be introduced if required for:

* Multi-device workflows
* Suspended carts
* Persistent carts
* Complex POS sessions

The initial architecture does not require a server-side cart for every checkout.

---

# 35. Sale Creation API

Primary endpoint:

```text
POST /api/v1/sales
```

Example:

```json
{
  "storeId": "store_id",
  "customerId": "customer_id",
  "items": [
    {
      "productId": "product_id",
      "variantId": "variant_id",
      "quantity": 2
    }
  ],
  "payments": [
    {
      "method": "UPI",
      "amount": 500
    }
  ]
}
```

The server must calculate authoritative values.

The client must not be trusted for:

```text
Final Total
Stock Availability
Tax Amount
Discount Amount
Unit Price
```

where those values are determined by server-side business rules.

---

# 36. Critical Sale Transaction

The sale operation conceptually follows:

```text
Request
   ↓
Authentication
   ↓
Tenant Resolution
   ↓
Store Scope
   ↓
Permission Check
   ↓
Input Validation
   ↓
Load Products
   ↓
Resolve Prices
   ↓
Validate Stock
   ↓
Calculate Totals
   ↓
BEGIN TRANSACTION
   ↓
Create Sale
   ↓
Create Sale Items
   ↓
Create Payment Records
   ↓
Create Inventory Movements
   ↓
Update Inventory Balance
   ↓
Create Invoice Record
   ↓
Create Audit Record
   ↓
COMMIT
```

If a critical operation fails:

```text
ROLLBACK
```

The exact internal database ordering may differ as long as the required invariants are preserved.

---

# 37. Sale Transaction Boundary

The core sale transaction should contain only operations required to finalize the business transaction.

Do not perform long-running work inside the critical transaction.

Avoid:

```text
Sale Transaction
   ↓
Generate PDF
   ↓
Call AI
   ↓
Send Email
   ↓
Send WhatsApp
   ↓
External Analytics
   ↓
COMMIT
```

Preferred:

```text
Sale Transaction
   ↓
Commit
   ↓
Post-Commit Events / Jobs
   ├── Generate Invoice PDF
   ├── Send Notification
   ├── Send Email / WhatsApp
   ├── Update Analytics
   └── Trigger AI-related processing
```

The sale should not fail because a non-critical notification provider is temporarily unavailable.

---

# 38. Idempotency

Critical APIs must support idempotency where duplicate requests can create financial or inventory problems.

Especially:

```text
POST /sales
POST /payments
POST /refunds
POST /purchases/:id/receive
```

Example:

```text
Idempotency-Key: unique-client-operation-id
```

The server should associate the key with the operation and prevent accidental duplicate processing.

This protects against:

* Double-clicks
* Network retries
* Browser reconnects
* Mobile connectivity problems
* Payment retries
* Webhook duplicate delivery

Idempotency implementation must be tenant-aware and transaction-safe.

---

# 39. Payment APIs

Core endpoints may include:

```text
POST /api/v1/payments
GET  /api/v1/payments/:id
POST /api/v1/payments/:id/refund
```

Payment methods may include:

```text
Cash
UPI
Card
Other configured methods
```

External payment integrations may use:

```text
POST /api/v1/payments/webhooks/:provider
```

Webhook authenticity must be verified before processing.

Payment state should be explicit.

Example:

```text
PENDING
AUTHORIZED
PAID
FAILED
REFUNDED
PARTIALLY_REFUNDED
```

The exact states depend on the payment integration model.

---

# 40. Payment and Sale Consistency

For internal/manual payment methods, the sale transaction can coordinate:

```text
Sale
Payment
Inventory
Invoice
```

within the appropriate database transaction.

External payment providers require additional handling.

Example:

```text
Create Payment Intent
        ↓
External Provider
        ↓
Provider Confirmation
        ↓
Webhook
        ↓
Verify Signature
        ↓
Idempotency Check
        ↓
Update Payment State
        ↓
Finalize Appropriate Business State
```

The system must not assume that an external payment request succeeding means the webhook or final settlement has already been safely recorded.

Payment reconciliation must handle delayed, duplicated, or missing callbacks.

---

# 41. Split Payments

A sale may support multiple payment records when the business capability requires it.

Example:

```text
Sale Total = ₹1,000

Cash = ₹400
UPI  = ₹600
```

The API should represent these as separate payment allocations rather than overwriting a single payment record.

The service must verify:

```text
sum(payment allocations)
=
amount required for settlement
```

subject to the supported credit/partial-payment rules.

---

# 42. Credit / Receivables

Credit sales must not be represented as an ordinary successful payment.

Where credit functionality is enabled:

```text
Sale
 ↓
Receivable / Outstanding Balance
 ↓
Future Payment
```

The exact accounting model belongs to the financial/accounting design.

Credit functionality should only be exposed when the tenant capability and permissions permit it.

---

# 43. Invoice APIs

```text
GET /api/v1/invoices
GET /api/v1/invoices/:id
GET /api/v1/invoices/:id/pdf
```

The sale transaction should create the authoritative invoice record when required.

The actual PDF generation may occur asynchronously after the transaction commits.

Example:

```text
Sale Transaction
      ↓
Invoice Record
      ↓
COMMIT
      ↓
Invoice PDF Job
      ↓
Storage
```

Finalized invoices should not be arbitrarily modified through generic update endpoints.

Corrections should use supported business workflows such as cancellation, credit note, return, or replacement where applicable.

---

# 44. Returns APIs

```text
POST /api/v1/sales/:id/returns
GET  /api/v1/returns
GET  /api/v1/returns/:id
POST /api/v1/returns/:id/refund
```

Return processing should:

```text
Validate Original Sale
       ↓
Validate Returnable Quantity
       ↓
Validate Authorization
       ↓
Determine Return Disposition
       ↓
Create Return Record
       ↓
Create Inventory Movement if applicable
       ↓
Process Refund / Credit if applicable
       ↓
Audit
```

Not every returned item must automatically return to sellable stock.

Examples:

```text
Resellable → RETURN
Damaged    → DAMAGE
Expired    → EXPIRY
```

The exact disposition rules belong to inventory/business logic.

---

# 45. Analytics APIs

Analytics APIs are primarily read-focused.

Examples:

```text
GET /api/v1/analytics/dashboard
GET /api/v1/analytics/sales
GET /api/v1/analytics/products
GET /api/v1/analytics/inventory
GET /api/v1/analytics/customers
GET /api/v1/analytics/profit
```

Analytics must not directly mutate transactional business records.

Analytics may consume:

```text
Sales
Purchases
Inventory Movements
Payments
Customers
```

through queries, aggregation, read models, or background processing.

---

# 46. Reporting APIs

Examples:

```text
GET /api/v1/reports/sales
GET /api/v1/reports/inventory
GET /api/v1/reports/purchases
GET /api/v1/reports/customers
GET /api/v1/reports/profit
```

Large reports may be processed asynchronously.

Example:

```text
POST /api/v1/reports/generate
```

Response:

```json
{
  "success": true,
  "data": {
    "jobId": "job_id",
    "status": "QUEUED"
  }
}
```

Status:

```text
GET /api/v1/reports/jobs/:jobId
```

---

# 47. AI APIs

Buzzsynx AI provides business intelligence rather than unrestricted database access.

Examples:

```text
GET  /api/v1/ai/insights
GET  /api/v1/ai/recommendations
GET  /api/v1/ai/forecasts
POST /api/v1/ai/analyze/sales
POST /api/v1/ai/analyze/inventory
```

AI processing may be synchronous for lightweight operations or asynchronous for expensive operations.

Example:

```text
POST /api/v1/ai/analyze/inventory
```

Response:

```json
{
  "success": true,
  "data": {
    "jobId": "job_id",
    "status": "QUEUED"
  }
}
```

Worker:

```text
BullMQ
   ↓
AI Service
   ↓
Approved Business Data
   ↓
Analysis
   ↓
AI Insight
   ↓
PostgreSQL
```

---

# 48. AI Safety Boundary

AI must remain inside controlled business workflows.

Incorrect:

```text
AI
 ↓
PostgreSQL
 ↓
Inventory Mutation
```

Preferred:

```text
Business Data
      ↓
AI
      ↓
Insight / Recommendation
      ↓
User or Deterministic Business Rule
      ↓
Normal Application Service
      ↓
Validation
      ↓
Transaction
      ↓
PostgreSQL
```

AI must not directly bypass:

* Tenant isolation
* Authorization
* Inventory rules
* Payment rules
* Financial controls
* Audit requirements

AI output is advisory unless a deterministic business workflow explicitly defines an automated action.

---

# 49. AI Data Access

AI services should receive only the data required for the requested analysis.

AI providers should not receive unrestricted database credentials.

Preferred:

```text
API
 ↓
AI Service
 ↓
Approved Query / Analytics Layer
 ↓
Sanitized Business Data
 ↓
AI Provider
```

AI output should be traceable to:

* Tenant
* Relevant business scope
* Analysis type
* Time range
* Source data or metrics where practical
* Model/provider metadata where required

---

# 50. Notification APIs

User-facing notification APIs:

```text
GET   /api/v1/notifications
PATCH /api/v1/notifications/:id/read
PATCH /api/v1/notifications/read-all
```

Notifications may be created through background jobs.

Example:

```text
Low Stock Signal
      ↓
BullMQ
      ↓
Notification Worker
      ↓
Notification
```

Notification failure should not normally roll back a completed business transaction.

---

# 51. User APIs

```text
GET   /api/v1/users/me
PATCH /api/v1/users/me
GET   /api/v1/users
GET   /api/v1/users/:id
PATCH /api/v1/users/:id
PATCH /api/v1/users/:id/status
```

Administrative operations require appropriate permissions.

User operations must respect tenant and store membership boundaries.

---

# 52. Membership APIs

Because Buzzsynx is multi-tenant, user membership should be treated separately from the global user identity.

Possible endpoints:

```text
GET   /api/v1/members
POST  /api/v1/members
GET   /api/v1/members/:id
PATCH /api/v1/members/:id
PATCH /api/v1/members/:id/status
```

Membership may define:

```text
User
Tenant
Role
Store Scope
Status
```

A single user may eventually belong to multiple tenants, subject to the account model.

---

# 53. Role and Permission APIs

Possible endpoints:

```text
GET   /api/v1/roles
POST  /api/v1/roles
PATCH /api/v1/roles/:id

GET   /api/v1/permissions
```

Role assignment should be handled through membership operations rather than assuming a global user role.

Example:

```text
PATCH /api/v1/members/:id/role
```

Changing high-privilege roles should require appropriate safeguards and auditing.

The exact custom-role capability may be introduced later if required.

---

# 54. Tenant APIs

Current tenant APIs may include:

```text
GET   /api/v1/tenants/current
PATCH /api/v1/tenants/current

GET   /api/v1/tenants/current/settings
PATCH /api/v1/tenants/current/settings
```

Tenant-level administrative APIs may include:

```text
GET  /api/v1/tenants
POST /api/v1/tenants/:id/suspend
POST /api/v1/tenants/:id/activate
```

These platform-level operations require Super Admin authorization.

---

# 55. Store / Branch APIs

Because Buzzsynx supports multiple stores/branches:

```text
GET   /api/v1/stores
POST  /api/v1/stores
GET   /api/v1/stores/:id
PATCH /api/v1/stores/:id
PATCH /api/v1/stores/:id/status
```

Store operations must verify tenant ownership.

Store-level users must only access stores assigned to them unless their role explicitly grants broader tenant scope.

---

# 56. Industry Capability APIs

Industry capabilities are extensibility boundaries.

They should not make pharmacy, clothing, restaurant, and supermarket functionality mandatory for the initial MVP.

The first complete implementation is:

```text
Supermarket / Grocery
```

Other industries are future capability extensions.

Where required, industry-specific APIs may be namespaced.

### Pharmacy — Future Capability

```text
GET  /api/v1/pharmacy/medicines
POST /api/v1/pharmacy/medicines
GET  /api/v1/pharmacy/batches
POST /api/v1/pharmacy/batches
GET  /api/v1/pharmacy/expiry
```

### Restaurant — Future Capability

```text
GET  /api/v1/restaurant/menu
POST /api/v1/restaurant/menu
GET  /api/v1/restaurant/ingredients
POST /api/v1/restaurant/recipes
GET  /api/v1/restaurant/tables
```

### Clothing — Future Capability

Clothing may primarily use shared product/variant APIs:

```text
/api/v1/products/:id/variants
```

with additional clothing-specific capability validation.

### Supermarket / Grocery — Initial Capability

The initial implementation primarily uses:

```text
products
inventory
purchases
pos
sales
payments
customers
```

with capabilities such as:

```text
barcode
bulk units
categories
stock alerts
expiry tracking where applicable
```

---

# 57. API Request Validation

Every write endpoint must validate incoming data.

```text
Request
   ↓
Schema Validation
   ↓
Business Validation
   ↓
Service
```

Validation should cover:

* Required fields
* Data types
* String length
* Enum values
* Numeric ranges
* Identifiers
* Dates
* Arrays
* Nested objects

Zod or the project's selected validation layer should be used consistently.

Frontend validation does not replace server-side validation.

---

# 58. Business Validation

Schema validation alone is insufficient.

Example:

```text
quantity = 5
```

may be structurally valid.

But:

```text
available stock = 2
```

makes the requested sale invalid.

Therefore:

```text
Input Validation
       +
Business Validation
       ↓
Valid Operation
```

Business validation belongs in the appropriate service/domain layer.

---

# 59. Response Format

Successful responses should follow a consistent structure.

Example:

```json
{
  "success": true,
  "data": {
    "id": "product_id",
    "name": "Example Product"
  }
}
```

List response:

```json
{
  "success": true,
  "data": [],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "totalPages": 5
  }
}
```

For asynchronous operations:

```json
{
  "success": true,
  "data": {
    "jobId": "job_id",
    "status": "QUEUED"
  }
}
```

The response contract should remain consistent across modules.

---

# 60. Error Response Format

Errors should use a consistent structure.

Example:

```json
{
  "success": false,
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product was not found.",
    "requestId": "req_123"
  }
}
```

Validation error:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed.",
    "fields": {
      "name": "Name is required."
    },
    "requestId": "req_123"
  }
}
```

The response must not expose stack traces or sensitive internal implementation details in production.

---

# 61. Error Codes

Application error codes must be stable and machine-readable.

Examples:

```text
AUTH_INVALID_CREDENTIALS
AUTH_UNAUTHORIZED
AUTH_FORBIDDEN

TENANT_NOT_FOUND
TENANT_ACCESS_DENIED
STORE_NOT_FOUND
STORE_ACCESS_DENIED

PRODUCT_NOT_FOUND
PRODUCT_SKU_EXISTS

INSUFFICIENT_STOCK
INVALID_STOCK_ADJUSTMENT

SALE_NOT_FOUND
SALE_ALREADY_COMPLETED

PAYMENT_FAILED
PAYMENT_ALREADY_PROCESSED
PAYMENT_RECONCILIATION_REQUIRED

RETURN_NOT_FOUND
RETURN_QUANTITY_EXCEEDED

VALIDATION_ERROR
RESOURCE_NOT_FOUND
CONFLICT
RATE_LIMITED
INTERNAL_ERROR
```

Frontend clients should rely on error codes rather than parsing human-readable messages.

---

# 62. HTTP Status Codes

Recommended conventions:

```text
200 OK
201 Created
202 Accepted
204 No Content

400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Entity
429 Too Many Requests

500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
```

The exact status should reflect the actual failure semantics.

---

# 63. Pagination

Collection endpoints should support pagination.

Example:

```text
GET /api/v1/products?page=1&limit=20
```

Response:

```json
{
  "success": true,
  "data": [],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 250,
    "totalPages": 13
  }
}
```

For very large datasets, cursor-based pagination may be introduced.

Maximum page sizes should be enforced server-side.

---

# 64. Filtering

Examples:

```text
GET /api/v1/products?status=ACTIVE

GET /api/v1/sales?status=COMPLETED

GET /api/v1/inventory?lowStock=true
```

Filters must be explicitly supported and validated.

The API must never accept arbitrary database column names from clients.

---

# 65. Sorting

Example:

```text
GET /api/v1/products?sortBy=createdAt&sortOrder=desc
```

Supported sort fields must be explicitly whitelisted.

This prevents unsafe dynamic query construction and accidental expensive database queries.

---

# 66. Date Filtering

Example:

```text
GET /api/v1/sales?from=2026-09-01&to=2026-09-30
```

The backend must validate:

```text
Date format
Timezone interpretation
from <= to
Maximum allowed range where appropriate
```

The API should use a consistent timezone policy.

Business reporting timezone should be derived from tenant/business configuration where required.

---

# 67. API Security

The API must implement:

```text
Authentication
Authorization
Tenant isolation
Store isolation
Input validation
Rate limiting
Secure headers
CORS restrictions
Request size limits
SQL injection protection
Sensitive data protection
Audit logging
```

Prisma parameterization provides strong protection against ordinary SQL injection.

Raw SQL must still be reviewed and parameterized safely.

---

# 68. Rate Limiting

Redis may be used for distributed rate limiting.

Different endpoint categories may have different limits.

Examples:

```text
Login
Password Reset
AI Endpoints
Public APIs
Search
Webhooks
```

AI endpoints may require stricter limits because of provider costs.

When a limit is exceeded:

```text
429 Too Many Requests
```

Rate limits should account for authenticated tenant/user context where appropriate.

---

# 69. CORS

Production APIs should allow only approved origins.

Development:

```text
http://localhost:3000
```

Production examples:

```text
https://buzzsynx.com
https://app.buzzsynx.com
```

Exact domains will be finalized during deployment.

Wildcard CORS should not be used for authenticated production APIs without a documented reason.

---

# 70. API Logging

Operational logs should contain useful information such as:

```text
requestId
method
path
status
duration
userId where appropriate
tenantId where appropriate
storeId where appropriate
errorCode
```

Do not log:

```text
Passwords
Authentication tokens
Payment secrets
API keys
Sensitive personal information
Full payment credentials
```

Logging must follow the application's data protection requirements.

---

# 71. Audit Logging

Business-critical operations should generate audit records.

Examples:

```text
Product Created
Product Updated
Stock Adjusted
Purchase Received
Sale Completed
Sale Refunded
Payment Processed
Role Changed
Membership Changed
Settings Changed
Tenant Suspended
```

Audit records should capture appropriate context, such as:

```text
tenantId
storeId
userId
action
entity
entityId
timestamp
requestId
reason where applicable
```

For critical business operations, audit creation should occur inside the same transaction or through an equivalent reliable mechanism.

Audit logs are different from technical application logs.

---

# 72. Database Transactions in APIs

Controllers must not independently manipulate multiple business entities.

Avoid:

```text
Controller
   ↓
Sale
   ↓
Inventory
   ↓
Payment
   ↓
Invoice
```

inside the controller.

Prefer:

```text
Controller
    ↓
SaleService
    ↓
Transaction
    ├── Sale
    ├── SaleItems
    ├── Payments
    ├── Inventory Movements
    ├── Inventory Balance
    ├── Invoice
    └── Audit
```

The service owns the business transaction boundary.

---

# 73. Service Layer

Business logic belongs in services.

Example:

```javascript
async function createSale(context, input) {
  // business workflow
}
```

The service receives trusted context such as:

```text
userId
tenantId
membershipId
permissions
activeStoreId
capabilities
```

The service must not trust arbitrary tenant or authorization values supplied in the request body.

---

# 74. Controller Layer

Controllers should remain thin.

Responsibilities:

```text
Read Request
      ↓
Pass Validated Input
      ↓
Call Service
      ↓
Format Response
```

Avoid:

```javascript
// hundreds of lines of sale logic inside controller
```

Prefer:

```text
Controller
    ↓
SaleService
    ↓
Domain / Application Logic
    ↓
Repositories
```

---

# 75. Repository Layer

Repositories/data-access functions encapsulate database access where useful.

Examples:

```text
ProductRepository
SaleRepository
InventoryRepository
CustomerRepository
PurchaseRepository
```

Repositories should not decide whether a user is allowed to perform an operation.

Authorization belongs at the application/service boundary.

Repositories should still enforce required tenant/store query constraints passed by trusted application context.

---

# 76. API and Background Jobs

Long-running or non-critical operations should not block synchronous API requests unnecessarily.

Use BullMQ for operations such as:

```text
AI analysis
Large reports
Email delivery
Notification processing
Data aggregation
Scheduled tasks
Bulk imports
Export generation
Invoice PDF generation
```

Example:

```text
POST /api/v1/reports/generate
        ↓
Create Job
        ↓
Return 202
        ↓
BullMQ
        ↓
Worker
        ↓
Generate Report
        ↓
Store Result
```

Background jobs must support appropriate:

* Retries
* Idempotency
* Failure handling
* Logging
* Monitoring

---

# 77. Post-Commit Processing

Business transactions must commit before non-critical background work is dispatched where practical.

Preferred:

```text
BEGIN TRANSACTION
      ↓
Business Changes
      ↓
COMMIT
      ↓
Reliable Event / Job Dispatch
      ↓
BullMQ Worker
```

For critical event delivery requirements, a transactional outbox or equivalent reliable post-commit mechanism may be introduced.

The API must avoid creating a situation where a transaction succeeds but required background processing is silently lost.

---

# 78. API Timeout Strategy

API endpoints must not wait indefinitely for external services.

Especially:

```text
AI Providers
Payment Providers
Email Providers
Storage Providers
External APIs
```

Use:

```text
Timeouts
Retries where safe
Fallbacks where appropriate
Async processing
Circuit-breaking where justified
```

Do not blindly retry financial operations.

Retries must respect idempotency.

---

# 79. Webhooks

External webhook endpoints must:

1. Verify authenticity/signature.
2. Validate payload structure.
3. Validate event type.
4. Resolve the related business context.
5. Check idempotency.
6. Process safely.
7. Record the event/result.
8. Return an appropriate response.

Example:

```text
Payment Provider
      ↓
Webhook
      ↓
Verify Signature
      ↓
Validate Payload
      ↓
Check Idempotency
      ↓
Process Payment State
      ↓
Update Database
```

Webhook handlers must support duplicate delivery.

Webhook endpoints must not trust arbitrary tenant IDs supplied in webhook payloads without resolving them from trusted provider/business identifiers.

---

# 80. API Documentation

Buzzsynx should maintain OpenAPI documentation.

Potential endpoint:

```text
/api/docs
```

Documentation should include:

```text
Endpoint
Method
Authentication
Permissions
Tenant Scope
Store Scope
Capability Requirements
Parameters
Request Body
Response
Errors
Examples
```

OpenAPI can later support:

* Frontend integration
* Mobile integration
* Third-party integrations
* Contract testing
* API testing

---

# 81. API Testing

API tests should cover critical business behavior.

### Authentication

```text
Register
Login
Logout
Invalid credentials
Session expiry
```

### Tenant Isolation

```text
Tenant A → Tenant A data
Tenant A → Cannot access Tenant B data
```

### Store Isolation

```text
Store A User → Store A data
Store A User → Cannot access Store B data without permission
```

### RBAC

```text
Allowed operation
Forbidden operation
```

### Products

```text
Create
Read
Update
Delete
Search
```

### Inventory

```text
Purchase
Sale
Adjustment
Transfer
Return
```

### POS

```text
Create Sale
Stock Validation
Payment
Invoice
Rollback
Duplicate Request
```

### Payments

```text
Success
Failure
Duplicate
Refund
Webhook
Reconciliation
```

### AI

```text
Queue Job
Worker Processing
Insight Creation
Failure Handling
Tenant Isolation
```

Critical financial and inventory workflows require stronger integration testing than ordinary CRUD endpoints.

---

# 82. API Performance

Performance should be measured rather than assumed.

Important metrics include:

```text
Average Latency
P95 Latency
P99 Latency
Requests Per Second
Error Rate
Database Query Duration
Redis Latency
External Provider Latency
Queue Processing Time
```

Optimization sequence:

```text
Correctness
    ↓
Measure
    ↓
Optimize Database Queries
    ↓
Add Indexes
    ↓
Cache Safe Reads
    ↓
Background Processing
    ↓
Scale Infrastructure
```

Do not introduce infrastructure complexity before identifying an actual bottleneck.

---

# 83. API Caching

Redis may be used for safe read-heavy operations.

Examples:

```text
Product Lookup
Business Settings
Static Configuration
Dashboard Summaries
```

Caching must have a clear invalidation or freshness strategy.

Avoid caching highly volatile transactional state without understanding consistency implications.

For example:

```text
POS Stock Availability
```

must use authoritative transactional data when finalizing a sale.

PostgreSQL remains the source of truth.

---

# 84. API Compatibility

API contracts should be treated as public interfaces.

Changes should avoid unexpectedly breaking:

```text
Frontend
Mobile Clients
Third-Party Integrations
Webhooks
Automations
```

Breaking changes require:

```text
Versioning
```

or a controlled migration/deprecation strategy.

---

# 85. API Folder Structure

Recommended backend structure:

```text
src/server/

├── index.js
│
├── modules/
│   ├── auth/
│   │   ├── auth.routes.js
│   │   ├── auth.controller.js
│   │   ├── auth.service.js
│   │   ├── auth.repository.js
│   │   └── auth.validation.js
│   │
│   ├── tenants/
│   ├── users/
│   ├── roles/
│   ├── products/
│   ├── inventory/
│   ├── pos/
│   ├── sales/
│   ├── purchases/
│   ├── suppliers/
│   ├── customers/
│   ├── payments/
│   ├── analytics/
│   ├── reports/
│   ├── ai/
│   └── notifications/
│
├── middleware/
│   ├── auth.js
│   ├── tenant.js
│   ├── permissions.js
│   ├── capability.js
│   ├── store-scope.js
│   ├── validation.js
│   ├── rate-limit.js
│   ├── error-handler.js
│   └── request-id.js
│
├── jobs/
└── queues/
```

---

# 86. API Development Order

The API should be implemented according to the product execution roadmap.

## Phase 1 — Platform Foundation

```text
Health
Authentication
Tenant Creation
Tenant Resolution
Stores
Memberships
Users
Roles
Permissions
```

## Phase 2 — Product Foundation

```text
Categories
Brands
Products
Variants
Product Search
```

## Phase 3 — Inventory

```text
Stock
Movements
Adjustments
Transfers
```

## Phase 4 — Purchasing

```text
Suppliers
Purchases
Receiving
```

## Phase 5 — POS and Sales

```text
Customers
POS Search
Sales
Payments
Invoices
```

## Phase 6 — Returns

```text
Returns
Refunds
Inventory Return Disposition
```

## Phase 7 — Supermarket Capability

```text
Barcode
Bulk Units
Retail Pricing
Stock Alerts
Relevant Expiry Tracking
```

## Phase 8 — Intelligence

```text
Analytics
Reports
AI Insights
AI Recommendations
Notifications
```

## Phase 9 — Future Industry Capabilities

```text
Pharmacy
Clothing
Restaurant
Other Validated Industry Extensions
```

## Phase 10 — Production Hardening

```text
Rate Limiting
Caching
Observability
Security Hardening
Performance
OpenAPI Documentation
Load Testing
Deployment Hardening
```

The existence of an API section does not mean that the corresponding feature is already implemented.

---

# 87. API Definition of Done

An API module is considered implementation-complete when applicable:

* [ ] Route is defined.
* [ ] Authentication requirement is defined.
* [ ] Tenant scope is defined.
* [ ] Store scope is defined where applicable.
* [ ] Permission is defined.
* [ ] Capability requirement is defined where applicable.
* [ ] Request schema is validated.
* [ ] Business validation is implemented.
* [ ] Controller is implemented.
* [ ] Service is implemented.
* [ ] Repository/data access is implemented where appropriate.
* [ ] Transaction boundary is defined where required.
* [ ] Error codes are defined.
* [ ] Response format is consistent.
* [ ] Audit logging is implemented where required.
* [ ] Idempotency is implemented where required.
* [ ] Background processing is implemented where required.
* [ ] Tests are implemented according to criticality.
* [ ] API documentation is updated.
* [ ] Observability requirements are satisfied.

---

# 88. API Design Principles

The following principles are mandatory.

### 1. Server is authoritative

Never trust client-calculated financial, inventory, tax, discount, or settlement values.

### 2. Tenant isolation is non-negotiable

Every tenant operation must execute inside a verified tenant context.

### 3. Store scope is explicit

Users must only access stores/branches permitted by their membership and permissions.

### 4. Authentication and authorization are separate

Knowing who the user is does not determine what they can do.

### 5. Controllers stay thin

Business logic belongs in services.

### 6. Critical operations are transactional

Sales, inventory movements, receiving, refunds, and other critical financial operations require carefully defined transaction boundaries.

### 7. APIs are idempotent where necessary

Financial, inventory, and webhook operations must handle retries safely.

### 8. Errors are structured

Clients should receive stable machine-readable error codes.

### 9. PostgreSQL is authoritative

Redis is an optimization/supporting layer, not the transactional source of truth.

### 10. Background work is asynchronous

Non-critical or long-running work should not unnecessarily block critical business transactions.

### 11. AI remains inside controlled workflows

AI provides intelligence and recommendations but cannot bypass normal business rules.

### 12. Capabilities are backend-enforced

Industry capabilities must be enforced by the API, not only hidden or shown by the frontend.

### 13. Auditability matters

Critical business operations must be traceable.

### 14. API contracts are documented

Production APIs must have clear and maintainable contracts.

### 15. Avoid premature complexity

Do not introduce microservices, dedicated search infrastructure, or other distributed-system complexity until actual product requirements justify them.

---

# 89. Final API Architecture

```text
                         CLIENT
                           │
                    Next.js / Mobile
                           │
                           ▼
                    REST API /v1
                           │
                           ▼
                  ┌─────────────────┐
                  │   Middleware    │
                  │                 │
                  │ Request ID      │
                  │ Authentication  │
                  │ Tenant Context  │
                  │ Membership      │
                  │ Store Scope     │
                  │ RBAC            │
                  │ Capability      │
                  │ Validation      │
                  │ Rate Limit      │
                  └────────┬────────┘
                           │
                           ▼
                     Controllers
                           │
                           ▼
                       Services
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         PostgreSQL      Redis       BullMQ
              │                         │
              │                      Workers
              │                         │
              │              ┌──────────┼──────────┐
              │              │          │          │
              │             AI       Reports   Notifications
              │
              ▼
        Business Data
```

External services:

```text
                    Services
                       │
        ┌──────────────┼───────────────┐
        │              │               │
        ▼              ▼               ▼
 Payment Providers   AI Providers   Storage
        │
        ▼
    Webhooks
        │
        ▼
 Verification
        │
        ▼
 Application Service
```

---

# 90. Final API Request Principle

The Buzzsynx API follows:

```text
Request
   ↓
Authenticate
   ↓
Resolve Tenant
   ↓
Resolve Membership
   ↓
Resolve Store Scope
   ↓
Authorize
   ↓
Check Capability
   ↓
Validate Input
   ↓
Execute Business Service
   ↓
Validate Business Rules
   ↓
Persist Transaction
   ↓
Commit
   ↓
Dispatch Non-Critical Background Work
   ↓
Respond
```

For critical business operations:

```text
Client
  ↓
API
  ↓
Application Service
  ↓
PostgreSQL Transaction
  ↓
Commit
  ↓
Async Processing
```

The core principle is:

> **The API enforces who can perform an operation, the service enforces how the business operation works, PostgreSQL records what actually happened, and background systems handle non-critical work after the transaction.**

---

# 91. Related Documentation

This document should remain aligned with:

```text
00-project-overview.md
01-system-architecture.md
02-system-workflow.md
03-database-design.md
05-multi-tenancy.md
06-industry-capabilities.md
07-security.md
08-ai-architecture.md
09-caching-and-queues.md
10-testing-strategy.md
11-devops.md
12-aws-infrastructure.md
13-observability.md
14-development-standards.md
15-phase-wise-execution.md
16-feature-checklist.md
17-production-readiness.md
18-project-completion.md
```

If a conflict exists between API behavior and the architecture/database/security documents, the documents must be reconciled before implementation.

---

# 92. API Scope Statement

The API design intentionally provides a **complete architectural direction without requiring every endpoint to be implemented immediately**.

The initial implementation priority is:

```text
Multi-Tenancy
      ↓
Authentication
      ↓
RBAC + Store Scope
      ↓
Products
      ↓
Inventory
      ↓
Purchasing
      ↓
POS
      ↓
Sales
      ↓
Payments
      ↓
Invoices
      ↓
Analytics
      ↓
AI
```

The first complete business workflow is:

```text
Supermarket / Grocery
```

Other industry APIs remain extensibility points until their corresponding capabilities are validated and implemented.

**Buzzsynx API principle:**

> **One modular API. One business engine. Strong tenant isolation. Transaction-safe operations. Industry capabilities where needed. AI as intelligence, not authority.**
