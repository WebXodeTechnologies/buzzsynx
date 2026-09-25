# Buzzsynx — Database Design

## 1. Purpose

This document defines the database architecture, data model, relationships, constraints, tenant isolation strategy, transaction rules, indexing strategy, and data integrity standards for Buzzsynx.

Buzzsynx is a multi-tenant business operations SaaS designed to support different types of businesses through a shared application platform.

Examples:

* Tenant A → Pharmacy
* Tenant B → Supermarket
* Tenant C → Clothing Store
* Tenant D → Restaurant

The database must therefore support:

1. Shared core business functionality.
2. Strict tenant-level data isolation.
3. Industry-specific capabilities.
4. High-integrity inventory management.
5. Transaction-safe POS operations.
6. Scalable analytics and reporting.
7. AI-ready historical business data.
8. Auditability.
9. Future SaaS scalability.

---

# 2. Database Technology

## Primary Database

**PostgreSQL**

PostgreSQL is the primary source of truth for all transactional and business-critical data.

### ORM

**Prisma**

Prisma will be used for:

* Schema definition
* Migrations
* Type-safe database access
* Relationships
* Transactions
* Query construction
* Database client management

---

# 3. Database Design Principles

Buzzsynx follows these principles:

### Principle 1 — PostgreSQL is the source of truth

Redis must never become the authoritative source for:

* Inventory
* Sales
* Payments
* Purchases
* Customers
* Products
* Financial records

Redis is used for performance and temporary state.

---

### Principle 2 — Tenant isolation is mandatory

Every tenant-owned business record must be associated with a `tenantId`.

Example:

```text
Tenant
   │
   ├── Users
   ├── Products
   ├── Inventory
   ├── Suppliers
   ├── Customers
   ├── Purchases
   ├── Sales
   ├── Payments
   └── Reports
```

A user belonging to Tenant A must never access Tenant B's records.

---

### Principle 3 — Inventory is ledger-driven

Inventory should not depend only on a mutable quantity field.

Every meaningful stock change should generate an inventory movement.

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

This creates a historical audit trail.

---

### Principle 4 — Financial data is immutable where appropriate

Completed transactions should not be casually overwritten.

For example:

```text
Sale
SaleItem
Payment
Invoice
InventoryMovement
```

should preserve historical information.

Corrections should generally happen through:

* Returns
* Refunds
* Adjustments
* Credit notes
* Reversal transactions

rather than silently modifying historical records.

---

### Principle 5 — Industry capabilities should not duplicate the entire database

Buzzsynx should not create separate databases for:

```text
Pharmacy
Supermarket
Clothing
Restaurant
```

Instead:

```text
Shared Core
     +
Industry Capabilities
     +
Tenant Configuration
```

---

# 4. High-Level Entity Model

The major database domains are:

```text
Tenant
 ├── Users
 ├── Roles
 ├── Permissions
 ├── Business Settings
 ├── Industry Configuration
 │
 ├── Products
 │    ├── Categories
 │    ├── Brands
 │    ├── Variants
 │    └── Product Attributes
 │
 ├── Inventory
 │    ├── Stock
 │    ├── Stock Locations
 │    └── Inventory Movements
 │
 ├── Suppliers
 │    └── Purchases
 │         └── Purchase Items
 │
 ├── Customers
 │
 ├── POS
 │    └── Sales
 │         ├── Sale Items
 │         ├── Payments
 │         └── Returns
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

# 5. Tenant Model

The `Tenant` represents an individual business using Buzzsynx.

Example:

```text
Tenant 1
Name: Sri Lakshmi Pharmacy
Industry: PHARMACY

Tenant 2
Name: ABC Supermarket
Industry: SUPERMARKET

Tenant 3
Name: Fashion Hub
Industry: CLOTHING

Tenant 4
Name: Spice Garden
Industry: RESTAURANT
```

### Core fields

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

### Industry type

Possible values:

```text
PHARMACY
SUPERMARKET
CLOTHING
RESTAURANT
GENERAL_RETAIL
```

The architecture should allow additional industries later.

---

# 6. Tenant Isolation Model

Buzzsynx uses a **shared database / shared schema / tenantId isolation** strategy initially.

Conceptually:

```text
PostgreSQL
│
├── Tenant A records
├── Tenant B records
├── Tenant C records
└── Tenant D records
```

Every tenant-owned table should contain:

```text
tenantId
```

Example:

```text
Product
------
id
tenantId
name
sku
price
...
```

The backend must always scope queries by tenant.

Correct:

```javascript
prisma.product.findMany({
  where: {
    tenantId: currentTenantId
  }
});
```

Incorrect:

```javascript
prisma.product.findMany();
```

unless the operation is explicitly system-level.

---

# 7. Authentication and User Data

## User

Represents an individual Buzzsynx account.

Possible fields:

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

A user may belong to one or more tenants depending on the future account model.

---

## TenantUser

A membership model should be considered when users can work across multiple businesses.

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

This provides:

```text
User
  ↓
TenantUser
  ↓
Tenant
```

and allows the same account to potentially participate in multiple tenants.

---

# 8. Roles and Permissions

Buzzsynx uses RBAC.

Example roles:

```text
OWNER
ADMIN
MANAGER
CASHIER
INVENTORY_MANAGER
ACCOUNTANT
STAFF
```

The exact role set may evolve.

Core entities:

```text
Role
Permission
RolePermission
TenantUser
```

Relationship:

```text
TenantUser
    ↓
  Role
    ↓
RolePermission
    ↓
Permission
```

Permissions should be granular enough to control operations such as:

```text
product.create
product.update
product.delete

inventory.view
inventory.adjust
inventory.transfer

sale.create
sale.refund

purchase.create
purchase.receive

customer.view
customer.create

report.view

settings.manage
```

---

# 9. Business Configuration

Each tenant should have configurable business settings.

Possible configuration domains:

```text
Business Profile
Tax Settings
Currency
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
tenantId
currency
timezone
taxMode
invoicePrefix
lowStockThreshold
...
```

Configuration should be stored separately from transactional data.

---

# 10. Product Domain

The product domain is shared across most industries.

Core entities:

```text
Product
Category
Brand
ProductVariant
ProductAttribute
ProductImage
Unit
```

---

# 11. Product

A product represents the logical business item.

Example:

```text
Product
-------
Paracetamol 500mg
```

or:

```text
Product
-------
Men's T-Shirt
```

or:

```text
Product
-------
Basmati Rice
```

Potential fields:

```text
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

Not every industry will use every field.

---

# 12. Product Variants

Variants are especially important for clothing and other businesses where one logical product has multiple sellable variations.

Example:

```text
Product
Men's T-Shirt

Variants:
M / Black
M / White
L / Black
L / White
XL / Black
XL / White
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

The `attributes` structure may initially use JSON where appropriate.

Later, heavily-used attributes can be normalized if required.

---

# 13. Industry-Specific Product Data

The shared Product model should remain clean.

Industry-specific data should be represented through capability-specific models.

### Pharmacy

Possible entities:

```text
MedicineDetails
MedicineBatch
Expiry
Manufacturer
PrescriptionRequirement
```

Example:

```text
Medicine
--------
productId
genericName
strength
dosageForm
prescriptionRequired
manufacturerId
```

Batch information:

```text
MedicineBatch
-------------
id
tenantId
productId
batchNumber
expiryDate
mrp
purchasePrice
quantity
```

---

### Supermarket

Important attributes:

```text
barcode
unit
weight
bulkPricing
offer
```

---

### Clothing

Important attributes:

```text
size
color
material
variant
```

---

### Restaurant

Products may represent menu items while separate entities manage ingredients and recipes.

Possible entities:

```text
MenuItem
Ingredient
Recipe
RecipeItem
```

---

# 14. Category and Brand

Categories allow product organization.

Example:

```text
Electronics
Clothing
Groceries
Medicines
Beverages
```

Brands allow grouping by manufacturer or brand.

Both should normally be tenant-scoped.

---

# 15. Supplier Domain

Core entities:

```text
Supplier
Purchase
PurchaseItem
PurchasePayment
```

### Supplier

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

---

# 16. Purchase Domain

A purchase represents inventory acquired from a supplier.

Example:

```text
Purchase
--------
Supplier: ABC Distributors

Items:
Paracetamol × 100
Vitamin C × 50
```

Core fields:

```text
Purchase
--------
id
tenantId
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

---

# 17. Purchase Items

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

When a purchase is received, it should generate inventory movements.

```text
Purchase
   ↓
PurchaseItem
   ↓
InventoryMovement
   ↓
Stock
```

---

# 18. Inventory Domain

Inventory is one of the most critical domains in Buzzsynx.

Core entities:

```text
StockLocation
Stock
InventoryMovement
StockTransfer
StockAdjustment
```

---

# 19. Stock Location

A business may eventually have multiple locations.

Examples:

```text
Main Store
Warehouse
Branch 1
Branch 2
Cold Storage
```

Model:

```text
StockLocation
-------------
id
tenantId
name
type
address
status
createdAt
updatedAt
```

Even if version one supports only one location, designing around locations avoids painful restructuring later.

---

# 20. Stock

Stock represents the current calculated/maintained inventory state for a product at a location.

```text
Stock
-----
id
tenantId
locationId
productId
variantId
quantity
reservedQuantity
availableQuantity
updatedAt
```

A unique constraint should prevent duplicate stock records for the same:

```text
tenantId
locationId
productId
variantId
```

---

# 21. Inventory Movement Ledger

Every meaningful stock change should create an inventory movement.

```text
InventoryMovement
-----------------
id
tenantId
locationId
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
PURCHASE
SALE
SALE_RETURN
PURCHASE_RETURN
TRANSFER_IN
TRANSFER_OUT
DAMAGE
EXPIRY
ADJUSTMENT
OPENING_STOCK
```

Example:

```text
Purchase 100 units
        ↓
+100 PURCHASE

Sale 5 units
        ↓
-5 SALE

Damage 2 units
        ↓
-2 DAMAGE
```

This creates an auditable stock history.

---

# 22. Inventory Consistency

The conceptual inventory equation is:

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

The exact implementation may maintain a current stock table for performance while preserving the movement ledger as the historical record.

---

# 23. POS Domain

POS is responsible for fast transaction processing.

Core entities:

```text
Sale
SaleItem
Payment
Invoice
SaleReturn
SaleReturnItem
```

---

# 24. Sale

A sale represents a completed or in-progress customer transaction.

```text
Sale
----
id
tenantId
customerId
locationId
saleNumber
status
subtotal
discountAmount
taxAmount
totalAmount
createdBy
createdAt
updatedAt
```

Possible statuses:

```text
DRAFT
PENDING
COMPLETED
CANCELLED
REFUNDED
PARTIALLY_REFUNDED
```

---

# 25. Sale Items

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

Important historical rule:

The sale item should store the price used during the transaction.

It should not depend on the current product price.

For example:

```text
Product current price: ₹120

Historical sale:
unitPrice: ₹100
```

The historical transaction remains ₹100 even after the product price changes.

---

# 26. Payment Domain

A sale and payment should remain separate concepts.

One sale can potentially contain multiple payments.

Example:

```text
Sale Total = ₹1,000

Cash = ₹400
UPI = ₹600
```

Possible model:

```text
Payment
-------
id
tenantId
saleId
paymentMethod
amount
status
transactionReference
paidAt
createdAt
```

Payment methods:

```text
CASH
CARD
UPI
BANK_TRANSFER
WALLET
CREDIT
OTHER
```

---

# 27. Invoice Domain

Invoices should preserve transaction information.

```text
Invoice
-------
id
tenantId
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

Invoice numbering must be tenant-specific and configurable.

Example:

```text
INV-2026-000001
INV-2026-000002
```

---

# 28. Customer Domain

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

Customers may later support:

```text
Purchase history
Credit balance
Loyalty points
Preferences
Marketing consent
```

These should be added as separate capabilities where appropriate rather than making the base customer table excessively large.

---

# 29. Returns and Refunds

Returns should be represented as separate business events.

Avoid simply changing the original sale.

Example:

```text
Original Sale
     ↓
Return
     ↓
Inventory Movement
     ↓
Refund
```

Core entities:

```text
SaleReturn
SaleReturnItem
Refund
```

This preserves history.

---

# 30. Restaurant Capability

Restaurant functionality requires additional models.

Possible entities:

```text
Menu
MenuItem
Ingredient
Recipe
RecipeItem
DiningTable
Order
OrderItem
KitchenOrder
KitchenOrderItem
```

Conceptually:

```text
Menu Item
    ↓
Recipe
    ↓
Ingredients
    ↓
Inventory
```

Example:

```text
Chicken Biryani
      ↓
Rice
Chicken
Spices
Oil
      ↓
Inventory consumption
```

The restaurant capability should integrate with the shared inventory and sales systems.

---

# 31. Pharmacy Capability

Pharmacy-specific models may include:

```text
MedicineDetails
MedicineBatch
Manufacturer
Prescription
```

Important data:

```text
Batch Number
Expiry Date
MRP
Manufacturer
Prescription Requirement
```

Expiry-aware inventory is particularly important.

Example:

```text
Product
  ↓
Multiple Batches
  ↓
Expiry Dates
```

The system should support batch-aware stock deduction where required.

---

# 32. Analytics Data

Analytics should initially use transactional data from PostgreSQL.

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

The first implementation should avoid creating a completely separate analytics database unnecessarily.

Queries, indexes, aggregation tables, or scheduled summaries can be introduced as data volume grows.

---

# 33. AI Data Architecture

AI should use historical business data rather than directly modifying transactional data.

Conceptually:

```text
PostgreSQL
    ↓
Analytics / Aggregation
    ↓
AI Processing
    ↓
AI Insight
    ↓
Recommendation
    ↓
Business User
```

Possible AI-related entities:

```text
AIInsight
AIRecommendation
AIForecast
AIJob
```

Example:

```text
AIRecommendation
-----------------
id
tenantId
type
title
description
priority
data
status
createdAt
expiresAt
```

Examples:

```text
"Product X may run out within 5 days."

"Product Y has had no sales for 45 days."

"Weekend demand is increasing."

"Stock level appears unusually high."
```

AI should recommend actions, while transactional actions remain controlled by normal business workflows.

---

# 34. Notification Domain

Notifications may include:

```text
Notification
NotificationPreference
NotificationDelivery
```

Examples:

```text
Low stock
Expiry alert
Payment failure
AI recommendation
Purchase received
System alert
```

Notifications should be tenant-scoped when they relate to business activity.

---

# 35. Audit Log

Auditability is important for a business SaaS.

Core model:

```text
AuditLog
--------
id
tenantId
userId
action
entityType
entityId
metadata
ipAddress
userAgent
createdAt
```

Examples:

```text
PRODUCT_CREATED
PRODUCT_UPDATED
SALE_CREATED
SALE_REFUNDED
STOCK_ADJUSTED
USER_ROLE_CHANGED
SETTINGS_UPDATED
```

Audit logs should generally be append-only.

---

# 36. Soft Delete Strategy

Not every entity should be physically deleted.

For business-critical records, prefer statuses or soft deletion where appropriate.

Possible field:

```text
deletedAt
```

Examples:

```text
Product
Customer
Supplier
User
Category
```

Historical transactional records such as completed sales should generally remain available.

---

# 37. IDs

Database entities should use stable unique identifiers.

Recommended:

```text
UUID
```

or another strong generated identifier supported consistently across the application.

Human-readable numbers should be separate.

Example:

```text
Database ID:
550e8400-e29b-41d4-a716-446655440000

Sale Number:
SAL-2026-000182
```

Never use human-readable invoice/sale numbers as the primary database identity.

---

# 38. Timestamps

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

All timestamps should be handled consistently, preferably using UTC at the persistence layer.

Tenant-level timezone configuration should control business-facing date/time presentation.

---

# 39. Monetary Values

Money should not be stored using JavaScript floating-point arithmetic.

Database monetary values should use a suitable exact numeric/decimal representation.

Example:

```text
Decimal
```

This is important for:

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

Example:

```text
99.95
```

should not be represented using an imprecise binary floating-point value for financial calculations.

---

# 40. Transaction Management

Critical operations must use PostgreSQL transactions.

### POS sale

```text
BEGIN TRANSACTION

Validate cart
      ↓
Validate stock
      ↓
Create Sale
      ↓
Create Sale Items
      ↓
Create Payment
      ↓
Create Inventory Movements
      ↓
Update Stock
      ↓
Create Invoice
      ↓
Create Audit Log

COMMIT
```

If any critical operation fails:

```text
ROLLBACK
```

This prevents partial sales.

---

# 41. Purchase Receiving Transaction

```text
BEGIN

Validate Purchase
      ↓
Receive Purchase
      ↓
Create Purchase Items
      ↓
Create Inventory Movements
      ↓
Update Stock
      ↓
Update Purchase Status
      ↓
Audit Log

COMMIT
```

---

# 42. Concurrency and Stock Safety

Inventory operations must account for concurrent transactions.

Example:

```text
Stock = 5
```

Two POS terminals simultaneously attempt:

```text
Terminal A → buys 4
Terminal B → buys 3
```

The database must prevent both transactions from incorrectly assuming that five units are independently available.

Stock updates should therefore use appropriate transaction isolation, locking, or atomic update strategies.

This is a database integrity requirement, not merely a frontend concern.

---

# 43. Database Constraints

Important constraints include:

### Unique constraints

Examples:

```text
Tenant.slug
User.email
TenantUser(tenantId, userId)
Product(tenantId, sku)
ProductVariant(tenantId, sku)
Invoice(tenantId, invoiceNumber)
Sale(tenantId, saleNumber)
```

Uniqueness should generally be tenant-scoped where appropriate.

---

### Foreign keys

Relationships should use foreign keys wherever practical.

Example:

```text
SaleItem.saleId → Sale.id
SaleItem.productId → Product.id
PurchaseItem.purchaseId → Purchase.id
InventoryMovement.productId → Product.id
```

---

### Check constraints

Use database-level constraints where valuable.

Examples:

```text
quantity > 0
amount >= 0
price >= 0
taxRate >= 0
```

Application validation should still exist, but important integrity rules should not rely solely on frontend validation.

---

# 44. Indexing Strategy

Indexes should be designed around actual query patterns.

Common indexes:

```text
tenantId
tenantId + status
tenantId + createdAt
tenantId + sku
tenantId + barcode
tenantId + productId
tenantId + locationId
tenantId + customerId
tenantId + supplierId
```

Examples:

```text
Product
INDEX (tenantId, sku)

Sale
INDEX (tenantId, createdAt)

InventoryMovement
INDEX (tenantId, productId, createdAt)

Stock
INDEX (tenantId, locationId, productId)
```

The exact indexes should be finalized after query patterns are implemented and measured.

Avoid blindly indexing every column.

---

# 45. Search Strategy

For common POS searches, the database should support efficient lookup by:

```text
SKU
Barcode
Product Name
Variant SKU
```

Example:

```text
Barcode scan
     ↓
Product lookup
     ↓
Variant resolution
     ↓
Stock lookup
     ↓
POS cart
```

PostgreSQL indexing should handle the initial implementation.

A dedicated search engine should only be considered if actual scale and search requirements justify it.

---

# 46. Redis and Database Relationship

Redis is not the primary database.

Architecture:

```text
Application
    │
    ├── PostgreSQL → Source of Truth
    │
    └── Redis → Cache / Temporary State
```

Redis may cache:

```text
Product lookup
Dashboard summaries
Frequently accessed configuration
Rate-limit counters
Sessions where applicable
Temporary POS state
```

When cached data becomes stale, PostgreSQL remains authoritative.

---

# 47. Queue and Database Relationship

BullMQ jobs should store references to database records rather than large business payloads where practical.

Example:

```text
Queue Job
   ↓
tenantId
saleId
jobType
```

Worker:

```text
Job
 ↓
Fetch PostgreSQL data
 ↓
Process
 ↓
Store result
```

This reduces stale or duplicated job data.

---

# 48. AI Processing and Database Safety

AI workers must not directly bypass business rules.

Incorrect:

```text
AI
 ↓
Direct inventory update
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
Transaction
 ↓
Database
```

For example:

```text
AI recommends reorder
        ↓
User reviews
        ↓
Purchase workflow
        ↓
Purchase created
        ↓
Inventory received
```

AI should augment business operations rather than become an uncontrolled transactional layer.

---

# 49. Data Lifecycle

Buzzsynx should distinguish between:

### Operational data

Frequently accessed:

```text
Products
Stock
Customers
Active Sales
```

### Historical data

Long-term business records:

```text
Completed Sales
Payments
Invoices
Inventory Movements
Audit Logs
```

### Derived data

Calculated or generated:

```text
Analytics
Forecasts
AI Insights
Dashboard summaries
```

Derived data can be rebuilt from authoritative records where practical.

---

# 50. Data Retention

Retention requirements will depend on:

* Business requirements
* Applicable regulations
* Tenant plan
* Financial/accounting requirements
* Storage cost

The application should avoid permanently deleting important transactional history simply because a user removes an item from the UI.

Retention and deletion policies will be finalized before production launch.

---

# 51. Backup and Recovery

Production PostgreSQL must have:

```text
Automated backups
Point-in-time recovery where supported
Backup monitoring
Restore testing
Recovery documentation
```

Backups are not considered reliable until restoration has been tested.

---

# 52. Migration Strategy

All schema changes must use version-controlled Prisma migrations.

Example:

```text
prisma/
├── schema.prisma
└── migrations/
    ├── migration_001
    ├── migration_002
    └── migration_003
```

Production schema changes must never be performed manually without an appropriate migration process.

Migration principles:

1. Make changes backward-compatible where possible.
2. Test migrations locally.
3. Test migrations in staging.
4. Back up production before risky migrations.
5. Deploy application and schema changes in a controlled order.

---

# 53. Environment Separation

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

# 54. Seed Data

Prisma seed scripts should create development data such as:

```text
Demo Tenant
Demo Users
Roles
Permissions
Categories
Products
Customers
Suppliers
Sample Inventory
```

Industry demo tenants can be useful:

```text
Demo Pharmacy
Demo Supermarket
Demo Clothing Store
Demo Restaurant
```

This will make development and testing much easier.

---

# 55. Recommended Core Relationship Map

The main business relationship is:

```text
Tenant
 │
 ├── Users
 │
 ├── Products
 │    ├── Categories
 │    ├── Brands
 │    └── Variants
 │
 ├── Locations
 │    └── Stock
 │
 ├── Suppliers
 │    └── Purchases
 │         └── Purchase Items
 │
 ├── Customers
 │
 ├── Sales
 │    ├── Sale Items
 │    └── Payments
 │
 ├── Invoices
 │
 ├── Inventory Movements
 │
 ├── AI Insights
 │
 ├── Notifications
 │
 └── Audit Logs
```

Industry capabilities attach to this shared model.

---

# 56. Example: Pharmacy Data Flow

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
Inventory Movement
  ↓
Stock
  ↓
POS Sale
  ↓
Batch-aware Stock Deduction
  ↓
Payment
  ↓
Invoice
  ↓
Analytics
  ↓
Expiry / Reorder AI
```

---

# 57. Example: Clothing Data Flow

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
Stock
  ↓
POS
  ↓
Sale
  ↓
Payment
  ↓
Invoice
```

Each variant can maintain independent SKU and inventory.

---

# 58. Example: Restaurant Data Flow

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

This allows restaurant-specific workflows without creating an entirely separate platform.

---

# 59. Example: Supermarket Data Flow

```text
Tenant
  ↓
Product
  ↓
Barcode
  ↓
Stock
  ↓
Fast POS Scan
  ↓
Sale
  ↓
Payment
  ↓
Invoice
  ↓
Inventory Movement
```

The POS must optimize for fast product lookup and transaction completion.

---

# 60. Database Security

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

Application users should never receive direct database credentials.

Only backend services should communicate with PostgreSQL.

---

# 61. Application Database Access

Database access should be centralized through the backend architecture.

Preferred flow:

```text
Controller
    ↓
Service
    ↓
Repository / Prisma
    ↓
PostgreSQL
```

Business rules should not be scattered throughout controllers.

For example:

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

This keeps transactional logic maintainable.

---

# 62. Repository Responsibility

Repositories/data-access functions should primarily handle:

```text
Queries
Creates
Updates
Deletes
Transactions
Database-specific operations
```

Services should handle:

```text
Business rules
Validation orchestration
Workflow
Authorization context
Domain logic
```

This separation should remain consistent across modules.

---

# 63. Database Observability

Production database monitoring should track:

```text
CPU
Memory
Storage
Connections
Query latency
Slow queries
Locks
Deadlocks
Transaction failures
Backup status
```

AWS CloudWatch and PostgreSQL/RDS monitoring can be used when deployed on AWS.

Application-level database errors should also be captured by Sentry and structured logging.

---

# 64. Performance Strategy

Performance optimization should follow:

```text
Correctness
   ↓
Measurement
   ↓
Indexing
   ↓
Query optimization
   ↓
Caching
   ↓
Aggregation
   ↓
Scaling
```

Do not introduce complex infrastructure before identifying an actual bottleneck.

---

# 65. Future Database Scaling

The initial strategy is:

```text
One PostgreSQL database
        +
Tenant isolation
        +
Proper indexes
        +
Transactions
        +
Redis caching
```

As Buzzsynx grows, possible future strategies include:

```text
Read replicas
Partitioning
Analytics database
Warehouse
Tenant-specific databases
Database sharding
```

These are future scaling options, not initial requirements.

---

# 66. Multi-Tenant Evolution

Initial:

```text
Shared Database
Shared Schema
tenantId isolation
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

# 67. Database Design Rules

The following rules are mandatory:

### Rule 1

Every tenant-owned entity must have a clear tenant relationship.

### Rule 2

Never trust a client-provided `tenantId` for authorization.

### Rule 3

Every business-critical query must be tenant-scoped.

### Rule 4

Inventory changes must create inventory movements.

### Rule 5

Completed financial transactions should preserve historical values.

### Rule 6

Critical workflows must use database transactions.

### Rule 7

Money must use exact decimal representation.

### Rule 8

Foreign keys should be used to maintain relationships.

### Rule 9

Indexes should follow real query patterns.

### Rule 10

AI must not bypass business transaction services.

### Rule 11

Redis must never replace PostgreSQL as the source of truth.

### Rule 12

Schema changes must be version-controlled through migrations.

---

# 68. Database Implementation Order

The database should be implemented in phases.

## Phase 1 — Foundation

```text
Tenant
User
TenantUser
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
StockLocation
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
PurchasePayment
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

## Phase 7 — Industry Capabilities

```text
MedicineDetails
MedicineBatch

Clothing attributes

Restaurant Menu
Ingredient
Recipe
DiningTable
KitchenOrder
```

## Phase 8 — Intelligence

```text
Analytics summaries
AIInsight
AIRecommendation
AIForecast
```

## Phase 9 — Platform

```text
Notification
AuditLog
System configuration
Subscription/billing models
```

---

# 69. Prisma Schema Strategy

The Prisma schema should remain readable and modular even as it becomes large.

Recommended organization:

```text
prisma/
├── schema.prisma
├── seed.js
└── migrations/
```

Initially, a single `schema.prisma` file is acceptable.

As the project grows, schema organization should prioritize maintainability and Prisma compatibility rather than artificially splitting models too early.

---

# 70. Database Testing Requirements

Database tests should verify:

### Tenant isolation

```text
Tenant A cannot access Tenant B data.
```

### Inventory

```text
Purchase increases stock.
Sale decreases stock.
Return increases stock.
Damage decreases stock.
```

### POS transaction

```text
Sale + payment + inventory movement + invoice
```

must behave atomically.

### Concurrency

Concurrent sales must not create invalid stock states.

### Constraints

Duplicate:

```text
SKU
Barcode
Invoice Number
Sale Number
```

should be handled correctly according to tenant rules.

### Financial accuracy

```text
Subtotal
Tax
Discount
Total
Payment
Refund
```

must remain consistent.

---

# 71. Database Definition of Done

The database layer is considered ready for implementation when:

* [ ] Core entities are defined.
* [ ] Tenant relationships are defined.
* [ ] Foreign-key relationships are defined.
* [ ] Unique constraints are defined.
* [ ] Required indexes are identified.
* [ ] Inventory movement model is finalized.
* [ ] POS transaction model is finalized.
* [ ] Payment model is finalized.
* [ ] Invoice model is finalized.
* [ ] Return/refund model is finalized.
* [ ] Industry capability models are defined.
* [ ] Audit model is defined.
* [ ] Monetary fields use exact decimal types.
* [ ] Critical workflows have transaction boundaries.
* [ ] Prisma schema is validated.
* [ ] Prisma migrations are tested.
* [ ] Seed data is available.
* [ ] Tenant isolation tests are implemented.
* [ ] Database backup strategy is defined for staging/production.

---

# 72. Final Database Architecture

The final conceptual architecture is:

```text
                         BUZZSYNX
                            │
                         TENANT
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
        USERS            PRODUCTS          SETTINGS
          │                 │
        RBAC          ┌─────┼─────┐
                      │     │     │
                  CATEGORY BRAND VARIANTS
                            │
                         INVENTORY
                            │
                  ┌─────────┼─────────┐
                  │         │         │
                STOCK    MOVEMENTS  LOCATIONS
                  │
          ┌───────┴────────┐
          │                │
      PURCHASING          POS
          │                │
       SUPPLIER          SALES
          │                │
      PURCHASE        ┌────┼────┐
          │          ITEMS PAYMENT
          │                │
          └───────┬────────┘
                  │
               INVOICE
                  │
              ANALYTICS
                  │
              AI ENGINE
                  │
        ┌─────────┼─────────┐
        │         │         │
      INSIGHTS  FORECASTS  ALERTS

Industry capabilities connect to the shared core:

Pharmacy ────────┐
Supermarket ─────┤
Clothing ────────┤──→ Shared Business Engine
Restaurant ──────┘
```

The fundamental database principle is:

> **One shared business platform, strict tenant isolation, transactional integrity, ledger-based inventory, configurable industry capabilities, and historical data that can power analytics and AI.**

---
