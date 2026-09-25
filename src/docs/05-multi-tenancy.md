# Buzzsynx — Multi-Tenancy Architecture

## 1. Purpose

This document defines the multi-tenancy architecture for Buzzsynx.

Buzzsynx is designed as a single SaaS platform that can serve multiple independent businesses while keeping their data, users, configurations, permissions, and business operations isolated from one another.

Example:

```text
                    BUZZSYNX
                       │
       ┌───────────────┼────────────────┐
       │               │                │
    Tenant A         Tenant B         Tenant C
    Pharmacy       Supermarket       Clothing
       │               │                │
    Users            Users            Users
    Products         Products         Products
    Inventory        Inventory        Inventory
    Sales            Sales            Sales
```

The architecture must provide:

* Strong tenant isolation
* Tenant-aware authentication
* Tenant-aware authorization
* Tenant-scoped database queries
* Tenant-specific configuration
* Tenant-specific industry capabilities
* Tenant-aware caching
* Tenant-aware background jobs
* Tenant-aware analytics
* Tenant-aware audit logs
* A path toward future tenant scaling

---

# 2. Multi-Tenancy Definition

A **tenant** represents an independent business or organization using Buzzsynx.

Examples:

```text
Tenant A
Business: Sri Lakshmi Pharmacy
Industry: Pharmacy

Tenant B
Business: ABC Supermarket
Industry: Supermarket

Tenant C
Business: Fashion Hub
Industry: Clothing

Tenant D
Business: Spice Garden
Industry: Restaurant
```

Each tenant operates inside the same Buzzsynx application but owns its own business data.

---

# 3. Core Multi-Tenant Principle

The fundamental rule is:

> **A tenant can access only the data and capabilities that belong to that tenant and that the authenticated user is authorized to access.**

Conceptually:

```text
Authenticated User
       ↓
Tenant Membership
       ↓
Current Tenant
       ↓
Role / Permissions
       ↓
Industry Capabilities
       ↓
Business Data
```

No business operation should bypass this chain.

---

# 4. Initial Multi-Tenant Strategy

Buzzsynx will initially use:

```text
Shared Application
       +
Shared PostgreSQL Database
       +
Shared Database Schema
       +
tenantId Isolation
```

Conceptually:

```text
PostgreSQL
│
├── Tenant A
│   ├── Products
│   ├── Inventory
│   ├── Sales
│   └── Customers
│
├── Tenant B
│   ├── Products
│   ├── Inventory
│   ├── Sales
│   └── Customers
│
└── Tenant C
    ├── Products
    ├── Inventory
    ├── Sales
    └── Customers
```

This is appropriate for the initial Buzzsynx architecture because it keeps infrastructure simpler while allowing the platform to serve many businesses.

---

# 5. Why Shared Schema?

The initial shared-schema approach provides:

* Lower infrastructure complexity
* Easier development
* Easier migrations
* Lower operating cost
* Centralized reporting infrastructure
* Straightforward Prisma integration
* Easier local development
* Easier automated testing

It also allows Buzzsynx to focus on correct application-level tenant isolation before introducing more complex infrastructure.

---

# 6. Tenant Identity

Every tenant must have a stable unique identifier.

Example:

```text
tenantId:
tenant_01JXYZ...
```

The tenant ID is the internal identity used to associate business records with the tenant.

Human-readable identifiers can also exist.

Example:

```text
Tenant ID:
tenant_abc123

Slug:
fashion-hub

Business Name:
Fashion Hub
```

The internal ID must be used for database relationships.

---

# 7. Tenant Data Ownership

Tenant-owned entities include:

```text
Users / Memberships
Products
Categories
Brands
Variants
Suppliers
Customers
Stock
Inventory Movements
Purchases
Sales
Payments
Invoices
Returns
Reports
AI Insights
Notifications
Audit Logs
```

These entities must have a clear relationship to the tenant.

Example:

```text
Product
-------
id
tenantId
name
sku
...
```

---

# 8. Tenant-Owned vs System-Owned Data

Not every record needs a tenant ID.

### Tenant-owned data

Examples:

```text
Products
Customers
Sales
Inventory
Suppliers
```

These belong to a specific business.

### System-owned data

Examples:

```text
Global Permission Definitions
Supported Industry Types
System Feature Definitions
Platform Configuration
```

These belong to Buzzsynx itself.

Conceptually:

```text
System Data
    │
    ├── Industry Definitions
    ├── Permission Definitions
    └── Feature Definitions

Tenant Data
    │
    ├── Products
    ├── Sales
    ├── Inventory
    └── Customers
```

The application must explicitly distinguish these two categories.

---

# 9. User and Tenant Relationship

A user account and a tenant are separate concepts.

```text
User
  ↓
Tenant Membership
  ↓
Tenant
```

This allows future support for a user belonging to multiple businesses.

Example:

```text
Akash
 │
 ├── Tenant A → Business Owner
 │
 └── Tenant B → Consultant
```

The initial product may support one primary tenant per user during onboarding, but the data model should not make multi-tenant membership impossible.

---

# 10. Tenant Membership

The membership model connects a user to a tenant.

Conceptual structure:

```text
TenantUser
----------
id
tenantId
userId
roleId
status
createdAt
updatedAt
```

Relationship:

```text
User
 │
 ├── TenantUser → Tenant A
 │
 └── TenantUser → Tenant B
```

This is preferable to placing a single permanent `tenantId` directly on the User model if future multi-tenant accounts are expected.

---

# 11. Tenant Context

Every authenticated business request should have a resolved tenant context.

Conceptually:

```javascript
context = {
  userId,
  tenantId,
  role,
  permissions,
  capabilities
}
```

The tenant context should be created by trusted backend logic.

It must not simply be copied from the request body.

---

# 12. Tenant Resolution

The API should resolve the current tenant after authentication.

Conceptual flow:

```text
Request
   ↓
Authentication
   ↓
Identify User
   ↓
Load Tenant Membership
   ↓
Resolve Current Tenant
   ↓
Create Tenant Context
   ↓
RBAC
   ↓
Business Operation
```

The exact tenant-selection mechanism may evolve depending on whether users can belong to multiple tenants.

---

# 13. Tenant Selection

If a user belongs to multiple tenants, the client may request a tenant context.

For example:

```text
Tenant A
Tenant B
Tenant C
```

However, the backend must verify:

```text
Does this user actually belong to the requested tenant?
```

before activating that tenant context.

The client can request a tenant.

The server decides whether that tenant is valid.

---

# 14. Never Trust Client tenantId

This is a mandatory security rule.

Incorrect:

```javascript
const { tenantId } = req.body;

const products = await prisma.product.findMany({
  where: { tenantId }
});
```

This allows a malicious client to attempt:

```text
tenantId = another_business
```

Instead:

```javascript
const tenantId = req.context.tenantId;

const products = await prisma.product.findMany({
  where: { tenantId }
});
```

The trusted tenant context must come from authenticated membership.

---

# 15. Tenant Isolation at the API Layer

Every tenant-scoped endpoint must operate within the current tenant.

Example:

```text
GET /api/v1/products
```

Internally:

```text
currentTenantId
      ↓
Product query
      ↓
WHERE tenantId = currentTenantId
```

Not:

```text
GET /products
      ↓
Return every product
```

---

# 16. Tenant Isolation at the Service Layer

Services should receive tenant context.

Example:

```javascript
await productService.getProducts({
  tenantId,
  filters
});
```

The service must not assume that the caller has already scoped the data correctly.

For sensitive operations, tenant ownership should be verified again where appropriate.

---

# 17. Tenant Isolation at the Repository Layer

Repositories should make tenant-aware queries easy and consistent.

Example:

```javascript
productRepository.findMany({
  tenantId,
  filters
});
```

Conceptually:

```javascript
where: {
  tenantId,
  ...filters
}
```

The repository should avoid exposing unsafe generic query methods that make accidental cross-tenant access easy.

---

# 18. Defense in Depth

Tenant isolation should not depend on a single layer.

Buzzsynx should use multiple protection layers:

```text
Authentication
      ↓
Tenant Membership
      ↓
Tenant Context
      ↓
RBAC
      ↓
Service Validation
      ↓
Tenant-Scoped Repository
      ↓
Database Constraints
      ↓
Audit Logging
```

The goal is to make cross-tenant access difficult even if one application layer contains a mistake.

---

# 19. Database Relationships

Tenant-owned records should have explicit relationships.

Example:

```text
Tenant
  │
  └── Product
        │
        └── ProductVariant
```

Both should have clear tenant ownership where required.

For example:

```text
Product
tenantId = Tenant A

ProductVariant
tenantId = Tenant A
productId = Product A
```

The application must ensure that related records belong to the same tenant.

---

# 20. Cross-Tenant Relationship Protection

A dangerous scenario:

```text
Tenant A Product
        +
Tenant B Category
```

The API must prevent relationships from crossing tenant boundaries.

Before assigning a category:

```text
Product tenant = Tenant A
Category tenant = Tenant B
```

must fail.

Expected result:

```text
TENANT_RESOURCE_MISMATCH
```

This rule applies to:

* Product → Category
* Product → Brand
* Sale → Customer
* Sale → Product
* Purchase → Supplier
* Stock → Location
* Inventory Movement → Product
* Invoice → Sale
* Payment → Sale

and similar relationships.

---

# 21. Tenant-Scoped Unique Constraints

Many unique values should be unique **within a tenant**, not globally.

Example:

```text
Tenant A
SKU: PROD-001

Tenant B
SKU: PROD-001
```

This is valid.

Database constraint:

```text
UNIQUE(tenantId, sku)
```

rather than:

```text
UNIQUE(sku)
```

Typical tenant-scoped unique values:

```text
SKU
Barcode where business rules allow
Sale Number
Invoice Number
Category Slug
Supplier Reference
```

The exact constraint should be defined per entity.

---

# 22. Tenant-Specific Numbering

Business documents should support tenant-specific numbering.

Example:

```text
Tenant A
INV-000001
INV-000002

Tenant B
INV-000001
INV-000002
```

The same invoice number can exist in different tenants if their numbering namespaces are independent.

Database uniqueness:

```text
UNIQUE(tenantId, invoiceNumber)
```

---

# 23. Tenant Configuration

Each tenant should have its own business configuration.

Examples:

```text
Currency
Timezone
Tax Settings
Invoice Prefix
Low Stock Rules
POS Settings
Business Information
Notification Preferences
AI Preferences
```

Conceptually:

```text
Tenant
  ↓
BusinessSettings
```

No tenant should accidentally inherit mutable business settings from another tenant.

---

# 24. Tenant Industry Configuration

A tenant can enable an industry capability.

Example:

```text
Tenant A
Industry: PHARMACY

Tenant B
Industry: SUPERMARKET

Tenant C
Industry: CLOTHING

Tenant D
Industry: RESTAURANT
```

The tenant configuration determines which specialized workflows are available.

---

# 25. Industry Capability Model

Industry should not mean a completely different application.

Instead:

```text
Shared Core
     +
Industry Capability
     +
Tenant Configuration
```

Example:

```text
Pharmacy Tenant
      ↓
Core Product
      +
Medicine Details
      +
Batch / Expiry
```

Clothing:

```text
Clothing Tenant
      ↓
Core Product
      +
Size / Color / Variant
```

Restaurant:

```text
Restaurant Tenant
      ↓
Core Product
      +
Menu / Recipe / Ingredient
```

---

# 26. Tenant Feature Flags

Some capabilities may be controlled through feature configuration.

Example:

```text
TenantFeatures
--------------
inventory
pos
advancedAnalytics
aiInsights
multiLocation
restaurant
pharmacy
```

Feature flags should be checked server-side.

Frontend visibility is not authorization.

---

# 27. Tenant + RBAC

Tenant capability and user permission are separate.

Example:

```text
Tenant Capability:
PHARMACY

User Permission:
inventory.view
```

Both may be required for a particular operation.

Conceptually:

```text
Authenticated
     +
Tenant Membership
     +
Permission
     +
Capability
     ↓
Allowed
```

---

# 28. Tenant Lifecycle

A tenant has a lifecycle.

Example:

```text
PENDING
ACTIVE
SUSPENDED
ARCHIVED
```

Conceptual lifecycle:

```text
Signup
  ↓
Tenant Created
  ↓
Configuration
  ↓
ACTIVE
  ↓
Suspension if required
  ↓
SUSPENDED
  ↓
Reactivation / ARCHIVED
```

Suspended tenants should not be able to perform normal business operations.

---

# 29. Tenant Onboarding

Tenant onboarding should follow:

```text
Create Account
      ↓
Create Tenant
      ↓
Create Membership
      ↓
Assign Owner Role
      ↓
Select Industry
      ↓
Create Business Settings
      ↓
Enable Capabilities
      ↓
Create Initial Configuration
      ↓
Dashboard
```

This should be transactional wherever multiple database records are created together.

---

# 30. Tenant Deactivation

Deactivating a tenant should not immediately destroy its data.

Preferred initial behavior:

```text
ACTIVE
  ↓
SUSPENDED
```

Business data remains available according to platform policies.

Permanent deletion, if supported, must be handled through a controlled data lifecycle process.

---

# 31. Tenant Deletion

Tenant deletion is a high-risk operation.

Before permanent deletion:

```text
Verify authorization
      ↓
Confirm tenant identity
      ↓
Check retention requirements
      ↓
Create backup where appropriate
      ↓
Execute deletion workflow
      ↓
Audit
```

Production deletion should never be a casual single-button database operation.

---

# 32. Tenant-Aware Caching

Redis keys must include tenant identity whenever cached data is tenant-specific.

Incorrect:

```text
products:all
```

Correct:

```text
tenant:{tenantId}:products:all
```

Example:

```text
tenant:abc123:dashboard
tenant:abc123:products:search:paracetamol
tenant:xyz789:dashboard
```

This prevents cached data from leaking across tenants.

---

# 33. Cache Invalidation

Tenant-specific cache entries must be invalidated when the underlying tenant data changes.

Example:

```text
Product Updated
      ↓
Database Updated
      ↓
Invalidate:
tenant:{tenantId}:products:*
```

The exact strategy may use targeted keys or cache versioning depending on implementation.

---

# 34. Tenant-Aware Background Jobs

BullMQ jobs must carry enough context to identify the tenant.

Example:

```json
{
  "tenantId": "tenant_123",
  "jobType": "AI_INVENTORY_ANALYSIS"
}
```

Workers must never assume a global tenant context.

Worker flow:

```text
Job
 ↓
Resolve tenant
 ↓
Load tenant configuration
 ↓
Query tenant data
 ↓
Process
 ↓
Store tenant-scoped result
```

---

# 35. Tenant-Aware AI

AI jobs must be tenant-isolated.

Example:

```text
Tenant A Sales
      ↓
Tenant A AI Analysis
```

must never become:

```text
Tenant A Sales
      +
Tenant B Sales
      ↓
Combined AI Analysis
```

unless the platform explicitly performs an authorized system-level aggregate operation.

---

# 36. AI Data Privacy

Tenant business information may be sensitive.

AI processing must follow explicit data boundaries.

Before sending business data to an external AI provider, the system should determine:

```text
What data is required?
Why is it required?
Is it necessary to send personally identifiable information?
Can the data be aggregated?
What provider is being used?
What retention policies apply?
```

The AI layer should minimize unnecessary data transmission.

---

# 37. Tenant-Aware Analytics

Analytics queries must always respect tenant boundaries.

Example:

```text
Tenant A Dashboard
      ↓
Tenant A Sales
      ↓
Tenant A Analytics
```

A normal tenant dashboard must never query:

```text
All Sales
```

without tenant filtering.

---

# 38. System-Level Analytics

Platform administrators may eventually require aggregate analytics.

For example:

```text
Total Active Tenants
Total Sales Volume
System Usage
Feature Adoption
```

These are system-level operations.

They must be explicitly separated from tenant-level analytics.

Conceptually:

```text
Tenant Analytics
      ↓
Single Tenant

Platform Analytics
      ↓
Authorized System Scope
```

Platform analytics should never be exposed to normal tenant users.

---

# 39. Tenant-Aware Audit Logs

Audit records must identify the tenant for business operations.

Example:

```text
AuditLog
--------
tenantId
userId
action
entityType
entityId
metadata
createdAt
```

Example:

```text
Tenant A
User: user_123
Action: STOCK_ADJUSTED
Product: product_456
```

This allows tenant-specific audit history.

---

# 40. Tenant-Aware Notifications

Notifications should be scoped to:

```text
tenantId
userId
```

Example:

```text
Tenant A
   ↓
User A
   ↓
Low Stock Notification
```

The notification service must never accidentally deliver Tenant B events to Tenant A users.

---

# 41. Tenant-Aware File Storage

Uploaded files should also follow tenant isolation.

Examples:

```text
Product Images
Invoices
Reports
Business Logos
Documents
```

Storage keys should include tenant identity.

Example:

```text
tenants/{tenantId}/products/{productId}/image.webp
tenants/{tenantId}/invoices/{invoiceId}/invoice.pdf
tenants/{tenantId}/reports/{reportId}/report.xlsx
```

The storage layer must enforce authorization before generating download URLs.

---

# 42. Tenant-Aware API Routes

Normal APIs should not require tenant IDs in every URL.

Preferred:

```text
GET /api/v1/products
```

instead of:

```text
GET /api/v1/tenants/:tenantId/products
```

The authenticated tenant context determines ownership.

Tenant IDs in URLs may be appropriate for explicit platform administration or multi-tenant management APIs.

---

# 43. Tenant-Aware Resource Access

Even if a resource ID is known, ownership must be checked.

Example:

```text
GET /api/v1/products/product_ABC
```

The backend must effectively perform:

```text
Find product
WHERE id = product_ABC
AND tenantId = currentTenantId
```

Not:

```text
Find product
WHERE id = product_ABC
```

This is critical because IDs can be obtained or guessed independently of tenant membership.

---

# 44. Preventing IDOR

Buzzsynx must protect against **Insecure Direct Object Reference** vulnerabilities.

Example attack:

```text
Tenant A
User requests:

/api/v1/invoices/invoice_BELONGS_TO_TENANT_B
```

Expected behavior:

```text
404 Not Found
```

or an appropriate authorization-safe response.

The server must not return Tenant B's invoice.

---

# 45. Tenant Context in Transactions

Tenant context must remain consistent inside transactions.

Example:

```text
BEGIN
   ↓
Create Sale
   ↓
Create Sale Items
   ↓
Create Inventory Movements
   ↓
Create Invoice
   ↓
COMMIT
```

Every created record must use the same trusted:

```text
tenantId
```

A transaction must never mix records from different tenants.

---

# 46. Tenant Consistency Validation

For complex operations involving multiple resources:

```text
Sale
Customer
Product
Location
Payment
```

the service should verify that all belong to the current tenant.

Example:

```text
Sale Tenant = A
Customer Tenant = A
Product Tenant = A
Location Tenant = A
```

If any belongs to Tenant B:

```text
Reject transaction
```

---

# 47. Tenant Isolation Testing

Tenant isolation must have dedicated automated tests.

Example:

```text
Tenant A
Product A

Tenant B
Product B
```

Test:

```text
Authenticated as Tenant A
GET products
```

Expected:

```text
Product A
```

Not:

```text
Product A
Product B
```

---

# 48. Cross-Tenant Access Tests

Tests must explicitly attempt:

```text
Tenant A → Tenant B Product
Tenant A → Tenant B Sale
Tenant A → Tenant B Customer
Tenant A → Tenant B Invoice
Tenant A → Tenant B Inventory
Tenant A → Tenant B User
```

Every unauthorized attempt must fail safely.

---

# 49. Tenant Security Test Matrix

The test suite should include:

| Scenario                                   | Expected         |
| ------------------------------------------ | ---------------- |
| User accesses own tenant product           | Allowed          |
| User accesses another tenant product       | Denied           |
| User creates product for own tenant        | Allowed          |
| User submits another tenant ID             | Ignored/rejected |
| User references another tenant category    | Denied           |
| User accesses another tenant sale          | Denied           |
| User accesses another tenant invoice       | Denied           |
| User accesses another tenant customer      | Denied           |
| User accesses another tenant stock         | Denied           |
| User accesses another tenant notifications | Denied           |

---

# 50. Tenant Isolation and Prisma

Prisma queries should consistently include tenant conditions.

Example:

```javascript
const product = await prisma.product.findFirst({
  where: {
    id: productId,
    tenantId
  }
});
```

For updates:

```javascript
await prisma.product.updateMany({
  where: {
    id: productId,
    tenantId
  },
  data: updateData
});
```

This avoids updating a record simply because its ID exists.

The exact Prisma patterns should be standardized during backend implementation.

---

# 51. Avoiding Global Query Shortcuts

Avoid generic functions such as:

```javascript
getById(id)
```

for tenant-owned resources when the function can be called without tenant context.

Prefer:

```javascript
getById({
  id,
  tenantId
})
```

This makes tenant context part of the access contract.

---

# 52. Tenant Context Helper

The backend may expose a trusted request context:

```javascript
req.context = {
  userId,
  tenantId,
  role,
  permissions,
  capabilities
};
```

Services should use this trusted context.

The request body should not override it.

---

# 53. Tenant-Aware Logging

Application logs should include tenant context where appropriate.

Example:

```text
requestId=req_123
tenantId=tenant_abc
userId=user_456
action=SALE_CREATED
duration=84ms
```

However, logs must not expose sensitive business or authentication information unnecessarily.

---

# 54. Tenant-Aware Metrics

Metrics should be designed carefully.

Potential metrics:

```text
sales.count
inventory.adjustments
api.requests
ai.jobs
```

Tenant identifiers should not automatically become high-cardinality metric labels.

For example, attaching thousands of unique tenant IDs to every metric can create monitoring problems.

Tenant-specific investigation can instead use logs or traces.

---

# 55. Tenant-Aware Rate Limiting

Rate limits may operate at multiple levels:

```text
IP
User
Tenant
Endpoint
```

Example:

```text
Tenant AI limit
Tenant API limit
User login limit
```

This prevents one tenant from consuming disproportionate shared resources.

---

# 56. Resource Isolation

Although tenants initially share infrastructure, noisy-neighbor protection should be considered.

One tenant performing:

```text
Massive report generation
Large imports
Heavy AI analysis
```

should not degrade the entire platform.

Possible controls:

```text
Per-tenant rate limits
Queue concurrency limits
Job quotas
API throttling
Plan-based limits
```

---

# 57. Tenant Quotas

Future SaaS plans may define limits such as:

```text
Maximum Users
Maximum Products
Maximum Locations
Monthly Transactions
AI Usage
Storage
Reports
```

Example:

```text
Plan
 ↓
Tenant Limits
 ↓
Usage Tracking
 ↓
Request Validation
```

Quota enforcement must occur server-side.

---

# 58. Tenant Subscription

Subscription/billing is a platform capability and should be separated from ordinary business transactions.

Potential entities:

```text
Plan
Subscription
TenantUsage
BillingEvent
```

Relationship:

```text
Tenant
  ↓
Subscription
  ↓
Plan
```

Subscription status can influence feature availability but should not replace RBAC.

---

# 59. Tenant Data Export

A tenant may eventually need to export its business data.

Possible endpoint:

```text
POST /api/v1/settings/data-export
```

Flow:

```text
Request
 ↓
Authorization
 ↓
Create Export Job
 ↓
BullMQ
 ↓
Collect Tenant Data
 ↓
Generate Export
 ↓
Store File
 ↓
Notify User
```

The export worker must operate strictly within the tenant context.

---

# 60. Tenant Data Import

Imports should also be tenant-scoped.

Example:

```text
POST /api/v1/products/import
```

Flow:

```text
Upload
 ↓
Verify Tenant
 ↓
Queue Import
 ↓
Validate Data
 ↓
Create Tenant Records
 ↓
Report Errors
```

Imported records must never accept arbitrary tenant ownership from the uploaded file.

---

# 61. Tenant-Aware Search

Search queries must always include tenant context.

Example:

```text
Search:
"shirt"
```

must effectively mean:

```text
Search "shirt"
WHERE tenantId = currentTenantId
```

This applies to:

* Product search
* Customer search
* Supplier search
* Invoice search
* Sales search

---

# 62. Tenant-Aware Reports

Generated reports must contain only authorized tenant data.

Example:

```text
Monthly Sales Report
Tenant: Fashion Hub
```

The report generator should receive:

```text
tenantId
dateRange
filters
```

and generate data only within that tenant.

---

# 63. Tenant-Aware Scheduled Jobs

Scheduled jobs such as:

```text
Daily sales summary
Low-stock scan
Expiry scan
AI analysis
Notification processing
```

must iterate over active tenants safely.

Conceptual flow:

```text
Scheduler
   ↓
Find eligible tenants
   ↓
Queue tenant-specific jobs
   ↓
Worker
   ↓
Process one tenant context
```

A worker should not accidentally retain tenant context from a previous job.

---

# 64. Tenant Context Reset

Background workers are long-lived processes.

Therefore:

```text
Job A → Tenant A
Job B → Tenant B
```

must not accidentally reuse:

```text
Tenant A context
```

for Job B.

Tenant context should be explicitly initialized for every job.

---

# 65. Tenant Data in Client State

Frontend state may contain:

```text
currentTenant
user
permissions
capabilities
```

but this is only UI state.

The frontend must never be treated as the authority for tenant security.

```text
Frontend
   ↓
Request
   ↓
Backend
   ↓
Trusted Tenant Context
```

---

# 66. Tenant Switching

If multi-tenant accounts are supported:

```text
User
 ↓
Tenant List
 ↓
Select Tenant
 ↓
Server validates membership
 ↓
New tenant context
```

After switching:

* Cached tenant-specific data must be cleared or re-keyed.
* Client state must refresh.
* Active subscriptions/features must be reloaded.
* Permissions must be re-evaluated.

---

# 67. Tenant Context and Browser Storage

Sensitive tenant authorization information should not be trusted merely because it exists in:

```text
localStorage
sessionStorage
cookies
```

Browser state can improve UX.

Server-side membership determines actual authorization.

---

# 68. Tenant Isolation and APIs

The API contract should make tenant isolation implicit for ordinary tenant operations.

Preferred:

```text
GET /api/v1/sales
```

The current tenant is inferred from authentication context.

Avoid exposing:

```text
GET /api/v1/all-sales
```

to tenant users.

---

# 69. Platform Administration

Future Buzzsynx platform administrators may need cross-tenant capabilities.

These should be explicitly separated.

Example:

```text
/api/v1/platform/tenants
/api/v1/platform/tenants/:id
```

Platform APIs require:

```text
Platform Authentication
+
Platform Authorization
```

They must never reuse ordinary tenant permissions as a shortcut.

---

# 70. Platform Admin Safety

Platform-level access is highly privileged.

Actions such as:

```text
Suspend tenant
Inspect tenant configuration
Reset tenant access
Perform migration
Export tenant data
```

should have:

* Strong authorization
* Audit logs
* Explicit action boundaries
* Additional safeguards for destructive operations

---

# 71. Cross-Tenant Operations

Cross-tenant operations should be extremely rare.

Examples:

```text
Platform analytics
Platform billing
System monitoring
```

They must be explicitly designed as system-level operations.

Normal business services should never silently perform cross-tenant queries.

---

# 72. Tenant Isolation and Database Backups

Shared database backups contain data belonging to multiple tenants.

Therefore:

* Production backups must be secured.
* Access must be restricted.
* Backup credentials must be protected.
* Restore procedures must be tested.
* Tenant-level export/deletion procedures must account for shared storage.

A tenant deletion operation must not compromise other tenants.

---

# 73. Future Tenant Scaling

The initial strategy is:

```text
Shared DB
Shared Schema
tenantId
```

Future strategies may include:

```text
Shared DB
Separate Schema
```

or:

```text
Database per Tenant
```

for high-value enterprise customers.

Possible evolution:

```text
                Buzzsynx
                   │
        ┌──────────┼──────────┐
        │          │          │
    Standard    Growth    Enterprise
        │          │          │
 Shared Schema  Shared DB   Dedicated DB
 tenantId       optimized   or isolated
```

This is a future capability, not a requirement for version one.

---

# 74. Tenant Data Migration

If a tenant moves from shared storage to dedicated storage, the application should support a controlled migration process.

Conceptual flow:

```text
Shared Tenant Data
       ↓
Validate
       ↓
Export
       ↓
Transform
       ↓
Import
       ↓
Verify
       ↓
Switch Routing
       ↓
Monitor
```

The application architecture should keep tenant identity independent of physical database location.

---

# 75. Tenant Routing Abstraction

Future architecture may introduce:

```text
Tenant
 ↓
Storage Location
 ↓
Database Connection
```

For example:

```text
Tenant A → Shared DB
Tenant B → Shared DB
Tenant C → Dedicated DB
```

The business services should ideally not need to know the physical storage strategy.

---

# 76. Multi-Tenant Security Checklist

Before production:

* [ ] Every tenant-owned model has tenant ownership.
* [ ] Tenant membership is verified.
* [ ] Tenant context is created server-side.
* [ ] Client tenant IDs are never trusted.
* [ ] All tenant queries are scoped.
* [ ] Resource ownership is verified.
* [ ] Cross-tenant relationships are blocked.
* [ ] Tenant-scoped unique constraints are defined.
* [ ] Redis keys include tenant identity.
* [ ] Background jobs include tenant identity.
* [ ] Reports are tenant-scoped.
* [ ] AI processing is tenant-scoped.
* [ ] Notifications are tenant-scoped.
* [ ] File storage is tenant-scoped.
* [ ] Audit logs include tenant identity.
* [ ] Tenant isolation tests exist.
* [ ] IDOR tests exist.
* [ ] Platform APIs are separated from tenant APIs.
* [ ] Tenant deletion is controlled.
* [ ] Backup access is restricted.

---

# 77. Multi-Tenant Testing Strategy

The test suite should use at least two tenants:

```text
Tenant A
Tenant B
```

and preferably multiple business types:

```text
Pharmacy
Supermarket
Clothing
Restaurant
```

Tests should verify:

```text
Tenant A → sees A
Tenant B → sees B

Tenant A → cannot access B
Tenant B → cannot access A
```

This should be tested across every major module.

---

# 78. Definition of Done

The multi-tenancy architecture is considered implementation-ready when:

* [ ] Tenant entity is defined.
* [ ] Tenant membership is defined.
* [ ] Tenant lifecycle is defined.
* [ ] Tenant context strategy is defined.
* [ ] Tenant resolution is defined.
* [ ] Tenant-aware RBAC is defined.
* [ ] Tenant-aware capability checks are defined.
* [ ] Tenant database relationships are defined.
* [ ] Tenant-scoped unique constraints are defined.
* [ ] Tenant-aware caching is defined.
* [ ] Tenant-aware queues are defined.
* [ ] Tenant-aware AI processing is defined.
* [ ] Tenant-aware file storage is defined.
* [ ] Tenant-aware analytics are defined.
* [ ] Cross-tenant protection is tested.
* [ ] IDOR protection is tested.
* [ ] Tenant switching behavior is defined for future multi-tenant users.
* [ ] Platform administration boundaries are defined.
* [ ] Future tenant storage migration strategy is documented.

---

# 79. Core Multi-Tenancy Rules

The following rules are mandatory for Buzzsynx:

### Rule 1

**Tenant identity must come from trusted backend context.**

### Rule 2

**Never trust `tenantId` from the request body, query string, or frontend state for authorization.**

### Rule 3

**Every tenant-owned query must be tenant-scoped.**

### Rule 4

**Every resource lookup must verify tenant ownership.**

### Rule 5

**Related resources must belong to the same tenant.**

### Rule 6

**Redis keys must be tenant-aware.**

### Rule 7

**Background jobs must carry tenant context.**

### Rule 8

**AI processing must remain tenant-isolated.**

### Rule 9

**Tenant users must never receive platform-level permissions implicitly.**

### Rule 10

**Cross-tenant operations require explicit platform-level authorization.**

---

# 80. Final Multi-Tenant Architecture

```text
                         BUZZSYNX
                            │
                    Authentication
                            │
                            ▼
                     User Identity
                            │
                            ▼
                    Tenant Membership
                            │
                            ▼
                     Current Tenant
                            │
              ┌─────────────┴─────────────┐
              │                           │
             RBAC                    Capabilities
              │                           │
              └─────────────┬─────────────┘
                            │
                            ▼
                    Business Services
                            │
                  ┌─────────┼─────────┐
                  │         │         │
              PostgreSQL   Redis    BullMQ
                  │         │         │
                  │         │         └── Tenant-aware Jobs
                  │         │
                  │         └────────── Tenant-aware Cache
                  │
                  ▼
             Tenant Data
                  │
       ┌──────────┼──────────┐
       │          │          │
    Products   Inventory    Sales
       │          │          │
    Suppliers  Movements  Payments
       │          │          │
    Customers  Purchases  Invoices
```

The fundamental Buzzsynx multi-tenancy model is:

> **One application, one shared business engine, multiple independent tenants, strict tenant-scoped data access, configurable industry capabilities, and defense-in-depth isolation across the API, database, cache, queues, AI, files, analytics, and observability layers.**

---

# 81. Related Documents

This document connects directly with:

```text
01-architecture.md
02-system-workflow.md
03-database-design.md
04-api-design.md
06-industry-capabilities.md
07-security.md
08-ai-architecture.md
09-caching-and-queues.md
10-testing-strategy.md
11-devops.md
12-aws-infrastructure.md
13-observability.md
```
