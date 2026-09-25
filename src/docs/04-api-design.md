# Buzzsynx — API Design

## 1. Purpose

This document defines the API architecture and standards for Buzzsynx.

Buzzsynx uses a backend API to expose business functionality to the Next.js frontend and other authorized clients.

The API must support:

* Multi-tenant SaaS operations
* Authentication
* Authorization and RBAC
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
* AI capabilities
* Notifications
* Industry-specific capabilities
* Audit logging

The API must be designed for security, consistency, maintainability, and future scalability.

---

# 2. API Architecture

Buzzsynx uses:

```text
Next.js Frontend
       ↓
HTTP / REST API
       ↓
Node.js + Express
       ↓
Application Modules
       ↓
Services
       ↓
Repositories / Prisma
       ↓
PostgreSQL
```

Supporting infrastructure:

```text
Express
 ├── Redis
 ├── BullMQ
 ├── AI Providers
 ├── S3 / Storage
 ├── Payment Providers
 └── External Services
```

The API is part of the modular monolith and follows clear domain boundaries.

---

# 3. API Style

Buzzsynx initially uses a **RESTful API**.

Example:

```text
GET    /api/v1/products
GET    /api/v1/products/:id
POST   /api/v1/products
PATCH  /api/v1/products/:id
DELETE /api/v1/products/:id
```

REST is preferred initially because it provides:

* Simple client integration
* Clear HTTP semantics
* Easy debugging
* Good tooling
* Straightforward authorization
* Easy testing
* Future mobile-app compatibility

GraphQL is not required for the initial architecture.

---

# 4. API Versioning

All public application APIs should use a version prefix.

Example:

```text
/api/v1
```

Example:

```text
/api/v1/products
/api/v1/inventory
/api/v1/sales
/api/v1/customers
```

Future breaking API changes can use:

```text
/api/v2
```

Versioning prevents breaking existing clients when major API contracts change.

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

The frontend must not hard-code environment-specific URLs.

---

# 6. API Module Structure

The Express backend should be organized by business module.

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

A typical module:

```text
products/
├── product.routes.js
├── product.controller.js
├── product.service.js
├── product.repository.js
├── product.validation.js
└── product.constants.js
```

The exact structure can evolve as implementation grows.

---

# 7. Request Lifecycle

Every authenticated tenant request should follow this conceptual flow:

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
RBAC
     ↓
Capability Check
     ↓
Validation
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

---

# 8. Request ID

Every API request should have a unique request identifier.

Example:

```text
X-Request-ID: req_01JXYZ...
```

If the client provides a valid request ID, the system may propagate it according to security rules.

Otherwise, the API should generate one.

Request IDs should be included in:

* Logs
* Error reports
* Sentry events
* Debugging information

This makes production troubleshooting significantly easier.

---

# 9. Authentication

Authentication determines **who the user is**.

Supported authentication methods may include:

```text
Email + Password
Google OAuth
```

Future options may include:

```text
Passkeys
Enterprise SSO
```

Authentication is separate from authorization.

```text
Authentication
     ↓
Who are you?

Authorization
     ↓
What are you allowed to do?
```

---

# 10. Authentication Endpoints

Initial endpoints:

```text
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/logout
POST /api/v1/auth/refresh
GET  /api/v1/auth/me
```

Potential future endpoints:

```text
POST /api/v1/auth/forgot-password
POST /api/v1/auth/reset-password
POST /api/v1/auth/verify-email
GET  /api/v1/auth/google
GET  /api/v1/auth/google/callback
```

---

# 11. Registration Workflow

Example:

```text
POST /api/v1/auth/register
```

Flow:

```text
Request
  ↓
Validate input
  ↓
Check email
  ↓
Hash password
  ↓
Create User
  ↓
Create Tenant
  ↓
Create TenantUser
  ↓
Assign Owner Role
  ↓
Create Business Settings
  ↓
Create Default Configuration
  ↓
Return authenticated session
```

This operation may require a database transaction.

---

# 12. Login Workflow

```text
POST /api/v1/auth/login
```

Flow:

```text
Credentials
     ↓
Validate input
     ↓
Find user
     ↓
Verify password
     ↓
Resolve tenant membership
     ↓
Resolve role
     ↓
Create session/token
     ↓
Return authenticated response
```

Failed authentication should return a generic authentication error rather than exposing sensitive account information.

---

# 13. Session Strategy

The exact session implementation will be finalized during authentication implementation.

The architecture must support:

* Secure session handling
* Token expiration
* Refresh/re-authentication
* Logout/revocation where applicable
* Secure cookies or equivalent secure token handling
* CSRF protection where cookie-based authentication requires it

Sensitive authentication tokens must never be logged.

---

# 14. Tenant Resolution

Tenant context is one of the most important API security requirements.

The API must establish:

```text
currentUser
currentTenant
currentRole
```

before executing tenant business operations.

Preferred conceptual flow:

```text
Authenticated User
       ↓
Tenant Membership
       ↓
Tenant Context
       ↓
RBAC
       ↓
Business Operation
```

The API must never blindly trust:

```text
tenantId
```

provided by the browser.

---

# 15. Tenant-Scoped API

Most business endpoints operate within the authenticated tenant context.

Example:

```text
GET /api/v1/products
```

The backend internally resolves:

```text
currentTenantId
```

and queries:

```javascript
where: {
  tenantId: currentTenantId
}
```

The client does not need to provide a tenant ID for normal tenant operations.

---

# 16. System-Level APIs

Some APIs may be system-level rather than tenant-level.

Examples:

```text
GET /api/v1/system/health
GET /api/v1/system/version
```

These should not use ordinary tenant authorization.

System endpoints must have their own security rules.

---

# 17. Authorization

Authorization is handled using:

```text
RBAC
+
Tenant Membership
+
Capability Checks
```

Example:

```text
User
 ↓
TenantUser
 ↓
Role
 ↓
Permissions
 ↓
Endpoint
```

---

# 18. Permission Naming

Permissions should follow a predictable convention.

Example:

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

report.view

settings.view
settings.update
```

This makes authorization rules easier to understand and maintain.

---

# 19. Capability Authorization

RBAC determines what a user can do.

Industry capability determines whether the tenant has a particular business feature.

Example:

```text
User Permission
      +
Tenant Capability
      ↓
Allowed Operation
```

Example:

```text
medicine.batch.manage
```

should only be available when the tenant has the pharmacy capability enabled.

Similarly:

```text
restaurant.recipe.manage
```

belongs to the restaurant capability.

---

# 20. Product APIs

Core endpoints:

```text
GET    /api/v1/products
POST   /api/v1/products
GET    /api/v1/products/:id
PATCH  /api/v1/products/:id
DELETE /api/v1/products/:id
```

Additional:

```text
GET /api/v1/products/:id/stock
GET /api/v1/products/:id/movements
GET /api/v1/products/:id/variants
POST /api/v1/products/:id/variants
```

---

# 21. Product Search

POS and inventory interfaces require fast product lookup.

Example:

```text
GET /api/v1/products/search?q=paracetamol
```

Barcode:

```text
GET /api/v1/products/barcode/:barcode
```

Search may support:

```text
name
SKU
barcode
variant SKU
```

Search performance should be optimized using PostgreSQL indexes before introducing a dedicated search engine.

---

# 22. Category APIs

```text
GET    /api/v1/categories
POST   /api/v1/categories
GET    /api/v1/categories/:id
PATCH  /api/v1/categories/:id
DELETE /api/v1/categories/:id
```

All tenant-scoped.

---

# 23. Brand APIs

```text
GET    /api/v1/brands
POST   /api/v1/brands
GET    /api/v1/brands/:id
PATCH  /api/v1/brands/:id
DELETE /api/v1/brands/:id
```

---

# 24. Inventory APIs

Core endpoints:

```text
GET  /api/v1/inventory
GET  /api/v1/inventory/:productId
GET  /api/v1/inventory/movements
POST /api/v1/inventory/adjustments
POST /api/v1/inventory/transfers
```

Inventory should not expose unrestricted quantity mutation.

Incorrect design:

```text
PATCH /inventory/:id
{
  "quantity": 500
}
```

Preferred:

```text
POST /api/v1/inventory/adjustments
```

with:

```text
reason
quantity
movementType
```

This preserves the inventory ledger.

---

# 25. Stock Adjustment API

Example:

```text
POST /api/v1/inventory/adjustments
```

Request:

```json
{
  "productId": "product_id",
  "locationId": "location_id",
  "quantity": 5,
  "reason": "Physical stock count correction"
}
```

Backend:

```text
Validate permission
      ↓
Validate product
      ↓
Validate location
      ↓
Create adjustment
      ↓
Create inventory movement
      ↓
Update stock
      ↓
Audit
```

The operation should be transactional.

---

# 26. Stock Transfer API

```text
POST /api/v1/inventory/transfers
GET  /api/v1/inventory/transfers
GET  /api/v1/inventory/transfers/:id
```

Example:

```text
Warehouse
   ↓
Transfer
   ↓
Branch
```

The transfer should produce corresponding movement records:

```text
TRANSFER_OUT
TRANSFER_IN
```

---

# 27. Supplier APIs

```text
GET    /api/v1/suppliers
POST   /api/v1/suppliers
GET    /api/v1/suppliers/:id
PATCH  /api/v1/suppliers/:id
DELETE /api/v1/suppliers/:id
```

---

# 28. Purchase APIs

```text
GET  /api/v1/purchases
POST /api/v1/purchases
GET  /api/v1/purchases/:id
PATCH /api/v1/purchases/:id
POST /api/v1/purchases/:id/receive
POST /api/v1/purchases/:id/cancel
```

Receiving stock must use the purchase receiving workflow.

---

# 29. Purchase Receiving

The endpoint:

```text
POST /api/v1/purchases/:id/receive
```

should execute:

```text
Validate purchase
      ↓
Validate receiving quantities
      ↓
Create inventory movements
      ↓
Update stock
      ↓
Update purchase status
      ↓
Audit
```

This must be transactional.

---

# 30. Customer APIs

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

---

# 31. POS APIs

POS requires fast and transaction-safe APIs.

Possible endpoints:

```text
POST /api/v1/pos/cart
GET  /api/v1/pos/products/search
POST /api/v1/pos/sales
POST /api/v1/pos/sales/:id/pay
GET  /api/v1/pos/sales/:id
```

However, cart state may remain primarily client-side until checkout.

The most important server operation is the sale transaction.

---

# 32. Sale Creation

Primary endpoint:

```text
POST /api/v1/sales
```

Example request:

```json
{
  "customerId": "customer_id",
  "locationId": "location_id",
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
finalTotal
stockAvailability
taxAmount
discountAmount
unitPrice
```

where those values are determined by server-side business rules.

---

# 33. Critical Sale Transaction

The sale endpoint should execute conceptually:

```text
Request
  ↓
Authentication
  ↓
Tenant resolution
  ↓
Permission check
  ↓
Validate input
  ↓
Load products
  ↓
Resolve prices
  ↓
Validate stock
  ↓
Calculate totals
  ↓
BEGIN TRANSACTION
  ↓
Create Sale
  ↓
Create Sale Items
  ↓
Create Payments
  ↓
Create Inventory Movements
  ↓
Update Stock
  ↓
Create Invoice
  ↓
Create Audit Log
  ↓
COMMIT
```

If any critical step fails:

```text
ROLLBACK
```

---

# 34. Idempotency

Critical APIs should support idempotency where duplicate requests could create financial or inventory problems.

Especially:

```text
POST /sales
POST /payments
POST /refunds
POST /purchases/:id/receive
```

Example header:

```text
Idempotency-Key: unique-client-operation-id
```

The server should prevent accidental duplicate processing.

This is particularly important for:

* Network retries
* Double clicks
* Mobile connections
* Payment callbacks
* Client reconnects

---

# 35. Payment APIs

Core endpoints:

```text
POST /api/v1/payments
GET  /api/v1/payments/:id
POST /api/v1/payments/:id/refund
```

External payment integrations may also require webhook endpoints:

```text
POST /api/v1/payments/webhooks/:provider
```

Webhook handling must verify provider signatures before processing events.

---

# 36. Invoice APIs

```text
GET  /api/v1/invoices
GET  /api/v1/invoices/:id
GET  /api/v1/invoices/:id/pdf
```

Invoice creation should generally occur as part of the appropriate sale transaction.

The API should not allow arbitrary invoice manipulation after finalization.

---

# 37. Returns APIs

```text
POST /api/v1/sales/:id/returns
GET  /api/v1/returns
GET  /api/v1/returns/:id
POST /api/v1/returns/:id/refund
```

Return processing should:

```text
Validate original sale
      ↓
Validate returnable quantity
      ↓
Create return
      ↓
Create inventory movement
      ↓
Process refund if applicable
      ↓
Audit
```

---

# 38. Analytics APIs

Analytics endpoints are read-focused.

Examples:

```text
GET /api/v1/analytics/dashboard
GET /api/v1/analytics/sales
GET /api/v1/analytics/products
GET /api/v1/analytics/inventory
GET /api/v1/analytics/customers
GET /api/v1/analytics/profit
```

Analytics APIs should not directly mutate transactional records.

---

# 39. Reporting APIs

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
  "jobId": "job_id",
  "status": "QUEUED"
}
```

Then:

```text
GET /api/v1/reports/jobs/:jobId
```

---

# 40. AI APIs

AI endpoints should expose business intelligence rather than unrestricted AI operations.

Examples:

```text
GET  /api/v1/ai/insights
GET  /api/v1/ai/recommendations
GET  /api/v1/ai/forecasts
POST /api/v1/ai/analyze/sales
POST /api/v1/ai/analyze/inventory
```

AI processing may be asynchronous.

Example:

```text
POST /api/v1/ai/analyze/inventory
```

Response:

```json
{
  "jobId": "job_id",
  "status": "QUEUED"
}
```

Worker:

```text
BullMQ
   ↓
AI Service
   ↓
Analysis
   ↓
AIInsight
   ↓
PostgreSQL
```

---

# 41. AI Safety Boundary

AI should not directly bypass normal business services.

Incorrect:

```text
AI → PostgreSQL → Inventory Mutation
```

Preferred:

```text
AI
 ↓
Recommendation
 ↓
User / Business Rule
 ↓
Normal Service
 ↓
Validation
 ↓
Transaction
 ↓
Database
```

This keeps AI explainable and controlled.

---

# 42. Notification APIs

```text
GET   /api/v1/notifications
PATCH /api/v1/notifications/:id/read
PATCH /api/v1/notifications/read-all
```

Notification creation may happen through background workers.

Example:

```text
Low Stock
   ↓
BullMQ
   ↓
Notification Worker
   ↓
Notification
```

---

# 43. User APIs

```text
GET   /api/v1/users/me
PATCH /api/v1/users/me
GET   /api/v1/users
GET   /api/v1/users/:id
PATCH /api/v1/users/:id
PATCH /api/v1/users/:id/status
```

Administrative user operations require appropriate permissions.

---

# 44. Role and Permission APIs

```text
GET  /api/v1/roles
POST /api/v1/roles
PATCH /api/v1/roles/:id
GET  /api/v1/permissions
```

Role assignment:

```text
PATCH /api/v1/users/:id/role
```

Changing an owner's permissions should have additional safeguards.

---

# 45. Tenant APIs

Tenant management:

```text
GET   /api/v1/tenants/current
PATCH /api/v1/tenants/current
GET   /api/v1/tenants/current/settings
PATCH /api/v1/tenants/current/settings
```

Future multi-tenant account management may include:

```text
GET  /api/v1/tenants
POST /api/v1/tenants
POST /api/v1/tenants/:id/members
```

These should only be exposed when the account model requires them.

---

# 46. Industry Capability APIs

Industry-specific APIs should be namespaced where appropriate.

### Pharmacy

```text
GET  /api/v1/pharmacy/medicines
POST /api/v1/pharmacy/medicines
GET  /api/v1/pharmacy/batches
POST /api/v1/pharmacy/batches
GET  /api/v1/pharmacy/expiry
```

### Restaurant

```text
GET  /api/v1/restaurant/menu
POST /api/v1/restaurant/menu
GET  /api/v1/restaurant/ingredients
POST /api/v1/restaurant/recipes
GET  /api/v1/restaurant/tables
```

### Clothing

Clothing-specific variant information can primarily use:

```text
/api/v1/products/:id/variants
```

with clothing capability validation.

### Supermarket

Supermarket-specific workflows can primarily use shared:

```text
products
inventory
pos
sales
```

with barcode and bulk-unit capabilities enabled.

---

# 47. API Request Validation

Every write endpoint must validate incoming data.

Validation should happen before business logic.

Example:

```text
Request
 ↓
Schema Validation
 ↓
Service
```

Validation should cover:

```text
Required fields
Data types
String length
Enum values
Numeric ranges
Identifiers
Dates
Arrays
Nested objects
```

Validation must happen server-side even if the frontend already validates the same input.

---

# 48. Business Validation

Schema validation is not enough.

Example:

```text
quantity = 5
```

may be structurally valid.

But the business rule may be:

```text
available stock = 2
```

Therefore:

```text
Input Validation
      +
Business Validation
```

are both required.

---

# 49. Response Format

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

The exact response envelope should remain consistent throughout the API.

---

# 50. Error Response Format

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

---

# 51. Error Codes

Application error codes should be stable and machine-readable.

Examples:

```text
AUTH_INVALID_CREDENTIALS
AUTH_UNAUTHORIZED
AUTH_FORBIDDEN

TENANT_NOT_FOUND
TENANT_ACCESS_DENIED

PRODUCT_NOT_FOUND
PRODUCT_SKU_EXISTS

INSUFFICIENT_STOCK
INVALID_STOCK_ADJUSTMENT

SALE_NOT_FOUND
SALE_ALREADY_COMPLETED

PAYMENT_FAILED
PAYMENT_ALREADY_PROCESSED

VALIDATION_ERROR
RESOURCE_NOT_FOUND
CONFLICT
INTERNAL_ERROR
```

The frontend should rely on error codes rather than parsing human-readable messages.

---

# 52. HTTP Status Codes

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

The exact usage should remain consistent.

---

# 53. Pagination

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

---

# 54. Filtering

Examples:

```text
GET /api/v1/products?status=ACTIVE
GET /api/v1/sales?status=COMPLETED
GET /api/v1/inventory?lowStock=true
```

Filters must be validated and restricted to supported fields.

The API should never dynamically accept arbitrary database column names from clients.

---

# 55. Sorting

Example:

```text
GET /api/v1/products?sortBy=createdAt&sortOrder=desc
```

Supported sort fields should be explicitly whitelisted.

This prevents unsafe dynamic query construction.

---

# 56. Date Filtering

Example:

```text
GET /api/v1/sales?from=2026-09-01&to=2026-09-30
```

The backend must validate:

```text
date format
timezone interpretation
from <= to
maximum allowed date range
```

where appropriate.

---

# 57. API Security

The API must implement:

```text
Authentication
Authorization
Tenant isolation
Input validation
Rate limiting
Secure headers
CORS restrictions
Request size limits
SQL injection protection
Sensitive data protection
Audit logging
```

Prisma parameterization helps prevent SQL injection, but unsafe raw queries must still be handled carefully.

---

# 58. Rate Limiting

Redis may be used for distributed rate limiting.

Different endpoints may have different limits.

Examples:

```text
Login
Password reset
AI endpoints
Search
Public APIs
```

AI endpoints may have stricter limits because of provider cost.

Rate limiting should return:

```text
429 Too Many Requests
```

when limits are exceeded.

---

# 59. CORS

The production API should only allow approved application origins.

Development:

```text
http://localhost:3000
```

Production:

```text
https://buzzsynx.com
https://app.buzzsynx.com
```

Exact domains will be finalized during deployment.

Wildcard CORS should not be used for authenticated production APIs unless there is a documented reason.

---

# 60. API Logging

Logs should contain useful operational information:

```text
requestId
method
path
status
duration
userId where appropriate
tenantId where appropriate
errorCode
```

Do not log:

```text
passwords
tokens
payment secrets
API keys
sensitive personal information
```

---

# 61. Audit Logging

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
Settings Changed
```

Audit logging should happen inside or immediately around the appropriate business transaction depending on the event.

---

# 62. Database Transactions in APIs

Controllers should not independently manipulate multiple business entities.

Example:

```text
SaleController
```

should call:

```text
SaleService.createSale()
```

The service coordinates:

```text
Sale
SaleItem
Payment
Inventory
Invoice
Audit
```

inside the appropriate transaction boundary.

---

# 63. Service Layer

Business logic belongs in services.

Example:

```javascript
async function createSale(context, input) {
  // business workflow
}
```

The service should receive trusted context:

```text
userId
tenantId
role
capabilities
```

rather than relying on arbitrary request-body values.

---

# 64. Controller Layer

Controllers should remain thin.

Controller responsibilities:

```text
Read request
 ↓
Pass validated input
 ↓
Call service
 ↓
Format response
```

Controllers should not contain large business workflows.

Avoid:

```javascript
// 300 lines of sale logic inside controller
```

Prefer:

```text
Controller
   ↓
SaleService
   ↓
InventoryService
PaymentService
InvoiceService
```

---

# 65. Repository Layer

Repositories/data-access functions should encapsulate Prisma queries where useful.

Example:

```text
ProductRepository
SaleRepository
InventoryRepository
CustomerRepository
```

Repositories should not decide whether a user is allowed to perform an operation.

Authorization belongs at the application/service boundary.

---

# 66. API and Background Jobs

Not every operation should remain synchronous.

Use BullMQ for long-running operations such as:

```text
AI analysis
Large reports
Email delivery
Notification processing
Data aggregation
Scheduled tasks
Bulk imports
Export generation
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

---

# 67. API Timeout Strategy

API endpoints should not wait indefinitely for external services.

Especially:

```text
AI providers
Payment providers
Email providers
Storage providers
External APIs
```

Use:

```text
Timeout
Retry where safe
Circuit-breaker strategy where justified
Fallback
Async processing
```

Do not blindly retry financial transactions.

---

# 68. Webhooks

External webhook endpoints must:

1. Verify authenticity.
2. Validate payload.
3. Check event type.
4. Check idempotency.
5. Process safely.
6. Record the event.
7. Return an appropriate response.

Example:

```text
Payment Provider
      ↓
Webhook
      ↓
Verify Signature
      ↓
Check Idempotency
      ↓
Process Payment
      ↓
Update Database
```

Webhook processing should be designed for duplicate delivery.

---

# 69. API Documentation

Buzzsynx should eventually expose OpenAPI documentation.

Example:

```text
/api/docs
```

Documentation should include:

```text
Endpoint
Method
Authentication
Permissions
Parameters
Request Body
Response
Errors
Examples
```

OpenAPI can later be used to support frontend/mobile integration and automated testing.

---

# 70. API Testing

API tests should cover:

### Authentication

```text
Register
Login
Logout
Invalid credentials
Session expiry
```

### Tenant isolation

```text
Tenant A → Tenant A data
Tenant A → cannot access Tenant B data
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
Create sale
Stock validation
Payment
Invoice
Rollback
Duplicate request
```

### Payments

```text
Success
Failure
Duplicate
Refund
Webhook
```

### AI

```text
Queue job
Worker processing
Insight creation
Failure handling
```

---

# 71. API Performance

Performance should be measured rather than assumed.

Important metrics:

```text
Average latency
P95 latency
P99 latency
Requests per second
Error rate
Database query duration
Redis latency
External provider latency
Queue processing time
```

Optimization sequence:

```text
Correctness
 ↓
Measure
 ↓
Optimize database queries
 ↓
Add indexes
 ↓
Cache
 ↓
Background processing
 ↓
Scale infrastructure
```

---

# 72. API Caching

Safe read-heavy endpoints may use Redis caching.

Examples:

```text
Product lookup
Business settings
Dashboard summaries
Static configuration
```

Avoid caching highly volatile transactional state without a clear invalidation strategy.

For example, POS stock availability requires careful consistency.

PostgreSQL remains authoritative.

---

# 73. API Compatibility

API contracts should be treated as public interfaces.

Changes should avoid unexpectedly breaking:

```text
Frontend
Mobile clients
Third-party integrations
Webhooks
Automations
```

Breaking changes require versioning or a controlled migration strategy.

---

# 74. API Folder Structure

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
│   ├── products/
│   │   ├── product.routes.js
│   │   ├── product.controller.js
│   │   ├── product.service.js
│   │   ├── product.repository.js
│   │   └── product.validation.js
│   │
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
│   ├── validation.js
│   ├── rate-limit.js
│   ├── error-handler.js
│   └── request-id.js
│
├── jobs/
└── queues/
```

---

# 75. API Development Order

The backend API should be implemented in this order.

## Phase 1 — Platform

```text
Health
Authentication
Tenant resolution
Users
Roles
Permissions
```

## Phase 2 — Product

```text
Categories
Brands
Products
Variants
```

## Phase 3 — Inventory

```text
Locations
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

## Phase 5 — POS

```text
Customers
Sales
Payments
Invoices
```

## Phase 6 — Returns

```text
Returns
Refunds
```

## Phase 7 — Industry Capabilities

```text
Pharmacy
Supermarket
Clothing
Restaurant
```

## Phase 8 — Intelligence

```text
Analytics
Reports
AI
Notifications
```

## Phase 9 — Production Hardening

```text
Rate limiting
Caching
Observability
Security
Performance
API documentation
```

---

# 76. API Definition of Done

An API module is considered complete when:

* [ ] Route is defined.
* [ ] Authentication requirement is defined.
* [ ] Tenant scope is defined.
* [ ] Permission is defined.
* [ ] Capability requirement is defined where applicable.
* [ ] Request schema is validated.
* [ ] Business validation is implemented.
* [ ] Controller is implemented.
* [ ] Service is implemented.
* [ ] Repository/data access is implemented.
* [ ] Transaction boundary is defined where required.
* [ ] Error codes are defined.
* [ ] Response format is consistent.
* [ ] Audit logging is implemented where required.
* [ ] Idempotency is implemented where required.
* [ ] Tests are implemented.
* [ ] API documentation is updated.

---

# 77. API Design Principles

The following principles are mandatory:

### 1. Server is authoritative

Never trust client-calculated financial or inventory values.

### 2. Tenant isolation is non-negotiable

Every tenant request must execute inside a verified tenant context.

### 3. Authentication and authorization are separate

Knowing who the user is does not determine what they can do.

### 4. Controllers stay thin

Business logic belongs in services.

### 5. Critical operations are transactional

Sales, payments, inventory, receiving, and refunds require careful transaction boundaries.

### 6. APIs are idempotent where necessary

Financial and inventory operations must handle retries safely.

### 7. Errors are structured

Clients should receive stable error codes.

### 8. Redis is an optimization layer

It does not replace PostgreSQL.

### 9. AI remains inside controlled workflows

AI cannot bypass normal business rules.

### 10. API contracts are documented

Every production API should have a clear contract.

---

# 78. Final API Architecture

```text
                         CLIENT
                           │
                    Next.js / Mobile
                           │
                           ▼
                    REST API /v1
                           │
                    ┌──────┴──────┐
                    │ Middleware  │
                    │             │
                    │ Request ID  │
                    │ Auth        │
                    │ Tenant      │
                    │ RBAC        │
                    │ Validation  │
                    │ Rate Limit  │
                    └──────┬──────┘
                           │
                           ▼
                     Controllers
                           │
                           ▼
                       Services
                           │
             ┌─────────────┼─────────────┐
             │             │             │
        PostgreSQL       Redis        BullMQ
             │                           │
             │                         Workers
             │                           │
             │                    ┌──────┼──────┐
             │                    │      │      │
             │                   AI   Reports Notifications
             │
             ▼
        Business Data
```

The API architecture therefore follows:

> **Request → Authenticate → Resolve Tenant → Authorize → Validate → Execute Business Service → Persist Transaction → Respond**

---
