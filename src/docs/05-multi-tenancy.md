# Buzzsynx — Multi-Tenancy Architecture

**Version:** v0.2
**Status:** Architecture-Aligned Multi-Tenancy Baseline
**Product:** Buzzsynx
**Architecture:** Multi-Tenant Modular Monolith
**Initial Industry:** Supermarket / Grocery
**Database:** PostgreSQL + Prisma
**Cache:** Redis
**Background Processing:** BullMQ

---

# 1. Purpose

This document defines the multi-tenancy architecture for Buzzsynx.

Buzzsynx is designed as a single SaaS platform capable of serving multiple independent businesses while keeping their:

* Data
* Users
* Memberships
* Stores / Branches
* Configurations
* Permissions
* Business operations
* Analytics
* AI processing
* Notifications
* Files

properly isolated.

The initial product implementation focuses on:

> **Supermarket / Grocery**

Other industries such as:

* Pharmacy
* Clothing
* Restaurant
* Clinic
* Hardware
* Other retail/business categories

are future industry capabilities and are not required to be implemented simultaneously.

The architecture must provide:

* Strong tenant isolation
* Tenant-aware authentication
* Tenant-aware authorization
* Store/branch-aware authorization
* Tenant-scoped database queries
* Store-scoped database queries where applicable
* Tenant-specific configuration
* Store-specific configuration where applicable
* Industry capability configuration
* Tenant-aware caching
* Tenant-aware background jobs
* Tenant-aware analytics
* Tenant-aware AI processing
* Tenant-aware audit logs
* Tenant-aware file storage
* A path toward future tenant scaling

---

# 2. Multi-Tenancy Definition

A **tenant** represents an independent business or organization using Buzzsynx.

Example:

```text
Buzzsynx
   │
   ├── Tenant A
   │     └── Supermarket Business
   │
   ├── Tenant B
   │     └── Pharmacy Business
   │
   └── Tenant C
         └── Clothing Business
```

Each tenant operates inside the same Buzzsynx application but owns its own business data.

A tenant may contain one or multiple stores/branches.

```text
Tenant
   │
   ├── Store A
   ├── Store B
   └── Store C
```

---

# 3. Core Multi-Tenant Principle

The fundamental rule is:

> **A tenant can access only the data and capabilities belonging to that tenant, and an authenticated user can access only the stores, resources, and operations permitted by their membership and permissions.**

Conceptually:

```text
Authenticated User
        ↓
Tenant Membership
        ↓
Current Tenant
        ↓
Store / Branch Scope
        ↓
Role / Permissions
        ↓
Industry Capabilities
        ↓
Business Data
```

No normal tenant business operation should bypass this chain.

---

# 4. Initial Multi-Tenant Strategy

Buzzsynx initially uses:

```text
Shared Application
        +
Shared PostgreSQL Database
        +
Shared Database Schema
        +
Tenant ID Isolation
        +
Store / Branch Scope
```

Conceptually:

```text
PostgreSQL

├── Tenant A
│    ├── Store A1
│    ├── Store A2
│    ├── Products
│    ├── Inventory
│    └── Sales
│
├── Tenant B
│    ├── Store B1
│    ├── Products
│    ├── Inventory
│    └── Sales
│
└── Tenant C
     ├── Store C1
     ├── Products
     ├── Inventory
     └── Sales
```

This approach keeps the initial infrastructure relatively simple while providing a clear path toward future scaling.

---

# 5. Why Shared Schema?

The initial shared-schema approach provides:

* Lower infrastructure complexity
* Easier development
* Easier migrations
* Lower operating cost
* Centralized platform management
* Straightforward Prisma integration
* Easier local development
* Easier automated testing
* Efficient resource utilization

The main responsibility is enforcing strong application-level tenant and store isolation.

---

# 6. Tenant Identity

Every tenant must have a stable unique identifier.

Example:

```text
tenant_01JXYZ...
```

The internal tenant ID is used for database relationships and authorization context.

A tenant may also have:

```text
tenantId:
tenant_01JXYZ...

slug:
fashion-hub

businessName:
Fashion Hub
```

The internal immutable ID should be used for database relationships.

Human-readable slugs may change according to business rules and should not replace internal IDs.

---

# 7. Store / Branch Identity

Every store or branch must have its own stable identifier.

Example:

```text
store_01JXYZ...
```

Relationship:

```text
Tenant
   │
   ├── Store A
   ├── Store B
   └── Store C
```

A store belongs to exactly one tenant.

Conceptually:

```text
Store

id
tenantId
name
code
status
createdAt
updatedAt
```

Store codes may be unique within a tenant:

```text
UNIQUE(tenantId, code)
```

---

# 8. Tenant-Owned vs Store-Owned Data

Not every tenant-owned entity is necessarily store-specific.

This distinction is important.

### Tenant-scoped data

Examples:

```text
Business Profile
Categories
Brands
Product Master
Suppliers
Customers
Tenant Settings
Roles
Memberships
Tenant Capabilities
```

### Store-scoped data

Examples:

```text
Stock
Inventory Movements
Sales
POS Sessions
Store-specific Pricing
Purchase Receiving
Store Cash Operations
Store-level Reports
```

Some entities may contain both:

```text
tenantId
storeId
```

when they belong to a tenant and operate within a specific store.

---

# 9. Tenant Data Ownership

Tenant-owned entities may include:

```text
Memberships
Products
Categories
Brands
Variants
Suppliers
Customers
Purchases
Sales
Payments
Invoices
Returns
Reports
AI Insights
Notifications
Audit Logs
Business Settings
```

Store-specific entities may additionally include:

```text
Stock
Inventory Movements
POS Transactions
Store Transfers
Store Settings
Store-level Pricing
```

Every entity must have a clearly defined ownership scope in the database design.

---

# 10. System-Owned Data

Not every record requires a tenant ID.

### System-owned data

Examples:

```text
Permission Definitions
Supported Industry Definitions
Capability Definitions
System Feature Definitions
Platform Configuration
Subscription Plans
```

These belong to Buzzsynx itself.

Conceptually:

```text
System Data
   │
   ├── Industry Definitions
   ├── Permission Definitions
   └── Capability Definitions

Tenant Data
   │
   ├── Products
   ├── Sales
   ├── Inventory
   └── Customers
```

The application must explicitly distinguish system-level and tenant-level data.

---

# 11. User and Tenant Relationship

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
User
 │
 ├── Tenant A → Owner
 │
 └── Tenant B → Consultant
```

The initial onboarding experience may create one primary tenant, but the data model should not permanently prevent multi-tenant membership.

---

# 12. Tenant Membership

The membership model connects a user to a tenant.

Conceptually:

```text
Membership

id
tenantId
userId
status
createdAt
updatedAt
```

Roles should be assigned through membership rather than being permanently attached to the global user.

Example:

```text
User
 │
 ├── Membership → Tenant A
 │                    └── Owner
 │
 └── Membership → Tenant B
                      └── Admin
```

---

# 13. Membership and Store Scope

Membership should support store access.

Conceptually:

```text
Membership
    │
    ├── Tenant
    ├── User
    ├── Roles
    └── Store Scope
```

A user may have:

```text
Owner
→ All stores

Manager
→ Store A + Store B

Cashier
→ Store A only

Store Staff
→ Store B only
```

The exact permissions are configurable according to the RBAC model.

This is essential for multi-branch businesses.

---

# 14. Tenant Context

Every authenticated tenant request should have a trusted context.

Conceptually:

```javascript
context = {
  userId,
  tenantId,
  membershipId,
  activeStoreId,
  roleIds,
  permissions,
  capabilities
}
```

Not every request requires an `activeStoreId`.

For example:

```text
Tenant Settings
```

may be tenant-scoped.

Whereas:

```text
Inventory
POS
Store Sales
```

normally require store scope.

---

# 15. Tenant Resolution

The API should resolve the current tenant after authentication.

Conceptual flow:

```text
Request
   ↓
Authentication
   ↓
Identify User
   ↓
Load Membership
   ↓
Resolve Current Tenant
   ↓
Resolve Store Scope where required
   ↓
Load Roles / Permissions
   ↓
Load Capabilities
   ↓
Business Operation
```

The exact tenant-selection mechanism may evolve as multi-tenant accounts are introduced.

---

# 16. Tenant Selection

If a user belongs to multiple tenants, the client may request a tenant context.

Example:

```text
Tenant A
Tenant B
Tenant C
```

The backend must verify:

```text
Does this user have an active membership in the requested tenant?
```

before activating that context.

The client can request a tenant.

The server decides whether that tenant is valid.

---

# 17. Store Selection

A multi-store user may also need to select an active store.

Example:

```text
Tenant: ABC Supermarket

Stores:
   Store A
   Store B
   Store C
```

The client may request:

```text
Store B
```

The backend must verify:

```text
User
  ↓
Membership
  ↓
Permission
  ↓
Store B Access
```

before using Store B as the active context.

A client-provided `storeId` is therefore an input to be **validated**, not an authority.

---

# 18. Never Trust Client tenantId

This is a mandatory security rule.

Incorrect:

```javascript
const { tenantId } = req.body;

const products = await prisma.product.findMany({
  where: { tenantId }
});
```

A malicious client could attempt:

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

The trusted tenant context must originate from authenticated membership.

---

# 19. Never Trust Client Store Scope Without Validation

A store ID may legitimately be selected by a multi-store user.

However:

```javascript
const { storeId } = req.body;
```

must never automatically mean:

```text
User is allowed to access this store.
```

The backend must verify:

```text
storeId belongs to current tenant
AND
user membership permits access
AND
user permission allows the operation
```

Only then can the store be used.

---

# 20. Tenant Isolation at API Layer

Every tenant-scoped endpoint must operate within the current tenant.

Example:

```text
GET /api/v1/products
```

Internally:

```text
currentTenantId
       ↓
Product Query
       ↓
WHERE tenantId = currentTenantId
```

For store-scoped resources:

```text
currentTenantId
       +
authorizedStoreId
       ↓
Store-scoped query
```

The API must never return unrestricted cross-tenant data.

---

# 21. Tenant Isolation at Service Layer

Services should receive trusted tenant context.

Example:

```javascript
await productService.getProducts({
  tenantId,
  filters
});
```

Store-scoped service:

```javascript
await inventoryService.getStock({
  tenantId,
  storeId,
  filters
});
```

Services must not assume that a controller has already performed every ownership check correctly.

Critical cross-resource operations should verify ownership and scope again where appropriate.

---

# 22. Tenant Isolation at Repository Layer

Repositories/data-access functions should make tenant-aware queries easy and consistent.

Example:

```javascript
productRepository.findMany({
  tenantId,
  filters
});
```

Store-scoped example:

```javascript
inventoryRepository.findMany({
  tenantId,
  storeId,
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

or:

```javascript
where: {
  tenantId,
  storeId,
  ...filters
}
```

Repositories should avoid unsafe generic access patterns that make cross-tenant queries easy.

---

# 23. Defense in Depth

Tenant isolation must not depend on one layer.

Buzzsynx should use multiple protection layers:

```text
Authentication
       ↓
Tenant Membership
       ↓
Tenant Context
       ↓
Store Scope
       ↓
RBAC / Permissions
       ↓
Capability Check
       ↓
Service Validation
       ↓
Tenant/Store-Scoped Data Access
       ↓
Database Constraints
       ↓
Audit Logging
```

The goal is to make accidental or malicious cross-tenant access difficult even if one application layer contains a defect.

---

# 24. Database Relationships

Tenant-owned records should have explicit relationships.

Example:

```text
Tenant
   │
   └── Product
          │
          └── ProductVariant
```

Where appropriate, both records may carry tenant ownership.

Example:

```text
Product

id
tenantId
name
sku
```

```text
ProductVariant

id
tenantId
productId
name
```

The application must ensure:

```text
Product.tenantId
=
ProductVariant.tenantId
```

---

# 25. Cross-Tenant Relationship Protection

A dangerous scenario:

```text
Tenant A Product
        +
Tenant B Category
```

This must fail.

Example:

```text
Product tenant = Tenant A

Category tenant = Tenant B
```

Expected:

```text
TENANT_RESOURCE_MISMATCH
```

The same rule applies to relationships such as:

```text
Product → Category
Product → Brand
Sale → Customer
Sale → Product
Purchase → Supplier
Inventory → Product
Inventory Movement → Product
Invoice → Sale
Payment → Sale
Return → Sale
```

---

# 26. Tenant + Store Relationship Protection

Store-owned resources must belong to the same tenant as the current operation.

Example:

```text
Sale
tenantId = Tenant A
storeId  = Store A1
```

Store A1 must belong to Tenant A.

Invalid:

```text
Sale
tenantId = Tenant A
storeId  = Tenant B's Store
```

This must be rejected.

The same rule applies to:

```text
Inventory
POS Sessions
Transfers
Store-level Purchases
Store-level Reports
Store Settings
```

---

# 27. Tenant-Scoped Unique Constraints

Many unique values should be unique **within a tenant**, not globally.

Example:

```text
Tenant A
SKU: PROD-001

Tenant B
SKU: PROD-001
```

This can be valid.

Database constraint:

```text
UNIQUE(tenantId, sku)
```

rather than:

```text
UNIQUE(sku)
```

Typical tenant-scoped unique values include:

```text
SKU
Barcode where appropriate
Category Slug
Supplier Reference
Invoice Number
Sale Number
Store Code
```

The exact constraint must be defined per entity.

---

# 28. Store-Scoped Unique Constraints

Some values should be unique within a store.

Example:

```text
Store A
POS-001

Store B
POS-001
```

This may be valid if POS numbering is store-specific.

Possible constraint:

```text
UNIQUE(tenantId, storeId, documentNumber)
```

The correct uniqueness scope must be determined for each business document.

---

# 29. Tenant-Specific Numbering

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

If invoice numbering is tenant-wide:

```text
UNIQUE(tenantId, invoiceNumber)
```

If numbering is store-specific:

```text
UNIQUE(tenantId, storeId, invoiceNumber)
```

The numbering policy belongs to tenant/business configuration.

---

# 30. Tenant Configuration

Each tenant should have its own business configuration.

Examples:

```text
Business Name
Business Contact Information
Currency
Timezone
Tax Settings
Invoice Prefix
Numbering Rules
Low Stock Rules
POS Settings
Notification Preferences
AI Preferences
```

Conceptually:

```text
Tenant
  ↓
BusinessSettings
```

Tenant configuration must never accidentally be shared as mutable state between businesses.

---

# 31. Store Configuration

Store-specific settings may include:

```text
Store Name
Store Address
Store Contact
POS Configuration
Invoice Prefix
Tax Registration Information
Receipt Settings
Operating Hours
```

Conceptually:

```text
Tenant
   ↓
Store
   ↓
StoreSettings
```

Store settings must remain within tenant boundaries.

---

# 32. Industry Configuration

A tenant may have a primary industry configuration.

Example:

```text
Tenant A
Primary Industry: SUPERMARKET
```

Future:

```text
Tenant B
Primary Industry: PHARMACY
```

However, the architecture should not depend on the industry value alone.

Industry-specific functionality should be represented through capabilities.

---

# 33. Industry Capability Model

Industry should not create a completely separate application.

Instead:

```text
Shared Core
     +
Industry Capabilities
     +
Tenant Configuration
```

Initial:

```text
Supermarket / Grocery
       ↓
Shared Core
       +
Retail Capabilities
```

Future pharmacy:

```text
Pharmacy
   ↓
Shared Core
   +
Medicine
   +
Batch / Expiry
   +
Pharmacy-specific Rules
```

Future clothing:

```text
Clothing
   ↓
Shared Core
   +
Size
   +
Color
   +
Variant-specific Rules
```

Future restaurant:

```text
Restaurant
   ↓
Shared Core
   +
Menu
   +
Recipe
   +
Ingredient
```

These future capabilities should be implemented only when their business requirements are validated.

---

# 34. Tenant Capabilities

Capabilities should determine which functionality is enabled for a tenant.

Example:

```text
TenantCapabilities

inventory
pos
advancedAnalytics
aiInsights
multiStore
barcode
expiryTracking
```

Future:

```text
pharmacy
restaurant
clothing
```

Capabilities should be stored/configured explicitly rather than inferred only from the tenant's industry label.

---

# 35. Capability Authorization

RBAC and capability configuration are separate.

Example:

```text
Tenant Capability:
inventory

User Permission:
inventory.adjust
```

Both may be required.

Conceptually:

```text
Authenticated
     +
Tenant Membership
     +
Store Scope
     +
Permission
     +
Capability
     ↓
Allowed Operation
```

The API must enforce capability checks server-side.

Frontend feature visibility is not authorization.

---

# 36. Tenant Lifecycle

A tenant may have:

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
Initial Configuration
   ↓
ACTIVE
   ↓
SUSPENDED if required
   ↓
Reactivated or ARCHIVED
```

Suspended tenants should not perform normal business operations.

Platform-level administrative operations may remain available where required.

---

# 37. Tenant Onboarding

The preferred onboarding workflow is:

```text
Create Account
      ↓
Create Tenant
      ↓
Create Initial Store
      ↓
Create Membership
      ↓
Assign Owner Role
      ↓
Select Primary Industry
      ↓
Initialize Capabilities
      ↓
Create Business Settings
      ↓
Create Store Settings
      ↓
Dashboard
```

Where the initial records are tightly coupled, they should be created transactionally.

The Owner should be able to begin using the platform without waiting for a manual platform approval step unless a specific product policy requires otherwise.

---

# 38. Initial Store Creation

Even if the business starts with one store, the tenant model should support multiple stores from the beginning.

Example:

```text
Tenant
   │
   └── Main Store
```

Later:

```text
Tenant
   ├── Main Store
   ├── Branch 2
   └── Branch 3
```

This avoids redesigning the tenant model when multi-branch customers arrive.

---

# 39. Tenant Deactivation

Deactivating a tenant should not immediately destroy its data.

Preferred:

```text
ACTIVE
   ↓
SUSPENDED
```

Business data remains preserved according to platform policies.

Permanent deletion should be treated as a separate controlled lifecycle process.

---

# 40. Tenant Deletion

Tenant deletion is a high-risk operation.

Before permanent deletion:

```text
Verify Authorization
       ↓
Confirm Tenant Identity
       ↓
Check Retention Requirements
       ↓
Backup / Export where appropriate
       ↓
Execute Controlled Deletion
       ↓
Verify Related Data
       ↓
Audit
```

Production deletion must never be an uncontrolled database operation.

Shared infrastructure makes this especially important because deletion must never affect another tenant.

---

# 41. Tenant-Aware Caching

Redis keys must include tenant identity whenever the cached data is tenant-specific.

Incorrect:

```text
products:all
```

Correct:

```text
tenant:{tenantId}:products:all
```

For store-specific data:

```text
tenant:{tenantId}:store:{storeId}:inventory
```

Examples:

```text
tenant:abc123:dashboard

tenant:abc123:products:search:milk

tenant:abc123:store:store001:inventory

tenant:xyz789:dashboard
```

This prevents accidental cache leakage across tenants and stores.

---

# 42. Cache Invalidation

Tenant-specific cache entries must be invalidated when underlying data changes.

Example:

```text
Product Updated
      ↓
Database Updated
      ↓
Invalidate Tenant Product Cache
```

Store-specific:

```text
Stock Updated
      ↓
Database Updated
      ↓
Invalidate:
tenant:{tenantId}:store:{storeId}:inventory
```

The exact strategy may use:

* Targeted invalidation
* Versioned keys
* Short TTLs
* Event-driven invalidation

depending on the use case.

---

# 43. Tenant-Aware Background Jobs

BullMQ jobs must carry sufficient context.

Example:

```json
{
  "tenantId": "tenant_123",
  "storeId": "store_456",
  "jobType": "AI_INVENTORY_ANALYSIS"
}
```

`storeId` is included only when the job is store-specific.

Worker flow:

```text
Job
 ↓
Resolve Tenant
 ↓
Resolve Store if applicable
 ↓
Load Tenant Configuration
 ↓
Validate Capability
 ↓
Query Authorized Data
 ↓
Process
 ↓
Store Tenant-Scoped Result
```

Workers must never assume a global tenant context.

---

# 44. Tenant Context Reset in Workers

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

Every job must explicitly establish its own tenant and store context.

No tenant context should be stored in mutable global worker state.

---

# 45. Tenant-Aware AI

AI processing must remain tenant-isolated.

Correct:

```text
Tenant A Sales
      ↓
Tenant A AI Analysis
```

Incorrect:

```text
Tenant A Sales
      +
Tenant B Sales
      ↓
Combined AI Analysis
```

unless an explicitly authorized platform-level aggregate operation exists.

AI services must receive only the business data required for the requested operation.

---

# 46. AI Data Privacy

Before sending tenant business information to an external AI provider, Buzzsynx should determine:

```text
What data is required?
Why is it required?
Can the data be aggregated?
Is personally identifiable information necessary?
Which provider receives it?
What retention policy applies?
```

The AI layer should minimize unnecessary data transmission.

AI provider credentials must never have direct unrestricted access to tenant databases.

---

# 47. Tenant-Aware Analytics

Analytics queries must always respect tenant boundaries.

Example:

```text
Tenant A Dashboard
       ↓
Tenant A Sales
       ↓
Tenant A Analytics
```

A tenant dashboard must never execute an unrestricted:

```text
SELECT * FROM Sales
```

without appropriate tenant/store filtering.

Store-level analytics must additionally respect store scope.

---

# 48. Platform-Level Analytics

Platform administrators may eventually require aggregate analytics.

Examples:

```text
Active Tenant Count
Platform Usage
Feature Adoption
System Health
Aggregate Transaction Volume
```

These are platform-level operations.

They must be explicitly separated from tenant analytics.

```text
Tenant Analytics
      ↓
Single Tenant Scope

Platform Analytics
      ↓
Authorized System Scope
```

Platform analytics must never be exposed to ordinary tenant users.

---

# 49. Tenant-Aware Audit Logs

Audit records must identify tenant context for business operations.

Example:

```text
AuditLog

id
tenantId
storeId
userId
action
entityType
entityId
metadata
createdAt
```

`storeId` should be nullable for tenant-level actions.

Example:

```text
Tenant A
Store A1
User: user_123
Action: STOCK_ADJUSTED
Product: product_456
```

Audit logs are separate from technical application logs.

---

# 50. Tenant-Aware Notifications

Notifications should be scoped to:

```text
tenantId
userId
```

and, where applicable:

```text
storeId
```

Example:

```text
Tenant A
   ↓
Store A1
   ↓
Low Stock Event
   ↓
Authorized User
```

The notification service must never deliver Tenant B events to Tenant A users.

---

# 51. Tenant-Aware File Storage

Uploaded files must follow tenant isolation.

Examples:

```text
Product Images
Invoices
Reports
Business Logos
Documents
Exports
```

Recommended storage structure:

```text
tenants/{tenantId}/products/{productId}/image.webp

tenants/{tenantId}/invoices/{invoiceId}/invoice.pdf

tenants/{tenantId}/reports/{reportId}/report.xlsx
```

Store-specific files may additionally include store identity:

```text
tenants/{tenantId}/stores/{storeId}/documents/{documentId}
```

Storage authorization must be checked before generating download URLs.

---

# 52. Tenant-Aware API Routes

Normal tenant APIs should not require tenant IDs in every URL.

Preferred:

```text
GET /api/v1/products
GET /api/v1/sales
GET /api/v1/inventory
```

The authenticated tenant context determines ownership.

Avoid:

```text
GET /api/v1/tenants/:tenantId/products
```

for ordinary tenant operations.

Tenant IDs in URLs are appropriate for explicitly authorized platform administration APIs.

---

# 53. Tenant-Aware Resource Access

Even when a resource ID is known, ownership must be verified.

Example:

```text
GET /api/v1/products/product_ABC
```

Backend behavior:

```text
Find Product

WHERE
    id = product_ABC
AND
    tenantId = currentTenantId
```

For store-scoped resources:

```text
WHERE
    id = resourceId
AND
    tenantId = currentTenantId
AND
    storeId = authorizedStoreId
```

The backend must not retrieve a resource by ID alone and then perform authorization afterward in an inconsistent manner.

---

# 54. Preventing IDOR

Buzzsynx must protect against **Insecure Direct Object Reference (IDOR)** vulnerabilities.

Example:

```text
Tenant A User

GET /api/v1/invoices/invoice_BELONGS_TO_TENANT_B
```

The server must not return Tenant B's invoice.

An appropriate safe response may be:

```text
404 Not Found
```

or another security-safe authorization response according to the endpoint policy.

The important requirement is:

> **Knowing a resource ID must never grant access to that resource.**

---

# 55. Tenant Context in Transactions

Tenant and store context must remain consistent inside transactions.

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
Create Payment Records
   ↓
Create Invoice
   ↓
Create Audit
   ↓
COMMIT
```

All tenant-owned records must belong to:

```text
tenantId = currentTenantId
```

and store-specific records must belong to the authorized:

```text
storeId
```

A transaction must never mix resources from different tenants.

---

# 56. Tenant Consistency Validation

Complex operations may involve:

```text
Sale
Customer
Product
Store
Payment
```

The service must verify their consistency.

Valid:

```text
Sale       → Tenant A
Customer   → Tenant A
Product    → Tenant A
Store      → Tenant A
Payment    → Tenant A
```

Invalid:

```text
Sale       → Tenant A
Customer   → Tenant B
```

The operation must fail.

For store-scoped operations:

```text
Sale Store = Store A
Product/Stock = Store A
```

or the appropriate shared product/store model must be respected.

---

# 57. Tenant Isolation Testing

Tenant isolation requires dedicated automated tests.

Example:

```text
Tenant A
   Product A

Tenant B
   Product B
```

Authenticated as Tenant A:

```text
GET /api/v1/products
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

# 58. Store Isolation Testing

Store-level isolation must also be tested.

Example:

```text
Tenant A

Store A1
   Stock A1

Store A2
   Stock A2
```

A Store A1-only user should receive:

```text
Stock A1
```

and not:

```text
Stock A2
```

unless their membership/permissions allow access to both stores.

---

# 59. Cross-Tenant Access Tests

Tests should explicitly attempt:

```text
Tenant A → Tenant B Product
Tenant A → Tenant B Sale
Tenant A → Tenant B Customer
Tenant A → Tenant B Invoice
Tenant A → Tenant B Inventory
Tenant A → Tenant B User
Tenant A → Tenant B Notification
Tenant A → Tenant B File
```

Every unauthorized attempt must fail safely.

---

# 60. Tenant Security Test Matrix

| Scenario                                   | Expected                 |
| ------------------------------------------ | ------------------------ |
| User accesses own tenant product           | Allowed                  |
| User accesses another tenant product       | Denied                   |
| User creates product for own tenant        | Allowed                  |
| User submits another tenant ID             | Ignored / rejected       |
| User references another tenant category    | Denied                   |
| User accesses another tenant sale          | Denied                   |
| User accesses another tenant invoice       | Denied                   |
| User accesses another tenant customer      | Denied                   |
| User accesses another tenant stock         | Denied                   |
| User accesses another tenant notification  | Denied                   |
| Store A user accesses Store A stock        | Allowed                  |
| Store A user accesses Store B stock        | Denied unless authorized |
| Store A user submits unauthorized store ID | Denied                   |
| User references another tenant's store     | Denied                   |

---

# 61. Tenant Isolation and Prisma

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

Store-scoped:

```javascript
const stock = await prisma.stock.findFirst({
  where: {
    productId,
    tenantId,
    storeId
  }
});
```

For updates, ownership should be included in the query whenever practical.

Example:

```javascript
await prisma.product.updateMany({
  where: {
    id: productId,
    tenantId
  },
  data: updateData
});
```

The exact Prisma patterns should be standardized during backend implementation.

---

# 62. Avoiding Global Query Shortcuts

Avoid unsafe generic functions such as:

```javascript
getById(id)
```

for tenant-owned resources when tenant context is not part of the access contract.

Prefer:

```javascript
getById({
  id,
  tenantId
});
```

For store-specific resources:

```javascript
getById({
  id,
  tenantId,
  storeId
});
```

This makes scope explicit.

---

# 63. Tenant Context Helper

The backend may expose trusted request context:

```javascript
req.context = {
  userId,
  tenantId,
  membershipId,
  activeStoreId,
  roleIds,
  permissions,
  capabilities
};
```

Services should use this trusted context.

The request body, query parameters, and frontend state must not override authorization context.

---

# 64. Tenant-Aware Logging

Application logs may include tenant context where appropriate.

Example:

```text
requestId=req_123
tenantId=tenant_abc
storeId=store_001
userId=user_456
action=SALE_CREATED
duration=84ms
```

However, logs must not unnecessarily expose:

```text
Passwords
Tokens
API Keys
Payment Credentials
Sensitive Personal Data
```

---

# 65. Tenant-Aware Metrics

Metrics should be designed carefully.

Potential metrics:

```text
api.requests
sales.count
inventory.adjustments
ai.jobs
queue.processing_time
```

Tenant IDs should not automatically become high-cardinality metric labels.

For example, thousands of tenant IDs as Prometheus labels can create unnecessary monitoring cost and complexity.

Tenant-specific investigation should primarily use:

```text
Logs
Traces
Audit Records
```

where appropriate.

---

# 66. Tenant-Aware Rate Limiting

Rate limits may operate at multiple levels:

```text
IP
User
Tenant
Endpoint
```

Examples:

```text
Login rate limit
API rate limit
AI usage limit
Report generation limit
Import limit
```

Tenant-level limits help prevent a single tenant from consuming disproportionate shared resources.

---

# 67. Resource Isolation / Noisy Neighbor Protection

Although tenants initially share infrastructure, Buzzsynx should account for noisy-neighbor scenarios.

One tenant performing:

```text
Large Imports
Massive Report Generation
Heavy AI Analysis
High API Traffic
```

should not unnecessarily degrade the entire platform.

Possible controls:

```text
Per-tenant rate limits
Queue concurrency limits
Job quotas
API throttling
Plan-based limits
Worker prioritization
```

These should be introduced based on actual platform needs.

---

# 68. Tenant Quotas

Future SaaS plans may define limits such as:

```text
Maximum Users
Maximum Products
Maximum Stores
Monthly Transactions
AI Usage
Storage
Reports
```

Conceptually:

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

# 69. Tenant Subscription

Subscription and billing are platform capabilities and should be separated from ordinary tenant business transactions.

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

Subscription status may influence:

```text
Feature Availability
Usage Limits
AI Quotas
Storage Limits
```

Subscription status must not replace RBAC.

A user can have permission to perform an action while the tenant's subscription does not include the required capability.

---

# 70. Tenant Data Export

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
Notify Authorized User
```

The export worker must operate strictly within the tenant context.

Store-specific exports must additionally respect store scope.

---

# 71. Tenant Data Import

Imports must also be tenant-scoped.

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

Uploaded files must never be allowed to determine arbitrary ownership.

For example, an uploaded CSV containing:

```text
tenantId = another_tenant
```

must not cause records to be created for that tenant.

The server determines ownership.

---

# 72. Tenant-Aware Search

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

For store-specific searches:

```text
WHERE tenantId = currentTenantId
AND storeId = authorizedStoreId
```

This applies to:

* Product search
* Customer search
* Supplier search
* Invoice search
* Sales search
* Inventory search

---

# 73. Tenant-Aware Reports

Generated reports must contain only authorized tenant data.

Example:

```text
Monthly Sales Report

Tenant:
Fashion Hub

Store:
Main Branch
```

The report generator should receive trusted scope such as:

```text
tenantId
storeId where applicable
dateRange
filters
```

and generate data only within that scope.

---

# 74. Tenant-Aware Scheduled Jobs

Scheduled jobs may include:

```text
Daily Sales Summary
Low Stock Scan
Expiry Scan
AI Analysis
Notification Processing
Report Generation
```

The scheduler should identify eligible tenants and queue tenant-specific jobs.

Conceptually:

```text
Scheduler
   ↓
Find Eligible Tenants
   ↓
Queue Tenant Jobs
   ↓
Worker
   ↓
Resolve Tenant Context
   ↓
Process
```

Store-specific jobs must additionally identify the relevant store.

---

# 75. Tenant Context Reset

Long-running workers must never rely on persistent mutable tenant state.

Correct:

```text
Job A
 ↓
Create Context for Tenant A
 ↓
Process
 ↓
Clear Context

Job B
 ↓
Create Context for Tenant B
 ↓
Process
```

Do not allow:

```text
Global currentTenantId
```

inside worker processes.

---

# 76. Tenant Data in Client State

Frontend state may contain:

```text
currentTenant
currentStore
user
permissions
capabilities
```

but this is only UI/application state.

The frontend must never be treated as the security authority.

```text
Frontend
   ↓
Request
   ↓
Backend
   ↓
Trusted Tenant / Store Context
```

---

# 77. Tenant Switching

If multi-tenant accounts are supported:

```text
User
 ↓
Tenant List
 ↓
Select Tenant
 ↓
Server Validates Membership
 ↓
Activate Tenant Context
 ↓
Reload Permissions / Capabilities
 ↓
Reload Store Context
```

After switching:

* Tenant-specific cached data must be cleared or re-keyed.
* Client state must refresh.
* Active subscription/features must be reloaded.
* Permissions must be re-evaluated.
* Available stores must be reloaded.
* Active store must be revalidated.

---

# 78. Store Switching

For users with access to multiple stores:

```text
Tenant
 ↓
Available Stores
 ↓
Select Store
 ↓
Server Validates Store Membership
 ↓
Activate Store Context
 ↓
Reload Store-Specific Data
```

Store switching must not change the tenant.

The selected store must always belong to the current tenant.

---

# 79. Tenant Context and Browser Storage

Sensitive authorization information should not be trusted merely because it exists in:

```text
localStorage
sessionStorage
cookies
```

Browser state can improve UX.

Actual authorization must come from server-side:

```text
Authentication
Membership
Roles
Permissions
Capabilities
Store Scope
```

---

# 80. Tenant-Aware API Contract

Ordinary tenant APIs should keep tenant context implicit.

Preferred:

```text
GET /api/v1/products
GET /api/v1/sales
GET /api/v1/inventory
```

The current authenticated tenant determines ownership.

Avoid exposing:

```text
GET /api/v1/all-sales
```

to tenant users.

Platform APIs are separate.

---

# 81. Platform Administration

Future Buzzsynx platform administrators may require cross-tenant capabilities.

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

They must not reuse ordinary tenant permissions as a shortcut.

---

# 82. Platform Admin Safety

Platform-level access is highly privileged.

Actions such as:

```text
Suspend Tenant
Inspect Tenant Configuration
Reset Tenant Access
Perform Migration
Export Tenant Data
Change Subscription
```

should require:

* Strong authorization
* Audit logs
* Explicit action boundaries
* Additional safeguards for destructive operations

Platform administrators should not automatically become owners of tenant businesses.

---

# 83. Cross-Tenant Operations

Cross-tenant operations should be extremely rare and explicitly designed.

Examples:

```text
Platform Analytics
Platform Billing
System Monitoring
Subscription Management
```

Normal tenant business services should never silently perform cross-tenant queries.

---

# 84. Tenant Isolation and Database Backups

Shared database backups contain data belonging to multiple tenants.

Therefore:

* Production backups must be secured.
* Backup access must be restricted.
* Backup credentials must be protected.
* Restore procedures must be tested.
* Tenant-level export/deletion procedures must account for shared storage.
* Backup retention must follow platform policy.
* Restored environments must maintain tenant isolation.

A tenant deletion operation must never compromise another tenant's data.

---

# 85. Future Tenant Scaling

The initial strategy is:

```text
Shared Database
Shared Schema
tenantId
storeId where applicable
```

Future strategies may include:

```text
Shared Database
Separate Schema
```

or:

```text
Database Per Tenant
```

for specific enterprise requirements.

Possible future architecture:

```text
                    Buzzsynx
                       │
        ┌──────────────┼──────────────┐
        │              │              │
     Standard        Growth       Enterprise
        │              │              │
   Shared Schema   Optimized     Dedicated DB
   tenantId        Shared DB     or isolated
```

This is a future scaling capability, not an MVP requirement.

---

# 86. Tenant Data Migration

If a tenant eventually moves from shared storage to dedicated infrastructure, the application should support a controlled migration.

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

Tenant identity must remain independent from physical database location.

---

# 87. Tenant Routing Abstraction

Future infrastructure may introduce:

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

Business services should ideally remain unaware of the physical storage strategy.

This keeps future tenant migration from requiring major business-logic rewrites.

---

# 88. Tenant Data Residency / Infrastructure Expansion

If Buzzsynx eventually serves enterprise customers with specific infrastructure or residency requirements, physical data location may become tenant-specific.

Potential abstraction:

```text
Tenant
   ↓
Data Region / Storage Location
   ↓
Database / Storage Provider
```

This is not required for the initial implementation.

The architecture should simply avoid making future isolation impossible.

---

# 89. Multi-Tenant Security Checklist

Before production:

* [ ] Every tenant-owned model has clear tenant ownership.
* [ ] Store-owned models have clear store ownership.
* [ ] Tenant membership is verified.
* [ ] Store membership/scope is verified.
* [ ] Tenant context is created server-side.
* [ ] Client tenant IDs are never trusted.
* [ ] Client store IDs are validated before use.
* [ ] All tenant queries are scoped.
* [ ] Store queries are scoped where required.
* [ ] Resource ownership is verified.
* [ ] Cross-tenant relationships are blocked.
* [ ] Cross-store access is blocked where unauthorized.
* [ ] Tenant-scoped unique constraints are defined.
* [ ] Store-scoped unique constraints are defined where required.
* [ ] Redis keys include tenant identity.
* [ ] Store-specific Redis keys include store identity.
* [ ] Background jobs include tenant identity.
* [ ] Store-specific jobs include store identity.
* [ ] Reports are tenant-scoped.
* [ ] AI processing is tenant-scoped.
* [ ] Notifications are tenant-scoped.
* [ ] File storage is tenant-scoped.
* [ ] Audit logs include tenant identity.
* [ ] Tenant isolation tests exist.
* [ ] Store isolation tests exist.
* [ ] IDOR tests exist.
* [ ] Platform APIs are separated from tenant APIs.
* [ ] Tenant deletion is controlled.
* [ ] Backup access is restricted.
* [ ] Worker tenant context cannot leak between jobs.

---

# 90. Multi-Tenant Testing Strategy

The test suite should use at least:

```text
Tenant A
Tenant B
```

with:

```text
Tenant A
 ├── Store A1
 └── Store A2

Tenant B
 └── Store B1
```

Tests should verify:

```text
Tenant A → sees Tenant A
Tenant B → sees Tenant B
Tenant A → cannot access Tenant B
Tenant B → cannot access Tenant A
```

and:

```text
Store A1 User
   ↓
Can access Store A1

Store A1 User
   ↓
Cannot access Store A2 unless authorized
```

Major modules must include tenant/store isolation tests.

---

# 91. Multi-Tenant Test Categories

### Authentication

* User authentication
* Tenant membership resolution
* Tenant switching
* Session invalidation

### Tenant Isolation

* Product access
* Customer access
* Supplier access
* Sales access
* Invoice access
* Payment access

### Store Isolation

* Inventory access
* POS access
* Store sales
* Store transfers
* Store reports

### Authorization

* Owner
* Admin / Manager
* Cashier
* Accountant
* Store Staff
* Super Admin

### Capability Isolation

* Enabled capability
* Disabled capability
* Unauthorized capability access

### Background Processing

* Tenant-specific jobs
* Store-specific jobs
* Worker context reset
* Duplicate jobs

### AI

* Tenant-specific AI data
* Store-specific analysis
* Cross-tenant data leakage prevention

### Storage

* Tenant file access
* Cross-tenant file access
* Signed URL authorization

---

# 92. Definition of Done

The multi-tenancy architecture is considered implementation-ready when:

* [ ] Tenant entity is defined.
* [ ] Store / Branch entity is defined.
* [ ] Tenant membership is defined.
* [ ] Membership store scope is defined.
* [ ] Tenant lifecycle is defined.
* [ ] Tenant context strategy is defined.
* [ ] Store context strategy is defined.
* [ ] Tenant resolution is defined.
* [ ] Store resolution is defined.
* [ ] Tenant-aware RBAC is defined.
* [ ] Store-aware authorization is defined.
* [ ] Tenant capability checks are defined.
* [ ] Tenant database relationships are defined.
* [ ] Store database relationships are defined.
* [ ] Tenant-scoped unique constraints are defined.
* [ ] Store-scoped unique constraints are defined where required.
* [ ] Tenant-aware caching is defined.
* [ ] Tenant/store-aware queues are defined.
* [ ] Tenant-aware AI processing is defined.
* [ ] Tenant-aware file storage is defined.
* [ ] Tenant-aware analytics are defined.
* [ ] Cross-tenant protection is tested.
* [ ] Cross-store protection is tested.
* [ ] IDOR protection is tested.
* [ ] Tenant switching behavior is defined for future multi-tenant users.
* [ ] Store switching behavior is defined.
* [ ] Platform administration boundaries are defined.
* [ ] Tenant deletion is controlled.
* [ ] Future tenant storage migration strategy is documented.

---

# 93. Core Multi-Tenancy Rules

The following rules are mandatory for Buzzsynx.

### Rule 1

> **Tenant identity must come from trusted backend context.**

### Rule 2

> **Never trust `tenantId` from the request body, query string, URL, or frontend state for authorization.**

### Rule 3

> **A client-provided `storeId` may be used as a requested scope, but it must always be validated against the current tenant, membership, permissions, and store access.**

### Rule 4

> **Every tenant-owned query must be tenant-scoped.**

### Rule 5

> **Every store-owned query must be tenant + store scoped where appropriate.**

### Rule 6

> **Every resource lookup must verify ownership and scope.**

### Rule 7

> **Related resources must belong to the same tenant.**

### Rule 8

> **Store-specific resources must belong to an authorized store of the current tenant.**

### Rule 9

> **Redis keys must be tenant-aware and store-aware where applicable.**

### Rule 10

> **Background jobs must carry tenant context and store context where required.**

### Rule 11

> **AI processing must remain tenant-isolated.**

### Rule 12

> **Tenant users must never receive platform-level permissions implicitly.**

### Rule 13

> **Cross-tenant operations require explicit platform-level authorization.**

### Rule 14

> **Cross-store operations require explicit permission and valid store scope.**

### Rule 15

> **PostgreSQL remains the authoritative source of tenant business data.**

---

# 94. Final Multi-Tenant Architecture

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
                    ┌───────┴───────┐
                    │               │
                  Stores       Tenant Settings
                    │
             ┌──────┼──────┐
             │      │      │
           Store A Store B Store C
             │      │      │
             └──────┼──────┘
                    │
                    ▼
            Store Scope / Access
                    │
          ┌─────────┴─────────┐
          │                   │
         RBAC            Capabilities
          │                   │
          └─────────┬─────────┘
                    │
                    ▼
              Business Services
                    │
          ┌─────────┼─────────┐
          │         │         │
      PostgreSQL   Redis    BullMQ
          │         │         │
          │         │         └── Tenant/Store Jobs
          │         │
          │         └──────────── Tenant/Store Cache
          │
          ▼
       Tenant Data
          │
    ┌─────┼──────────┐
    │     │          │
Products Inventory  Sales
    │     │          │
Suppliers Movements Payments
    │     │          │
Customers Purchases Invoices
```

---

# 95. Buzzsynx Multi-Tenant Business Model

The intended model is:

```text
                     Buzzsynx Platform
                            │
                            ▼
                         Tenant
                            │
                 ┌──────────┴──────────┐
                 │                     │
          Tenant Configuration     Capabilities
                 │                     │
                 └──────────┬──────────┘
                            │
                         Stores
                            │
                 ┌──────────┼──────────┐
                 │          │          │
              Store A    Store B    Store C
                 │          │          │
                 └──────────┼──────────┘
                            │
                       Memberships
                            │
                  ┌─────────┼─────────┐
                  │         │         │
                Owner     Manager   Cashier
                                      │
                                  Store Staff
```

This provides:

* One tenant with one store.
* One tenant with multiple stores.
* Different users with different permissions.
* Different users with different store access.
* Shared product/business master data where appropriate.
* Store-specific inventory and operations.
* Tenant-specific configuration.
* Tenant-specific capabilities.

---

# 96. Initial Buzzsynx Scope

The multi-tenancy architecture is intentionally broader than the first industry implementation.

The first complete business workflow is:

```text
Supermarket / Grocery
```

Initial tenant:

```text
Tenant
  ↓
Main Store
  ↓
Products
  ↓
Suppliers
  ↓
Purchasing
  ↓
Inventory
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
AI Insights
```

Future industries reuse the same tenant and store architecture.

For example:

```text
Future Pharmacy
       ↓
Same Tenant Model
       ↓
Same Store Model
       ↓
Same RBAC
       ↓
Same Core Product / Inventory / Sales Engine
       +
Pharmacy Capabilities
```

This keeps the core platform stable while allowing industry-specific functionality to evolve independently.

---

# 97. Final Principle

The Buzzsynx multi-tenancy model is:

> **One application, one shared business engine, multiple independent tenants, optional multiple stores per tenant, strict tenant and store-scoped access, configurable capabilities, and defense-in-depth isolation across the API, database, cache, queues, AI, files, analytics, and platform operations.**

The architectural principle is:

> **The tenant defines the business boundary. The store defines the operational boundary. Membership defines who belongs where. RBAC defines what a user can do. Capabilities define which business functionality is enabled. PostgreSQL records the authoritative business state.**

---

# 98. Related Documents

This document connects directly with:

```text
00-project-overview.md
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
14-development-standards.md
15-phase-wise-execution.md
16-feature-checklist.md
17-production-readiness.md
18-project-completion.md
```

If a conflict exists between this document and the API, database, security, or workflow documentation, the documents must be reconciled before implementation.

**Buzzsynx — First Brick, Not the Whole Building.**
