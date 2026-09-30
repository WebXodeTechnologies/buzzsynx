# Buzzsynx — Database Design

**Version:** v0.2
**Status:** Architecture-Aligned Database Baseline
**Product:** Buzzsynx
**Initial Industry:** Supermarket / Grocery Retail
**Database:** PostgreSQL
**ORM:** Prisma

---

# 1. Purpose

This document defines the database architecture, data model, relationships, constraints, tenant isolation strategy, transaction rules, indexing strategy, and data integrity standards for Buzzsynx.

Buzzsynx is a **multi-tenant business operations SaaS** built around a shared business engine with configurable industry capabilities.

The first complete implementation is:

> **Supermarket / Grocery Retail**

Other industries such as pharmacy, clothing, restaurant, and clinic are planned as **future industry capabilities**, not simultaneous MVP implementations.

The database must therefore support:

1. Shared core business functionality.
2. Strict tenant-level data isolation.
3. Store/branch-level operational scope.
4. Configurable industry capabilities.
5. High-integrity inventory management.
6. Transaction-safe POS operations.
7. Scalable analytics and reporting.
8. AI-ready historical business data.
9. Auditability.
10. Future SaaS scalability.

---

# 2. Database Technology

## 2.1 Primary Database

**PostgreSQL**

PostgreSQL is the authoritative source of truth for all transactional and business-critical data.

It stores:

* Tenants
* Users and memberships
* Products
* Inventory
* Purchases
* Sales
* Payments
* Customers
* Invoices
* Audit records
* Business configuration
* AI and analytics results where persistence is required

---

## 2.2 ORM

**Prisma**

Prisma is used for:

* Schema definition
* Version-controlled migrations
* Database access
* Relationships
* Transactions
* Query construction
* Validation support
* Database client management

---

# 3. Database Design Principles

## Principle 1 — PostgreSQL is the Source of Truth

Redis must never become the authoritative source for:

* Inventory
* Sales
* Payments
* Purchases
* Customers
* Products
* Financial records

Redis is used for:

* Caching
* Temporary state
* Rate limiting
* Session-related state where required
* Queue support
* Performance optimization

---

## Principle 2 — Tenant Isolation is Mandatory

Every tenant-owned business record must have a clear relationship to its tenant.

Conceptually:

```text
Tenant
│
├── Stores / Branches
│   ├── Stock
│   ├── Sales
│   ├── Purchases
│   └── Store-scoped Users
│
├── Products
├── Suppliers
├── Customers
├── Business Settings
├── Analytics
├── AI
└── Audit
```

A user belonging to Tenant A must never access Tenant B's records.

The backend must derive tenant context from authenticated membership and authorization context.

A client-provided `tenantId` must never be trusted as proof of access.

---

# 4. Tenant and Store Architecture

Buzzsynx follows this hierarchy:

```text
Super Admin
    │
    ▼
Tenant / Business
    │
    ├── Store / Branch
    │       │
    │       ├── Users / Memberships
    │       ├── Stock
    │       ├── POS
    │       └── Store Operations
    │
    ├── Products
    ├── Suppliers
    ├── Customers
    ├── Business Settings
    └── Tenant-wide Analytics
```

A tenant may have:

* One store
* Multiple stores
* A warehouse
* Future specialized stock locations

The MVP may begin with one store, but the schema should not require a redesign when additional branches are introduced.

---

# 5. Initial Product Scope

The database is designed as a reusable platform, but implementation priority is:

```text
Phase 1
Supermarket / Grocery
```

Future capability examples:

```text
Pharmacy
Clothing
Restaurant
Clinic
General Retail
```

These future industries must not force unnecessary MVP tables or workflows into the initial implementation.

Shared entities should remain generic where possible.

Industry-specific entities should be introduced through capability modules.

---

# 6. High-Level Entity Model

```text
Platform
│
├── Tenant
│   │
│   ├── Memberships
│   ├── Roles / Permissions
│   ├── Stores
│   │   └── Store Operations
│   │
│   ├── Products
│   │   ├── Categories
│   │   ├── Brands
│   │   └── Variants
│   │
│   ├── Suppliers
│   │   └── Purchases
│   │
│   ├── Customers
│   │
│   ├── Inventory
│   │   ├── Stock
│   │   └── Inventory Movements
│   │
│   ├── POS
│   │   ├── Sales
│   │   ├── Sale Items
│   │   └── Payments
│   │
│   ├── Invoices
│   ├── Analytics
│   ├── AI
│   ├── Notifications
│   └── Audit Logs
```

---

# 7. Tenant Model

A `Tenant` represents a business using Buzzsynx.

Example:

```text
Tenant
Name: ABC Supermarket
Industry: SUPERMARKET
Status: ACTIVE
```

Core fields:

```text
Tenant
------
id
name
slug
industryType
status
planId
settings
createdAt
updatedAt
```

Possible initial industry types:

```text
SUPERMARKET
```

Future:

```text
PHARMACY
CLOTHING
RESTAURANT
CLINIC
GENERAL_RETAIL
```

The system should allow additional industries without redesigning the shared core.

---

# 8. Tenant Lifecycle

A tenant may have the following lifecycle:

```text
PENDING
ACTIVE
SUSPENDED
ARCHIVED
```

Owner onboarding should not be blocked by manual platform approval.

Conceptually:

```text
User Signup
    ↓
Tenant Creation
    ↓
Initial Store Creation
    ↓
Owner Membership
    ↓
Default Configuration
    ↓
Dashboard Access
    ↓
Platform Review / Control
```

Super Admin may subsequently suspend, activate, or otherwise manage the tenant.

---

# 9. User and Membership Model

## 9.1 User

A `User` represents a Buzzsynx account.

```text
User
----
id
name
email
passwordHash
status
createdAt
updatedAt
```

A user account may participate in one or more tenants.

---

## 9.2 TenantUser / Membership

`TenantUser` is mandatory for tenant membership.

```text
TenantUser
----------
id
tenantId
userId
status
createdAt
updatedAt
```

A membership establishes:

```text
User
  ↓
TenantUser
  ↓
Tenant
```

A user can therefore potentially belong to multiple businesses without duplicating the underlying account.

---

# 10. Store Membership Scope

A membership may optionally be assigned to one or more stores depending on the user's role and business configuration.

Conceptually:

```text
User
 ↓
TenantUser
 ↓
Role + Permissions
 ↓
Store Scope
```

Examples:

```text
Owner
→ All stores

Admin / Manager
→ Assigned stores

Cashier
→ Assigned store

Store Staff
→ Assigned store

Accountant
→ Tenant-level financial access or configured store scope
```

The exact permission model is controlled by RBAC and store scope.

---

# 11. Roles and Permissions

Buzzsynx uses permission-based RBAC.

## Platform Role

```text
SUPER_ADMIN
```

Super Admin operates at the Buzzsynx platform level.

## Tenant Roles

```text
OWNER
ADMIN
MANAGER
CASHIER
ACCOUNTANT
STORE_STAFF
```

The application should not depend exclusively on role names.

Permissions should control actual capabilities.

Examples:

```text
product.create
product.update
product.archive

inventory.view
inventory.adjust
inventory.transfer

sale.create
sale.view
sale.cancel
sale.refund

purchase.create
purchase.receive
purchase.view

customer.view
customer.create
customer.update

payment.view
payment.record
payment.refund

report.view

settings.manage

users.manage
```

---

# 12. Role and Permission Entities

Core entities:

```text
Role
Permission
RolePermission
TenantUser
```

Conceptually:

```text
TenantUser
    ↓
Role
    ↓
RolePermission
    ↓
Permission
```

Permissions should be granular enough to support future role customization without creating a new role for every business variation.

---

# 13. Store / Branch Model

A store represents an operational branch of a tenant.

```text
Store
-----
id
tenantId
name
code
address
phone
status
timezone
createdAt
updatedAt
```

Examples:

```text
Main Store
Branch 1
Branch 2
Warehouse
```

The MVP may begin with one store.

The database should nevertheless support multiple stores from the beginning.

---

# 14. Business Configuration

Business configuration should remain separate from transactional records.

Configuration areas may include:

```text
Business Profile
Tax Settings
Currency
Timezone
Invoice Settings
POS Settings
Inventory Settings
Notification Settings
AI Settings
Industry Settings
```

Example:

```text
BusinessSettings
----------------
id
tenantId
currency
timezone
taxMode
invoicePrefix
createdAt
updatedAt
```

Store-specific settings may be introduced where operational behavior differs by branch.

---

# 15. Product Domain

The product domain is shared across most retail industries.

Core entities:

```text
Product
Category
Brand
ProductVariant
Unit
ProductImage
```

Not every future industry needs every field.

The shared product model should remain clean.

---

# 16. Product

A product represents a logical business item.

Examples:

```text
Basmati Rice 5kg
Tata Salt 1kg
Coca-Cola 750ml
```

Possible fields:

```text
Product
-------
id
tenantId
categoryId
brandId
name
description
sku
barcode
unitId
productType
status
costPrice
sellingPrice
taxRate
createdAt
updatedAt
```

For MVP, this model should remain focused on supermarket/grocery requirements.

---

# 17. Product Variants

Variants represent independently sellable versions of a product.

Examples from future clothing capability:

```text
T-Shirt
├── M / Black
├── M / White
├── L / Black
└── L / White
```

Possible model:

```text
ProductVariant
--------------
id
tenantId
productId
sku
barcode
attributes
costPrice
sellingPrice
status
createdAt
updatedAt
```

`attributes` may initially use JSON where appropriate.

Frequently queried attributes can be normalized later if justified by actual requirements.

---

# 18. Product Commercial Data

Product-level pricing represents current/default commercial information.

Historical transactions must not depend on the current Product price.

For example:

```text
Product current selling price = ₹120
```

Historical transaction:

```text
SaleItem.unitPrice = ₹100
```

The historical sale remains ₹100.

Therefore:

```text
Product
→ Current/default business data

SaleItem
→ Historical transaction snapshot
```

---

# 19. Category and Brand

Categories organize products.

Examples:

```text
Groceries
Beverages
Snacks
Dairy
Personal Care
Household
```

Brands provide product grouping.

Both should normally be tenant-scoped.

Example:

```text
Category
---------
id
tenantId
name
status
createdAt
updatedAt
```

---

# 20. Supplier Domain

Core entities:

```text
Supplier
Purchase
PurchaseItem
```

Supplier:

```text
Supplier
--------
id
tenantId
name
phone
email
address
taxNumber
status
createdAt
updatedAt
```

Supplier payments may later be modeled through a dedicated supplier payable/payment capability.

---

# 21. Purchase Domain

A purchase represents inventory acquired from a supplier.

Core fields:

```text
Purchase
--------
id
tenantId
storeId
supplierId
purchaseNumber
status
subtotal
taxAmount
discountAmount
totalAmount
purchaseDate
createdAt
updatedAt
```

Purchase order workflows may be introduced later.

For MVP, direct purchase/receiving workflows may be sufficient.

---

# 22. Purchase Items

```text
PurchaseItem
------------
id
tenantId
purchaseId
productId
variantId
quantity
unitCost
taxAmount
discountAmount
totalAmount
```

When inventory is received:

```text
Purchase
   ↓
PurchaseItem
   ↓
InventoryMovement
   ↓
Stock update
```

The stock update and movement creation must occur transactionally.

---

# 23. Inventory Domain

Inventory is one of the most critical domains in Buzzsynx.

Core entities:

```text
Stock
InventoryMovement
StockAdjustment
StockTransfer
```

Inventory must distinguish between:

```text
Historical stock changes
        ↓
InventoryMovement

Current operational state
        ↓
Stock
```

---

# 24. Stock

`Stock` represents the current operational inventory state.

```text
Stock
-----
id
tenantId
storeId
productId
variantId
quantity
reservedQuantity
updatedAt
```

A unique constraint should prevent duplicate stock records for:

```text
tenantId
storeId
productId
variantId
```

If reservations are not required in the initial MVP, `reservedQuantity` may be deferred.

If used:

```text
availableQuantity
=
quantity - reservedQuantity
```

It does not need to become a separate authoritative stored value unless required for performance.

---

# 25. Inventory Movement Ledger

Every meaningful stock change must create an inventory movement.

```text
InventoryMovement
-----------------
id
tenantId
storeId
productId
variantId
movementType
quantity
referenceType
referenceId
reason
createdBy
createdAt
```

Movement types:

```text
OPENING_STOCK

PURCHASE
SALE

SALE_RETURN
PURCHASE_RETURN

TRANSFER_IN
TRANSFER_OUT

DAMAGE
EXPIRY

ADJUSTMENT
```

Example:

```text
Purchase 100
    ↓
+100 PURCHASE

Sale 5
    ↓
-5 SALE

Damage 2
    ↓
-2 DAMAGE
```

The movement ledger provides the historical stock trail.

---

# 26. Inventory Consistency

Conceptually:

```text
Current Stock =

Opening Stock
+ Purchases
+ Returns In
+ Transfers In
- Sales
- Returns Out
- Damage
- Expiry
- Transfers Out
± Adjustments
```

The implementation may maintain a current `Stock` table for fast operational access while preserving the movement ledger as the historical record.

---

# 27. Inventory Adjustment

An adjustment should record the actual counted quantity and calculated difference.

Conceptually:

```text
Current Stock
      ↓
Physical Count
      ↓
Calculate Difference
      ↓
Create Adjustment
      ↓
Create InventoryMovement
      ↓
Update Stock
```

Adjustment records should preserve:

* Reason
* Previous quantity where required
* Counted quantity
* Difference
* User
* Store
* Timestamp

---

# 28. Stock Transfer

Transfers support movement between stores or stock locations.

Conceptually:

```text
Store A
   ↓
TRANSFER_OUT
   ↓
Transfer
   ↓
TRANSFER_IN
   ↓
Store B
```

Transfers must be transactional and auditable.

The MVP may keep advanced transfer workflows minimal, but the database should not prevent future multi-store operations.

---

# 29. POS Domain

POS is responsible for fast transaction processing.

Core entities:

```text
Sale
SaleItem
Payment
Invoice
SaleReturn
SaleReturnItem
Refund
```

The supermarket POS should optimize for:

```text
Barcode / Search
      ↓
Product Lookup
      ↓
Cart
      ↓
Stock Validation
      ↓
Price / Tax / Discount
      ↓
Payment
      ↓
Sale Completion
      ↓
Inventory Movement
      ↓
Invoice
```

---

# 30. Sale

A sale represents a customer transaction.

```text
Sale
----
id
tenantId
storeId
customerId
saleNumber
status
subtotal
discountAmount
taxAmount
totalAmount
createdBy
createdAt
updatedAt
completedAt
```

Possible statuses:

```text
DRAFT
PENDING
COMPLETED
CANCELLED
PARTIALLY_REFUNDED
REFUNDED
```

`customerId` may be nullable because supermarket POS must support anonymous/walk-in customers.

---

# 31. Sale Items

```text
SaleItem
--------
id
tenantId
saleId
productId
variantId
quantity
unitPrice
discountAmount
taxAmount
totalAmount
```

Sale items must preserve the commercial values used during the transaction.

They must not dynamically reference the current product selling price.

---

# 32. Payment Domain

Payment is a separate business concept from the sale.

One sale may have multiple payments.

Example:

```text
Sale Total = ₹1,000

Cash = ₹400
UPI = ₹600
```

Model:

```text
Payment
-------
id
tenantId
storeId
saleId
paymentMethod
amount
status
transactionReference
provider
paidAt
createdAt
```

Payment methods may include:

```text
CASH
CARD
UPI
BANK_TRANSFER
WALLET
OTHER
```

Credit/receivable should not simply be treated as a successful payment method.

It should have an explicit receivable/credit workflow when implemented.

---

# 33. Payment Lifecycle

Payment status should support asynchronous payment providers.

Possible states:

```text
PENDING
AUTHORIZED
PAID
FAILED
CANCELLED
REFUNDED
PARTIALLY_REFUNDED
```

For external gateways:

```text
Sale
 ↓
Payment Intent
 ↓
Payment Provider
 ↓
Webhook / Verification
 ↓
Payment Status
 ↓
Sale Finalization
```

Gateway webhooks must be idempotent and verified.

---

# 34. Invoice Domain

An invoice represents the financial document associated with a sale.

```text
Invoice
-------
id
tenantId
storeId
saleId
invoiceNumber
status
subtotal
taxAmount
discountAmount
totalAmount
issuedAt
createdAt
```

Invoice numbering must be unique according to the tenant/store numbering policy.

Example:

```text
INV-2026-000001
INV-2026-000002
```

---

# 35. Invoice Record vs Invoice Document

The invoice record is part of the transactional business operation.

The actual PDF/document generation should not unnecessarily block the critical POS transaction.

Preferred architecture:

```text
Database Transaction
        ↓
Invoice Record Created
        ↓
COMMIT
        ↓
BullMQ Job
        ↓
PDF Generation
        ↓
Email / WhatsApp / Download / Print
```

A failure in PDF generation or notification must not roll back a successfully completed sale.

---

# 36. Customer Domain

Customers are tenant-specific.

```text
Customer
--------
id
tenantId
name
phone
email
address
taxNumber
status
createdAt
updatedAt
```

Customer association with a sale is optional for walk-in retail transactions.

Future capabilities may include:

```text
Purchase History
Credit Balance
Loyalty
Preferences
Marketing Consent
```

These should be introduced as dedicated capabilities rather than making the base customer model unnecessarily large.

---

# 37. Returns and Refunds

Returns must be represented as separate business events.

Do not silently modify the original sale.

Conceptually:

```text
Original Sale
     ↓
Sale Return
     ↓
Inventory Movement
     ↓
Refund / Credit / Exchange
```

Core entities:

```text
SaleReturn
SaleReturnItem
Refund
```

A return should preserve:

* Original sale reference
* Returned items
* Quantity
* Reason
* Condition/disposition
* Store
* User
* Refund state

Not every returned item must automatically be returned to sellable stock.

For example:

```text
Customer Return
      ↓
Damaged
      ↓
DAMAGE movement
```

rather than:

```text
Customer Return
      ↓
Sellable Stock
```

---

# 38. Analytics Data

Analytics should initially use PostgreSQL transactional data.

Examples:

```text
Daily Sales
Monthly Sales
Top Products
Low Stock
Dead Stock
Purchase Trends
Customer Trends
Profit Estimates
```

Initial strategy:

```text
PostgreSQL
    ↓
Queries / Aggregations
    ↓
Dashboard
```

As volume grows, introduce:

```text
Summary Tables
Materialized Views
Scheduled Aggregations
Read Models
```

A separate analytics database should only be introduced when justified by actual scale.

---

# 39. AI Data Architecture

AI should consume authoritative or derived business data.

Preferred flow:

```text
PostgreSQL
    ↓
Analytics / Business Data
    ↓
AI Processing
    ↓
AI Insight / Recommendation
    ↓
Business User
```

Possible future entities:

```text
AIInsight
AIRecommendation
AIForecast
AIJob
```

These are not all required for the initial MVP.

---

# 40. AI Safety Boundary

AI must not directly bypass business services.

Incorrect:

```text
AI
 ↓
Direct Database Update
```

Preferred:

```text
AI
 ↓
Recommendation
 ↓
Business Service
 ↓
Validation
 ↓
Authorization
 ↓
Database Transaction
```

Example:

```text
AI detects low stock
       ↓
Reorder recommendation
       ↓
Owner reviews
       ↓
Purchase workflow
       ↓
Purchase created
       ↓
Inventory received
```

AI provides intelligence.

Business services remain authoritative.

---

# 41. Notification Domain

Notifications may include:

```text
Notification
NotificationPreference
NotificationDelivery
```

Examples:

```text
Low Stock
Expiry Alert
Payment Failure
AI Recommendation
Purchase Received
System Alert
```

Notifications should normally be generated asynchronously after the successful business transaction.

---

# 42. Audit Log

Auditability is required for important business operations.

```text
AuditLog
--------
id
tenantId
storeId
userId
action
entityType
entityId
metadata
requestId
ipAddress
userAgent
createdAt
```

Examples:

```text
PRODUCT_CREATED
PRODUCT_UPDATED
SALE_CREATED
SALE_CANCELLED
SALE_REFUNDED
STOCK_ADJUSTED
STOCK_TRANSFERRED
USER_ROLE_CHANGED
SETTINGS_UPDATED
```

Audit records should generally be append-only.

---

# 43. Soft Delete Strategy

Soft deletion should be used only where appropriate.

Possible:

```text
deletedAt
```

Suitable examples:

```text
Product
Customer
Supplier
Category
Brand
User
```

Completed transactions should generally not be deleted.

Historical financial and inventory records should be preserved.

For transactional records, status/reversal/return mechanisms should normally be preferred over deletion.

---

# 44. IDs

Database entities should use stable unique identifiers.

Recommended:

```text
UUID
```

Human-readable business numbers must remain separate.

Example:

```text
Database ID:
550e8400-e29b-41d4-a716-446655440000

Sale Number:
SAL-2026-000182
```

Human-readable numbers must never be the primary database identity.

---

# 45. Timestamps

Business entities should generally include:

```text
createdAt
updatedAt
```

Transactional entities may additionally include:

```text
completedAt
cancelledAt
paidAt
issuedAt
receivedAt
```

Persistence should use UTC.

Tenant timezone configuration controls business-facing presentation and reporting.

---

# 46. Monetary Values

Money must use exact decimal representation.

Recommended database representation:

```text
Decimal / NUMERIC
```

Applicable fields include:

```text
Price
Tax
Discount
Subtotal
Total
Payment
Refund
Cost
Profit
```

JavaScript floating-point arithmetic must not be treated as the authoritative representation for financial calculations.

---

# 47. Critical Transaction Management

Critical business operations must use PostgreSQL transactions.

## POS Transaction

Conceptually:

```text
BEGIN TRANSACTION

Validate Tenant
Validate Store Scope
Validate Product
Validate Stock
Calculate Price / Tax / Discount

Create Sale
Create Sale Items
Record Payment State
Create Inventory Movements
Update Stock
Create Invoice Record
Create Audit Event

COMMIT
```

If a critical operation fails:

```text
ROLLBACK
```

The transaction must not leave the database in a partially completed business state.

External payment provider communication should not be treated as an ordinary database operation inside a long-running transaction.

---

# 48. Purchase Receiving Transaction

```text
BEGIN

Validate Purchase
Validate Store Scope
Receive Goods
Create / Update Purchase Items
Create Inventory Movements
Update Stock
Update Purchase Status
Create Audit Event

COMMIT
```

All inventory changes must remain consistent with the purchase receiving state.

---

# 49. Concurrency and Stock Safety

Inventory operations must account for concurrent POS transactions.

Example:

```text
Stock = 5
```

Two terminals simultaneously attempt:

```text
Terminal A → Buy 4
Terminal B → Buy 3
```

The database must prevent both transactions from independently assuming five units are available.

Appropriate mechanisms may include:

* PostgreSQL row-level locking
* Atomic stock updates
* Transaction isolation
* Conditional updates
* Proper transaction boundaries

The final strategy must be validated through concurrency tests.

Inventory safety is a database integrity requirement, not a frontend responsibility.

---

# 50. Database Constraints

## 50.1 Unique Constraints

Examples:

```text
Tenant.slug

User.email

TenantUser(tenantId, userId)

Store(tenantId, code)

Product(tenantId, sku)

ProductVariant(tenantId, sku)

Invoice(tenantId, invoiceNumber)

Sale(tenantId, saleNumber)
```

Where uniqueness depends on store scope, include `storeId`.

For example:

```text
(tenantId, storeId, invoiceNumber)
```

The exact numbering scope should be finalized in the invoice/business settings design.

---

## 50.2 Foreign Keys

Foreign keys should be used wherever practical.

Examples:

```text
SaleItem.saleId → Sale.id

SaleItem.productId → Product.id

PurchaseItem.purchaseId → Purchase.id

PurchaseItem.productId → Product.id

InventoryMovement.productId → Product.id

InventoryMovement.storeId → Store.id

TenantUser.tenantId → Tenant.id
```

---

## 50.3 Check Constraints

Database-level constraints should protect important invariants.

Examples:

```text
quantity > 0
amount >= 0
price >= 0
taxRate >= 0
```

Application-level validation remains necessary.

Database constraints provide an additional integrity boundary.

---

# 51. Tenant and Store Integrity

Cross-tenant references must never be allowed.

For example:

```text
Tenant A Product
        ↓
Tenant B SaleItem
```

must be impossible.

Similarly, store-scoped records must not accidentally reference a record belonging to another tenant or unauthorized store.

Application services must validate tenant/store ownership before performing cross-entity operations.

Where practical, database constraints should reinforce these relationships.

---

# 52. Indexing Strategy

Indexes should follow actual query patterns.

Common patterns include:

```text
tenantId

tenantId + status

tenantId + createdAt

tenantId + sku

tenantId + barcode

tenantId + productId

tenantId + storeId

tenantId + customerId

tenantId + supplierId
```

Examples:

```text
Product
INDEX (tenantId, sku)

Product
INDEX (tenantId, barcode)

Sale
INDEX (tenantId, storeId, createdAt)

InventoryMovement
INDEX (tenantId, storeId, productId, createdAt)

Stock
INDEX (tenantId, storeId, productId)
```

Exact indexes should be finalized after actual query patterns are implemented and measured.

Do not blindly index every field.

---

# 53. POS Search Strategy

Supermarket POS must support fast lookup by:

```text
Barcode
SKU
Product Name
Variant SKU
```

Conceptually:

```text
Barcode Scan
     ↓
Product Lookup
     ↓
Variant Resolution
     ↓
Store Stock Lookup
     ↓
POS Cart
```

PostgreSQL indexing should be sufficient for the initial implementation.

A dedicated search engine should only be introduced if actual scale or search requirements justify it.

---

# 54. Redis and Database Relationship

Architecture:

```text
Application
    │
    ├── PostgreSQL
    │       ↓
    │   Source of Truth
    │
    └── Redis
            ↓
      Cache / Temporary State
```

Redis may cache:

```text
Product lookup
Dashboard summaries
Frequently accessed configuration
Rate-limit counters
Temporary POS state
```

When cached data becomes stale, PostgreSQL remains authoritative.

Critical business writes must not depend on Redis availability.

---

# 55. Queue and Database Relationship

BullMQ jobs should reference database records rather than duplicating large business payloads where practical.

Example:

```text
Queue Job
   ↓
tenantId
storeId
entityId
jobType
```

Worker:

```text
Job
 ↓
Fetch PostgreSQL Data
 ↓
Process
 ↓
Persist Result
```

Jobs should support:

* Retry handling
* Idempotency
* Failure handling
* Logging
* Appropriate timeout behavior

---

# 56. Post-Transaction Background Processing

Non-critical work should happen after successful database commit.

Examples:

```text
Sale committed
    ↓
Queue
    ├── Generate invoice PDF
    ├── Send notification
    ├── Update analytics summary
    └── Generate AI-related processing
```

These operations must not unnecessarily block the core POS transaction.

A transactional outbox or equivalent reliable post-commit dispatch mechanism may be introduced when required to guarantee event delivery.

---

# 57. Data Lifecycle

Buzzsynx should distinguish between:

## Operational Data

Frequently accessed:

```text
Products
Stock
Customers
Active Sales
```

## Historical Data

Long-term business records:

```text
Completed Sales
Payments
Invoices
Inventory Movements
Audit Logs
```

## Derived Data

Calculated or generated:

```text
Analytics
Forecasts
AI Insights
Dashboard Summaries
```

Derived data should be rebuildable from authoritative data where practical.

---

# 58. Data Retention

Retention requirements depend on:

* Business requirements
* Applicable regulations
* Financial/accounting requirements
* Tenant plan
* Storage costs

The application should not permanently delete important transactional history simply because a user removes an item from the UI.

Retention and deletion policies must be finalized before production launch.

---

# 59. Backup and Recovery

Production PostgreSQL must have:

```text
Automated Backups
Point-in-Time Recovery where supported
Backup Monitoring
Restore Testing
Recovery Documentation
```

A backup strategy is not considered reliable until restoration has been tested.

---

# 60. Migration Strategy

All database schema changes must use version-controlled Prisma migrations.

```text
prisma/
├── schema.prisma
├── seed.js
└── migrations/
    ├── migration_001
    ├── migration_002
    └── migration_003
```

Production schema changes must not be performed manually without an appropriate migration process.

Migration principles:

1. Make changes backward-compatible where possible.
2. Test migrations locally.
3. Test migrations in staging.
4. Back up production before risky migrations.
5. Deploy application and schema changes in a controlled order.

---

# 61. Environment Separation

Buzzsynx should maintain separate databases for:

```text
Development
Staging
Production
```

Example:

```text
Local Docker PostgreSQL
        ↓
Development

AWS RDS
        ↓
Staging

AWS RDS
        ↓
Production
```

Development data must never be assumed to represent production data.

---

# 62. Seed Data

Prisma seed scripts should provide development and testing data such as:

```text
Demo Tenant
Demo Store
Demo Users
Roles
Permissions
Categories
Brands
Products
Customers
Suppliers
Sample Inventory
```

For the initial MVP, the primary demo dataset should represent:

```text
Demo Supermarket
```

Future industry demo datasets can be added later:

```text
Demo Pharmacy
Demo Clothing Store
Demo Restaurant
```

These should not become implementation dependencies for the supermarket MVP.

---

# 63. Analytics and AI Data Safety

Analytics and AI systems may read:

```text
Products
Sales
Purchases
Inventory
Customers
Payments
Business configuration
```

subject to authorization and data access rules.

They must not directly modify critical transactional records.

The authoritative flow remains:

```text
Business Data
     ↓
Analytics / AI
     ↓
Insight / Recommendation
     ↓
Business Workflow
     ↓
Validation + Authorization
     ↓
Transaction
     ↓
PostgreSQL
```

---

# 64. Future Industry Capabilities

Future industries should extend the shared database model instead of duplicating the entire platform.

## Pharmacy

Potential future entities:

```text
MedicineDetails
MedicineBatch
Manufacturer
Prescription
```

Capabilities may include:

```text
Batch tracking
Expiry tracking
Prescription requirements
Batch-aware stock deduction
```

---

## Clothing

Potential future entities/attributes:

```text
Size
Color
Material
Variant attributes
```

Product variants can maintain independent:

```text
SKU
Barcode
Price
Stock
```

---

## Restaurant

Potential future entities:

```text
Menu
MenuItem
Ingredient
Recipe
RecipeItem
DiningTable
RestaurantOrder
OrderItem
KitchenOrder
```

Conceptually:

```text
Menu Item
    ↓
Recipe
    ↓
Ingredients
    ↓
Inventory Consumption
```

These are future capabilities and should not unnecessarily enter the supermarket MVP schema.

---

# 65. Industry Capability Principle

The database architecture follows:

```text
Shared Core
     +
Tenant Configuration
     +
Industry Capability
```

Rather than:

```text
Pharmacy Database
Supermarket Database
Clothing Database
Restaurant Database
```

This allows Buzzsynx to maintain one shared business engine while adding specialized capabilities when justified.

---

# 66. Database Security

Database security requirements include:

```text
Strong credentials
Encrypted connections
Secrets outside source control
Least-privilege database users
Production network restrictions
Backups
Audit logging
Migration control
```

Application users must never receive direct database credentials.

Only authorized backend services should communicate with PostgreSQL.

---

# 67. Application Database Access

Preferred flow:

```text
Route / Controller
       ↓
Service
       ↓
Repository / Data Access
       ↓
Prisma
       ↓
PostgreSQL
```

Business rules should not be scattered throughout controllers.

Example:

```text
SaleController
      ↓
SaleService
      ↓
InventoryService
      ↓
PaymentService
      ↓
Prisma Transaction
```

The exact service boundaries may evolve, but transaction ownership must remain clear.

---

# 68. Repository Responsibility

Repositories/data-access functions primarily handle:

```text
Queries
Creates
Updates
Deletes
Transactions
Database-specific operations
```

Services handle:

```text
Business rules
Validation orchestration
Workflow
Authorization context
Domain logic
```

This separation should remain consistent across modules.

---

# 69. Database Observability

Production database monitoring should track:

```text
CPU
Memory
Storage
Connections
Query Latency
Slow Queries
Locks
Deadlocks
Transaction Failures
Backup Status
```

When deployed on AWS, RDS and CloudWatch can provide infrastructure-level monitoring.

Application database failures should also be captured through structured logging and, when enabled, Sentry.

---

# 70. Performance Strategy

Performance optimization should follow:

```text
Correctness
    ↓
Measurement
    ↓
Indexing
    ↓
Query Optimization
    ↓
Caching
    ↓
Aggregation
    ↓
Scaling
```

Do not introduce complex database infrastructure before identifying an actual bottleneck.

---

# 71. Future Database Scaling

Initial architecture:

```text
One PostgreSQL Database
        +
Shared Schema
        +
Tenant Isolation
        +
Proper Indexes
        +
Transactions
        +
Redis
```

Possible future strategies:

```text
Read Replicas
Partitioning
Analytics Database
Data Warehouse
Tenant-Specific Databases
Database Sharding
```

These are future scaling options, not initial requirements.

---

# 72. Multi-Tenant Evolution

Initial:

```text
Shared Database
Shared Schema
tenantId Isolation
```

Possible future:

```text
Shared Database
Separate Schema per Tenant
```

or:

```text
Database per Enterprise Tenant
```

The application architecture should avoid tightly coupling business logic to the initial storage strategy.

---

# 73. Database Design Rules

The following rules are mandatory.

### Rule 1

Every tenant-owned entity must have a clear tenant relationship.

### Rule 2

Never trust a client-provided `tenantId` for authorization.

### Rule 3

Every business-critical query must be tenant-scoped.

### Rule 4

Store-scoped operations must validate store authorization.

### Rule 5

Inventory changes must create inventory movements.

### Rule 6

The `Stock` table represents current operational state; the movement ledger preserves historical changes.

### Rule 7

Completed financial transactions must preserve historical values.

### Rule 8

Critical workflows must use database transactions.

### Rule 9

Money must use exact decimal representation.

### Rule 10

Foreign keys should be used to maintain relationships.

### Rule 11

Indexes should follow real query patterns.

### Rule 12

AI must not bypass normal business transaction services.

### Rule 13

Redis must never replace PostgreSQL as the source of truth.

### Rule 14

Schema changes must be version-controlled through migrations.

### Rule 15

Future industry capabilities must not unnecessarily complicate the initial supermarket MVP.

---

# 74. Database Implementation Order

Database implementation should follow the actual MVP roadmap.

## Phase 1 — Foundation

```text
Tenant
User
TenantUser
Store
Role
Permission
RolePermission
BusinessSettings
```

## Phase 2 — Product

```text
Product
Category
Brand
ProductVariant
Unit
```

## Phase 3 — Inventory

```text
Stock
InventoryMovement
StockAdjustment
StockTransfer
```

## Phase 4 — Purchasing

```text
Supplier
Purchase
PurchaseItem
```

## Phase 5 — POS

```text
Customer
Sale
SaleItem
Payment
Invoice
```

## Phase 6 — Returns

```text
SaleReturn
SaleReturnItem
Refund
```

## Phase 7 — Platform Operations

```text
Notification
NotificationPreference
NotificationDelivery
AuditLog
```

## Phase 8 — Intelligence

```text
Analytics Summaries
AIInsight
AIRecommendation
AIForecast
AIJob
```

Only the models actually required by the implemented workflow should be introduced.

## Phase 9 — Future Industry Capabilities

```text
MedicineDetails
MedicineBatch
Prescription
Clothing-specific attributes
Menu
Ingredient
Recipe
KitchenOrder
```

These are not required for the initial supermarket implementation.

---

# 75. Prisma Schema Strategy

Initially:

```text
prisma/
├── schema.prisma
├── seed.js
└── migrations/
```

A single `schema.prisma` file is acceptable during the initial implementation.

As the project grows, organization should prioritize:

* Readability
* Prisma compatibility
* Migration safety
* Developer maintainability

Do not artificially split the schema before there is a real maintenance problem.

---

# 76. Database Testing Requirements

## Tenant Isolation

```text
Tenant A cannot access Tenant B data.
```

## Store Isolation

```text
Store A users cannot access unauthorized Store B data.
```

## Inventory

```text
Purchase increases stock.
Sale decreases stock.
Return increases stock where appropriate.
Damage decreases stock.
Transfer moves stock between stores.
Adjustment reconciles stock.
```

## POS Atomicity

```text
Sale
+
Payment State
+
Inventory Movement
+
Stock Update
+
Invoice Record
```

must remain transactionally consistent.

## Concurrency

Concurrent sales must not create invalid stock states.

## Constraints

Duplicate:

```text
SKU
Barcode
Invoice Number
Sale Number
```

must be handled according to their defined tenant/store scope.

## Financial Accuracy

```text
Subtotal
Tax
Discount
Total
Payment
Refund
```

must remain mathematically consistent.

---

# 77. Database Definition of Done

The database foundation is considered ready for implementation when:

* [ ] Core entities are defined.
* [ ] Tenant relationships are defined.
* [ ] Store/branch relationships are defined.
* [ ] Membership and RBAC models are defined.
* [ ] Foreign-key relationships are defined.
* [ ] Unique constraints are defined.
* [ ] Required indexes are identified.
* [ ] Inventory movement model is finalized.
* [ ] Stock model is finalized.
* [ ] POS transaction model is finalized.
* [ ] Payment lifecycle is finalized.
* [ ] Invoice model is finalized.
* [ ] Return/refund model is finalized.
* [ ] Supermarket MVP capability requirements are defined.
* [ ] Future industry models are clearly separated.
* [ ] Audit model is defined.
* [ ] Monetary fields use exact decimal types.
* [ ] Critical workflows have transaction boundaries.
* [ ] Concurrency strategy is defined.
* [ ] Prisma schema is validated.
* [ ] Prisma migrations are tested.
* [ ] Seed data is available.
* [ ] Tenant isolation tests are implemented.
* [ ] Store-scope authorization tests are implemented.
* [ ] Database backup strategy is defined for staging/production.

---

# 78. Recommended Core Relationship Map

```text
Platform
   │
   └── Tenant / Business
          │
          ├── Memberships
          │     └── Users
          │
          ├── Roles / Permissions
          │
          ├── Stores / Branches
          │     ├── Stock
          │     ├── Sales
          │     ├── Purchases
          │     └── Inventory Movements
          │
          ├── Products
          │     ├── Categories
          │     ├── Brands
          │     └── Variants
          │
          ├── Suppliers
          │     └── Purchases
          │
          ├── Customers
          │
          ├── Invoices
          │
          ├── Analytics
          │
          ├── AI
          │
          ├── Notifications
          │
          └── Audit Logs
```

---

# 79. Supermarket MVP Data Flow

The canonical Buzzsynx database workflow is:

```text
Tenant
   ↓
Store
   ↓
Products
   ↓
Suppliers
   ↓
Purchase / Receiving
   ↓
Inventory Movement
   ↓
Stock
   ↓
POS Search / Barcode
   ↓
Cart
   ↓
Stock Validation
   ↓
Price / Tax / Discount
   ↓
Payment
   ↓
Sale
   ↓
Inventory Movement
   ↓
Stock Update
   ↓
Invoice Record
   ↓
Analytics
   ↓
AI Insights
   ↓
Notifications / Automation
```

The critical transaction is authoritative.

Analytics, AI, notifications, and document generation operate around the transactional core rather than replacing it.

---

# 80. Future Pharmacy Data Flow

Future capability example:

```text
Tenant
   ↓
Medicine Product
   ↓
Medicine Details
   ↓
Medicine Batch
   ↓
Purchase
   ↓
Inventory
   ↓
Batch-aware Stock
   ↓
POS Sale
   ↓
Payment
   ↓
Invoice
```

This is a future capability, not part of the initial supermarket database implementation.

---

# 81. Future Clothing Data Flow

Future capability example:

```text
Tenant
   ↓
Product
   ↓
Variants
   ├── M / Black
   ├── M / White
   ├── L / Black
   └── L / White
   ↓
Store Stock
   ↓
POS
   ↓
Sale
   ↓
Payment
   ↓
Invoice
```

Each sellable variant may maintain independent SKU, barcode, price, and stock.

---

# 82. Future Restaurant Data Flow

Future capability example:

```text
Tenant
   ↓
Menu Item
   ↓
Recipe
   ↓
Ingredients
   ↓
Inventory
   ↓
Restaurant Order
   ↓
Kitchen Workflow
   ↓
Payment
   ↓
Sale
   ↓
Inventory Consumption
```

This capability will integrate with the shared business engine when implemented.

---

# 83. Database Security Boundary

The database architecture follows:

```text
Client
  ↓
Frontend
  ↓
Backend Authentication
  ↓
Tenant Resolution
  ↓
Membership
  ↓
Role / Permission
  ↓
Store Scope
  ↓
Business Service
  ↓
Prisma
  ↓
PostgreSQL
```

The client must never directly access PostgreSQL.

The backend is responsible for enforcing:

* Tenant isolation
* Store isolation
* Authorization
* Validation
* Business rules
* Transaction boundaries

---

# 84. Final Database Architecture

```text
                         BUZZSYNX
                            │
                     TENANT / BUSINESS
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
      MEMBERSHIPS        PRODUCTS          SETTINGS
          │                 │
       RBAC          ┌──────┼──────┐
                     │      │      │
                 CATEGORY  BRAND  VARIANTS
                            │
                         STORES
                            │
                    ┌───────┼────────┐
                    │       │        │
                  STOCK    POS    PURCHASING
                    │       │        │
               MOVEMENTS  SALES    PURCHASES
                            │
                      ┌─────┼─────┐
                      │     │     │
                    ITEMS PAYMENT CUSTOMER
                            │
                         INVOICE
                            │
                       ANALYTICS
                            │
                         AI
                            │
                    ┌───────┼───────┐
                    │       │       │
                 INSIGHTS FORECASTS ALERTS

Future Industry Capabilities
             │
      ┌──────┼──────┬──────────┐
      │      │      │          │
  Pharmacy Clothing Restaurant Clinic
      │      │      │          │
      └──────┴──────┴──────────┘
                 │
        Shared Business Engine
```

---

# 85. Fundamental Database Principle

Buzzsynx follows this principle:

> **One shared business platform, strict tenant and store isolation, transactional integrity, ledger-based inventory, configurable industry capabilities, and historical business data that can power analytics and AI.**

The database has three fundamental layers:

```text
AUTHORITATIVE DATA
        ↓
PostgreSQL
        ↓
Business Transactions

DERIVED DATA
        ↓
Analytics / Aggregations
        ↓
Dashboards / Reports

INTELLIGENCE
        ↓
AI
        ↓
Insights / Recommendations
```

The database remains the foundation.

The application enforces what is allowed.

AI helps understand what happened and what may happen next.

**Buzzsynx — First Brick, Not the Whole Building.**
