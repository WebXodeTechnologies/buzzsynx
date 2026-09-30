# Buzzsynx — Industry Capabilities

**Document:** `docs/06-industry-capabilities.md`
**Project:** Buzzsynx
**Version:** 2.0
**Status:** Architecture Specification
**Architecture:** Multi-Tenant Modular Monolith
**Initial Industry:** Supermarket / Grocery Retail

---

# 1. Purpose

Buzzsynx is designed as a **multi-tenant, multi-industry business operations SaaS platform**.

The platform uses:

> **One shared business core + configurable capabilities + tenant-specific configuration + industry-specific modules where required.**

Buzzsynx should be capable of supporting different business types without creating a separate application or codebase for each industry.

Potential industries include:

* Supermarket / Grocery
* Pharmacy
* Clothing / Fashion
* Restaurant / Cafe
* Electronics
* Other retail and service businesses

However, these industries are **not equal MVP implementation commitments**.

### Current implementation strategy

**Supermarket / Grocery is the first complete industry implementation.**

Other industries are represented architecturally so that the platform can evolve without redesigning the shared core.

```text
BUZZSYNX

Shared Core
     │
     ├── Supermarket / Grocery ← Initial implementation
     │
     ├── Pharmacy             ← Future capability set
     ├── Clothing             ← Future capability set
     ├── Restaurant           ← Future capability set
     └── Other Industries     ← Future
```

### Core principle

> **Design for multiple industries. Build one industry completely first.**

---

# 2. Design Goals

The industry capability architecture must provide:

* Shared core business functionality
* Configurable capabilities
* Industry-specific workflows
* Tenant isolation
* Store/branch scope
* Reusable domain modules
* Clear business rules
* Controlled extensibility
* Simple tenant onboarding
* Minimal code duplication
* Ability to introduce future industries without rewriting the core

The architecture should avoid:

* Separate applications for each industry
* Separate codebases for each industry
* Large industry-specific `if/else` blocks
* Hardcoded industry logic inside generic services
* Duplicated product, inventory, POS and sales modules
* Frontend-controlled business permissions
* Premature implementation of every future industry

---

# 3. Architectural Model

Buzzsynx separates the platform into four conceptual layers:

```text
┌────────────────────────────────────────────┐
│              Tenant Context                │
│                                            │
│ Tenant / Business                          │
│ Industry Profile                           │
│ Enabled Capabilities                       │
│ Capability Configuration                   │
│ Active Store / Branch                      │
└──────────────────────┬─────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────┐
│          Industry Capabilities             │
│                                            │
│ Supermarket Capabilities                   │
│ Pharmacy Capabilities                      │
│ Clothing Capabilities                      │
│ Restaurant Capabilities                    │
│ Future Capabilities                        │
└──────────────────────┬─────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────┐
│              Shared Core                   │
│                                            │
│ Products                                   │
│ Inventory                                  │
│ Purchasing                                │
│ POS / Sales                                │
│ Customers                                  │
│ Payments                                   │
│ Suppliers                                  │
│ Reports                                    │
│ Analytics                                  │
│ Authentication / Authorization             │
└────────────────────────────────────────────┘
```

The shared core owns common business behavior.

Capabilities extend the core where a business requires additional behavior.

Industry configuration provides sensible defaults but does not replace authorization or business validation.

---

# 4. Tenant and Store Context

Industry capability resolution must operate inside a trusted tenant context.

Buzzsynx hierarchy:

```text
Super Admin
     │
     ▼
Tenant / Business
     │
     ▼
Store / Branch
     │
     ▼
Membership / User
```

A tenant may operate:

* One store
* Multiple stores
* Multiple branches

Capabilities belong to the tenant configuration, while some configuration and data may be scoped to a specific store.

Example:

```text
Tenant: ABC Supermarket

Industry:
SUPERMARKET

Capabilities:
- BARCODE
- WEIGHT_BASED_PRODUCTS
- LOW_STOCK_ALERTS
- OFFERS

Stores:
- Main Branch
- Town Branch
- Highway Branch
```

Capability checks must therefore consider:

```text
Authenticated User
       ↓
Tenant Membership
       ↓
Store Scope
       ↓
RBAC Permissions
       ↓
Capability Availability
       ↓
Business Rules
```

---

# 5. Shared Core

The shared core contains functionality that can be reused across multiple industries.

Core domains include:

* Authentication
* Tenant management
* Store / branch management
* Memberships
* RBAC
* Products
* Categories
* Brands
* Inventory
* Suppliers
* Purchasing
* POS
* Sales
* Customers
* Payments
* Invoices
* Reports
* Analytics
* Notifications
* Audit
* AI

The shared core should remain as industry-neutral as reasonably possible.

---

# 6. Authentication

Authentication is shared across all tenants and industries.

Capabilities include:

* Registration
* Login
* Logout
* Password management
* Account verification
* OAuth where enabled
* Session management
* Authentication recovery

Authentication answers:

> **Who is this user?**

It does not determine what the user can do.

Authorization and capability checks happen separately.

---

# 7. Tenant Management

Shared tenant functionality includes:

* Tenant creation
* Business profile
* Tenant status
* Industry selection
* Business settings
* Subscription information
* Capability configuration
* Store management

Tenant lifecycle:

```text
PENDING
   ↓
ACTIVE
   ↓
SUSPENDED
   ↓
ARCHIVED
```

Tenant status is controlled by the platform.

---

# 8. Store / Branch Management

Stores are part of the core architecture because Buzzsynx is designed for multi-branch businesses.

A store may contain:

* Name
* Address
* Contact information
* Tax/business information
* Operating status
* Store-specific configuration
* Staff assignments

Example:

```text
Tenant
│
├── Store A
│   ├── Cashiers
│   └── Inventory
│
├── Store B
│   ├── Cashiers
│   └── Inventory
│
└── Store C
    ├── Cashiers
    └── Inventory
```

Store-scoped operations must not expose data from another store unless the user's permissions explicitly allow cross-store access.

---

# 9. User Management and Memberships

A user belongs to a tenant through a membership.

Conceptually:

```text
User
  ↓
Tenant Membership
  ↓
Role / Permissions
  ↓
Store Scope
```

A user may have access to:

* One store
* Multiple stores
* All stores, depending on role and permissions

Roles should not be treated as global properties of the user.

---

# 10. RBAC

Current Buzzsynx platform roles are:

```text
SUPER_ADMIN
OWNER
ADMIN_MANAGER
CASHIER
ACCOUNTANT
STORE_STAFF
```

### Super Admin

Platform-level role.

Responsible for:

* Platform administration
* Tenant management
* Tenant status
* Platform configuration
* Operational oversight

### Owner

Tenant/business-level role.

Responsible for:

* Business configuration
* Store management
* Staff management
* Financial/business visibility
* Product and inventory management

### Admin / Manager

Operational management role.

### Cashier

Primarily responsible for:

* POS
* Sales
* Customer lookup
* Applicable returns

### Accountant

Primarily responsible for:

* Payments
* Financial records
* Reports
* Accounting-related operations

### Store Staff

Operational store activities according to assigned permissions.

---

# 11. Permission Model

Authorization should use permissions rather than relying only on role names.

Example:

```text
product.view
product.create
product.update

inventory.view
inventory.adjust
inventory.transfer

sale.create
sale.view
sale.return

payment.view
payment.create
payment.refund

report.view
```

Roles map to permissions.

Permissions are evaluated together with:

* Tenant scope
* Store scope
* Capability availability

Industry capability does not grant permission automatically.

---

# 12. Product Management

Products are part of the shared core.

Common product capabilities include:

* Product creation
* Product update
* Product archive
* Product search
* Categories
* Brands
* SKU
* Barcode
* Pricing
* Tax configuration
* Product status
* Images
* Product metadata

The product model should remain generic.

Industry-specific data should be represented through extensions or dedicated capability modules.

---

# 13. Product and Stockable Item Separation

The platform should distinguish between the **product definition** and the **stockable item** where necessary.

Example:

```text
Product
Classic T-Shirt

     ↓

Variants / Stockable Items

M / Black
M / White
L / Black
L / White
```

For a simple supermarket product:

```text
Product
Rice 5kg

     ↓

Stockable Item
Rice 5kg
```

This allows future industries to introduce variants without forcing every product to use complex variant structures.

---

# 14. Inventory

Inventory is a shared core domain.

Buzzsynx uses a **stock movement ledger** to maintain traceability.

Common movement categories include:

```text
PURCHASE
SALE
RETURN
TRANSFER
DAMAGE
EXPIRY
ADJUSTMENT
```

A stock movement should be treated as an immutable business record.

Corrections should generally be represented through new corrective movements rather than editing historical movements.

---

# 15. Inventory Balance

The stock ledger provides the historical movement trail.

The current stock balance may be maintained as a transactional balance for efficient reads.

Conceptually:

```text
Purchase +100
      ↓
Balance = 100

Sale -20
      ↓
Balance = 80

Damage -5
      ↓
Balance = 75
```

The important invariant is:

> **Stock-changing operations and the corresponding balance update must occur transactionally.**

Redis must not become the authoritative stock source.

POS stock validation must use authoritative transactional data.

---

# 16. Purchasing

Shared purchasing functionality may include:

* Suppliers
* Purchase documents
* Purchase items
* Receiving
* Purchase history
* Cost tracking
* Purchase returns
* Supplier payments where implemented

Purchase orders may be introduced where useful, but they are not required to be the foundation of every purchasing workflow.

The important inventory flow is:

```text
Supplier
   ↓
Purchase / Receiving
   ↓
Verify Quantity & Cost
   ↓
Inventory Transaction
   ↓
Stock Balance
```

Finalized purchasing records should not be freely editable.

Corrections should use controlled adjustment or reversal workflows.

---

# 17. POS

POS is a shared transaction engine.

Canonical workflow:

```text
Product Search / Barcode
        ↓
Cart
        ↓
Validate Product
        ↓
Validate Stock
        ↓
Calculate Price
        ↓
Calculate Discount / Tax
        ↓
Validate Payment State
        ↓
Create Sale
        ↓
Create Stock Movement
        ↓
Create Invoice Record
        ↓
Commit Transaction
```

The POS UI may differ by industry, but the transactional core should remain shared wherever possible.

---

# 18. POS Transaction Principles

The server is authoritative for:

* Product validity
* Price
* Discount rules
* Tax calculation
* Stock availability
* Sale totals
* Payment state
* Inventory movement
* Invoice numbering

The client must not be trusted to determine the final financial or inventory result.

External services such as:

* Payment gateways
* Email
* WhatsApp
* AI
* PDF generation

must not be allowed to compromise the core transaction.

---

# 19. Sales

Shared sales functionality includes:

* Sale transactions
* Sale items
* Discounts
* Taxes
* Returns
* Refunds
* Invoices
* Payment status
* Customer association
* Sales history

Customer association should be optional for workflows such as walk-in sales where applicable.

---

# 20. Payment Separation

Sale state and payment state should remain conceptually separate.

Example:

```text
SALE
  └── Payment
       ├── CASH
       ├── UPI
       ├── CARD
       └── CREDIT
```

Split payments may contain multiple payment allocations.

External payment providers must be confirmed through trusted provider responses/webhooks rather than frontend success callbacks.

---

# 21. Industry Capability Model

Each tenant has an industry profile and enabled capabilities.

Conceptually:

```text
Tenant
│
├── Industry
│
└── Capabilities
      ├── Capability A
      ├── Capability B
      └── Capability C
```

Example:

```js
{
  industry: "SUPERMARKET",
  capabilities: [
    "BARCODE",
    "WEIGHT_BASED_PRODUCTS",
    "LOW_STOCK_ALERTS",
    "OFFERS"
  ]
}
```

Industry provides default capability recommendations.

The actual enabled capabilities belong to the tenant configuration.

---

# 22. Capability vs Industry

Industry and capability are different concepts.

Example:

```text
Industry:
PHARMACY

Capabilities:
BATCH_TRACKING
EXPIRY_TRACKING
PRESCRIPTION_WORKFLOW
```

Another pharmacy may not require every capability.

Therefore:

> **Industry determines the default capability profile. Tenant configuration determines which capabilities are enabled.**

However, capability enablement must also respect:

* Product availability
* Subscription/plan rules where applicable
* Capability dependencies
* Platform restrictions
* Tenant configuration permissions

---

# 23. Capability Categories

Capabilities can be grouped by domain.

### Product

```text
PRODUCT_VARIANTS
BARCODE
BATCH_TRACKING
SERIAL_TRACKING
UNIT_MANAGEMENT
```

### Inventory

```text
EXPIRY_TRACKING
STOCK_TRANSFER
MULTI_STORE_INVENTORY
RECIPE_CONSUMPTION
LOW_STOCK_ALERTS
```

### POS

```text
FAST_CHECKOUT
WEIGHT_BASED_PRODUCTS
TABLE_ORDERING
PRESCRIPTION_VALIDATION
```

### Sales

```text
RETURNS
REFUNDS
DISCOUNTS
OFFERS
CUSTOMER_CREDIT
```

### Operations

```text
KITCHEN_WORKFLOW
WARRANTY_TRACKING
REPAIR_WORKFLOW
```

Not every capability needs to be implemented in the initial product.

---

# 24. Capability Resolution

Every protected request should resolve trusted context.

Conceptually:

```js
const context = {
  tenantId,
  userId,
  membership,
  storeId,
  permissions,
  capabilities
};
```

The backend determines this context.

A service may then check:

```text
Is capability enabled?
        ↓
Does user have permission?
        ↓
Does user have store access?
        ↓
Are business rules satisfied?
```

The frontend must never be the authority for capability access.

---

# 25. Capability Middleware

Capability checks may be implemented at the API boundary where appropriate.

Example:

```js
requireCapability("BATCH_TRACKING")
```

Request flow:

```text
Authentication
      ↓
Tenant Resolution
      ↓
Membership
      ↓
Store Scope
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
```

Capability middleware should not replace service-level business validation.

Critical business rules must remain enforced inside the application/domain layer.

---

# 26. Avoid Industry-Specific Conditionals

Avoid spreading logic such as:

```js
if (industry === "PHARMACY") {
   ...
}

if (industry === "RESTAURANT") {
   ...
}
```

through generic services.

Prefer:

```text
Shared Core
     │
     ├── Variant Capability
     ├── Batch Capability
     ├── Recipe Capability
     └── Serial Capability
```

Industry modules can compose these capabilities.

Some industry-specific orchestration may still require explicit configuration or strategy selection. The goal is to prevent industry knowledge from leaking throughout unrelated modules.

---

# 27. Capability Module Structure

A conceptual structure may be:

```text
src/server/modules/

├── products/
├── inventory/
├── purchasing/
├── pos/
├── sales/
├── customers/
├── payments/
├── analytics/
├── reports/
│
└── capabilities/
    ├── common/
    ├── supermarket/
    ├── pharmacy/
    ├── clothing/
    └── restaurant/
```

The exact folder structure may evolve.

The important rule is:

> **Architectural boundaries matter more than folder names.**

---

# 28. Initial Supermarket Capability Set

Supermarket / Grocery is the first complete industry implementation.

Initial capabilities may include:

```text
BARCODE
UNIT_MANAGEMENT
WEIGHT_BASED_PRODUCTS
LOW_STOCK_ALERTS
OFFERS
DISCOUNTS
FAST_POS
SUPPLIER_MANAGEMENT
```

The exact MVP capability list should remain aligned with the product feature checklist.

Do not implement every theoretical supermarket feature before validating the core workflow.

---

# 29. Supermarket Workflow

Canonical supermarket workflow:

```text
Tenant Onboarding
      ↓
Store Setup
      ↓
Product Setup
      ↓
Supplier Setup
      ↓
Purchase / Receiving
      ↓
Inventory
      ↓
POS
      ↓
Sale
      ↓
Payment
      ↓
Invoice
      ↓
Stock Movement
      ↓
Analytics
      ↓
AI Insights
```

This is the first complete vertical slice of Buzzsynx.

---

# 30. Supermarket Capability — Barcode

Barcode scanning is a core supermarket capability.

Workflow:

```text
Scan Barcode
      ↓
Resolve Product / Stockable Item
      ↓
Validate Store Availability
      ↓
Resolve Price
      ↓
Validate Stock
      ↓
Add to Cart
```

Barcode lookup should be scoped appropriately to the tenant/store context.

---

# 31. Supermarket Capability — Units

Supermarket products may use different units.

Examples:

```text
PIECE
KG
GRAM
LITER
ML
PACK
BOX
DOZEN
```

Unit management must be configurable rather than hardcoded around one industry-specific assumption.

Where conversions are supported, conversion rules must be explicit and validated.

---

# 32. Supermarket Capability — Offers

Offers may include:

```text
Buy 1 Get 1
Buy 2 Get 1
Percentage Discount
Fixed Discount
Bundle Pricing
Member Pricing
```

Offers should be represented as configurable pricing rules.

Pricing rules must be evaluated by the backend.

The frontend must not calculate the authoritative final price.

---

# 33. Future Pharmacy Capability

Pharmacy is a **future industry capability set**, not part of the initial supermarket MVP.

Potential pharmacy capabilities include:

```text
MEDICINE_INFORMATION
BATCH_TRACKING
EXPIRY_TRACKING
PRESCRIPTION_WORKFLOW
MANUFACTURER_DATA
GENERIC_MEDICINE
DOSAGE_INFORMATION
```

The exact implementation must be designed according to the applicable jurisdiction and business requirements.

---

# 34. Pharmacy — Batch Tracking

A pharmacy product may contain multiple batches.

Example:

```text
Paracetamol 500mg

Batch A
Expiry: 2027-01
Stock: 100

Batch B
Expiry: 2028-03
Stock: 250
```

Batch information should be represented as dedicated inventory/business data rather than generic nullable product fields.

---

# 35. Pharmacy — Expiry Management

Potential workflow:

```text
Expiry Monitoring
       ↓
Identify Affected Batch
       ↓
Generate Alert
       ↓
Staff Review
       ↓
Inventory Action
```

Expired inventory must not automatically become sellable inventory.

Expiry-related inventory changes must create traceable inventory movements.

---

# 36. Pharmacy — Prescription Workflow

Some products may require additional validation.

Conceptual workflow:

```text
Product Requires Prescription
        ↓
POS Requests Required Information
        ↓
Authorized Staff Verification
        ↓
Business / Regulatory Validation
        ↓
Sale Permitted or Rejected
```

The actual workflow must be configurable according to jurisdiction and applicable regulations.

---

# 37. Future Clothing Capability

Clothing is a future industry capability set.

Potential capabilities include:

```text
PRODUCT_VARIANTS
SIZE
COLOR
MATERIAL
STYLE
VARIANT_SKU
VARIANT_BARCODE
VARIANT_INVENTORY
```

The shared product and inventory engines should remain reusable.

---

# 38. Clothing — Variant Model

Example:

```text
Classic T-Shirt

Variants:

S / Black
S / White
M / Black
M / White
L / Black
L / White
XL / Black
XL / White
```

Each stockable variant may have:

* SKU
* Barcode
* Price
* Stock
* Images
* Attributes

The generic inventory engine operates against the correct stockable item.

---

# 39. Future Restaurant Capability

Restaurant functionality is a future capability set.

Restaurants introduce a different operational model involving:

```text
Menu Items
Ingredients
Recipes
Tables
Orders
Kitchen Operations
```

These should be implemented as restaurant capabilities rather than forcing restaurant-specific behavior into the generic POS engine.

---

# 40. Restaurant — Menu and Recipe Management

A menu item may contain:

```text
Selling Price
Category
Availability
Ingredients
Recipe
Preparation Time
Tax Configuration
```

Example:

```text
Burger
   ↓
Recipe
   ├── Bun
   ├── Patty
   ├── Cheese
   ├── Lettuce
   └── Sauce
```

---

# 41. Restaurant — Ingredient Consumption

When a menu item is sold:

```text
Menu Sale
     ↓
Recipe Resolution
     ↓
Ingredient Consumption
     ↓
Inventory Movements
```

Example:

```text
1 Burger Sold

Bun       -1
Patty     -1
Cheese    -1
Lettuce   -1
Sauce     -20ml
```

The inventory ledger remains shared.

The restaurant capability determines how the sale translates into ingredient consumption.

---

# 42. Restaurant — Tables and Kitchen

Potential restaurant capabilities:

```text
TABLE_MANAGEMENT
KITCHEN_WORKFLOW
DINE_IN
TAKEAWAY
DELIVERY
```

Table states may include:

```text
AVAILABLE
OCCUPIED
RESERVED
CLEANING
```

Kitchen workflow may include:

```text
ORDER_CREATED
QUEUED
PREPARING
READY
SERVED
COMPLETED
```

These are future capabilities, not initial supermarket requirements.

---

# 43. Industry Extensions and Data Model

Shared entities may include:

```text
Tenant
User
Membership
Store
Product
Category
Brand
Supplier
Inventory
StockMovement
Customer
Sale
SaleItem
Payment
Invoice
```

Industry-specific entities may include:

```text
MedicineBatch
Prescription
ProductVariant
Recipe
RecipeIngredient
RestaurantTable
KitchenOrder
Warranty
SerialNumber
Repair
```

Industry extensions should only be introduced when the business capability genuinely requires them.

---

# 44. Tenant Isolation

Every tenant-owned industry entity must be tenant-scoped.

Conceptually:

```text
tenantId
```

Store-owned records should additionally have appropriate store scope.

Example:

```text
tenantId
storeId
```

The backend must verify:

```text
Authenticated User
       ↓
Tenant Membership
       ↓
Store Access
       ↓
Resource Ownership
```

A valid resource ID from another tenant must never expose another tenant's data.

---

# 45. Product Extensibility

Do not create a generic product table containing unrelated industry fields such as:

```text
medicineExpiry
shirtSize
shirtColor
recipeId
prescriptionRequired
tableNumber
```

Instead:

```text
Product
 ├── Common Fields
 │
 └── Capability Extension
      ├── Pharmacy
      ├── Clothing
      ├── Restaurant
      └── Other
```

This keeps the shared model maintainable.

However, extensibility should not automatically mean a highly generic EAV/JSON schema for everything.

Use proper relational entities when an industry capability has:

* Relationships
* Transactions
* Constraints
* Queries
* Reporting requirements
* Significant business rules

---

# 46. Frontend Capability Handling

The frontend may use capability configuration to control:

* Navigation
* Pages
* Widgets
* Forms
* Actions
* Industry-specific UI

Example:

```text
Common:

Dashboard
Inventory
POS
Sales
Customers
Reports
AI

Supermarket:

Barcode
Offers
Units
```

Future examples:

```text
Pharmacy:

Batches
Expiry
Prescriptions

Restaurant:

Tables
Kitchen
Recipes
```

Frontend capability visibility is strictly a UX concern.

> **Frontend configuration is never the security authority.**

---

# 47. Dashboard Customization

The dashboard shell remains shared.

Widgets and metrics can change according to:

* Industry
* Enabled capabilities
* Store
* User permissions
* Available data

### Supermarket

```text
Today's Sales
Fast Moving Products
Low Stock
Top Categories
Offers
```

### Future Pharmacy

```text
Today's Sales
Low Stock
Expiring Stock
Medicine Performance
Purchase Summary
```

### Future Clothing

```text
Today's Sales
Top Variants
Size Performance
Color Performance
Low Stock
```

### Future Restaurant

```text
Today's Orders
Revenue
Kitchen Queue
Top Menu Items
Ingredient Alerts
Table Status
```

---

# 48. Industry-Aware Analytics

Analytics should use a shared analytics architecture with industry-specific metrics.

Common metrics may include:

```text
Revenue
Orders
Average Order Value
Customer Count
Product Performance
Inventory Value
```

Industry-specific metrics can be added through capability modules.

Examples:

### Supermarket

```text
Fast Moving SKUs
Category Sales
Offer Performance
```

### Future Pharmacy

```text
Expiring Stock Value
Medicine Sales
Batch Performance
```

### Future Clothing

```text
Size Performance
Color Performance
Variant Sales
```

### Future Restaurant

```text
Menu Performance
Ingredient Usage
Table Turnover
Preparation Time
```

Analytics are read/derived data.

They must not directly mutate transactional business records.

---

# 49. AI Industry Awareness

Buzzsynx AI should operate using trusted tenant and store context.

Example:

```text
Tenant
Industry = SUPERMARKET

Capabilities
BARCODE
LOW_STOCK_ALERTS
OFFERS
```

AI can use relevant business data to produce insights such as:

```text
Low-stock analysis
Sales trends
Dead-stock detection
Reorder recommendations
Product performance
```

Future pharmacy capabilities may support:

```text
Expiry analysis
Batch analysis
Medicine demand trends
```

Future restaurant capabilities may support:

```text
Ingredient demand
Waste analysis
Menu performance
Recipe consumption
```

AI must:

* Respect tenant isolation
* Respect store scope
* Respect user permissions
* Use approved/derived business data
* Remain explainable where practical
* Not directly mutate critical financial or inventory records
* Not replace deterministic business rules

---

# 50. Capability Configuration

Capabilities may contain configuration.

Example:

```js
{
  name: "LOW_STOCK_ALERTS",
  enabled: true,
  configuration: {
    defaultThreshold: 10
  }
}
```

Another example:

```js
{
  name: "EXPIRY_TRACKING",
  enabled: true,
  configuration: {
    alertDays: [30, 15, 7]
  }
}
```

Configuration must be validated by the backend.

A frontend must not be able to enable restricted capabilities simply by modifying a request.

---

# 51. Capability Dependencies

Capabilities may depend on other capabilities.

Example:

```text
EXPIRY_TRACKING
       ↓
BATCH_TRACKING
```

or:

```text
RECIPE_CONSUMPTION
       ↓
INGREDIENT_INVENTORY
```

The platform should validate dependencies before enabling a capability.

Example:

```text
Cannot enable RECIPE_CONSUMPTION

without:

INGREDIENT_INVENTORY
```

Dependency rules should be defined centrally rather than scattered throughout application code.

---

# 52. Capability Lifecycle

Capabilities may have lifecycle states:

```text
AVAILABLE
ENABLED
DISABLED
DEPRECATED
```

Example:

```text
EXPIRY_TRACKING
       ↓
AVAILABLE
       ↓
ENABLED
       ↓
DISABLED
```

A deprecated capability must not be silently removed when existing tenant data depends on it.

Migration or retirement procedures should be defined before permanently removing a capability.

---

# 53. Feature Flags vs Capabilities

Feature flags and business capabilities are different concepts.

### Capability

Represents a business function.

```text
BATCH_TRACKING
```

### Feature Flag

Controls product rollout or experimental behavior.

```text
NEW_POS_UI
```

Therefore:

> **Capability = business/domain functionality.**

> **Feature flag = application rollout mechanism.**

They should not be treated as interchangeable.

---

# 54. Cross-Industry Capabilities

Some functionality may eventually apply to multiple industries.

Examples:

```text
LOYALTY
MULTI_STORE
ONLINE_ORDERING
DELIVERY
STAFF_ATTENDANCE
CUSTOMER_CREDIT
SUBSCRIPTIONS
```

These should become reusable capabilities rather than permanently belonging to one industry.

Example:

```text
SUPERMARKET
      +
ONLINE_ORDERING

RESTAURANT
      +
ONLINE_ORDERING

CLOTHING
      +
ONLINE_ORDERING
```

---

# 55. Adding a New Industry

When adding a new industry, follow a controlled process.

### Step 1 — Identify shared functionality

Reuse existing core modules:

```text
Products
Inventory
Purchasing
Suppliers
POS
Sales
Customers
Payments
Reports
Analytics
```

### Step 2 — Identify unique requirements

Example electronics store:

```text
Serial Numbers
Warranty
IMEI
Repairs
```

### Step 3 — Define capabilities

```text
SERIAL_TRACKING
WARRANTY_TRACKING
IMEI_TRACKING
REPAIR_WORKFLOW
```

### Step 4 — Implement industry-specific modules

```text
industry/electronics/

serials/
warranty/
repairs/
```

### Step 5 — Define default configuration

```text
ELECTRONICS

SERIAL_TRACKING
WARRANTY_TRACKING
IMEI_TRACKING
REPAIR_WORKFLOW
```

### Step 6 — Validate tenant and store isolation

Verify that:

* Data remains tenant-scoped
* Store scope is respected
* RBAC works
* Capability checks work
* Existing industries remain unaffected

---

# 56. Business Rule Extensions

Industry capabilities may extend business workflows, but core invariants must remain protected.

Example:

```text
Sale
 ↓
Stock Validation
 ↓
Payment Validation
 ↓
Transactional Sale
 ↓
Stock Movement
```

A pharmacy capability may add:

```text
Prescription Validation
Batch Validation
Expiry Validation
```

A clothing capability may add:

```text
Variant Validation
Size / Color Selection
```

A restaurant capability may add:

```text
Table Validation
Kitchen Order Creation
Recipe Consumption
```

The shared transaction boundary must remain reliable.

---

# 57. Transaction Boundaries

Industry capabilities must not weaken transactional integrity.

For critical operations:

```text
Validate
   ↓
Begin Transaction
   ↓
Apply Business Changes
   ↓
Create Stock / Financial Records
   ↓
Create Audit Records Where Required
   ↓
Commit
```

External operations should not be performed inside the core database transaction.

Avoid holding a database transaction open while calling:

* Payment providers
* AI providers
* Email services
* WhatsApp
* Storage services
* Other external APIs

Post-commit work should use background jobs where appropriate.

---

# 58. Async Industry Operations

Non-critical operations should execute asynchronously.

Example:

```text
Sale Committed
      ↓
Post-Commit Event / Queue
      ↓
BullMQ Worker
      ↓
Analytics
Notifications
Invoice PDF
AI Processing
```

Failure of an asynchronous operation should not roll back a successfully committed sale.

Jobs should be designed for:

* Retry
* Idempotency
* Failure handling
* Observability

---

# 59. Security

Industry capabilities follow the same security architecture as the core platform.

Request flow:

```text
Authentication
      ↓
Tenant Membership
      ↓
Tenant Context
      ↓
Store Scope
      ↓
RBAC
      ↓
Capability Check
      ↓
Validation
      ↓
Business Logic
```

A user must not be able to:

* Access another tenant's capability data
* Access another store without permission
* Enable restricted capabilities
* Modify capability configuration without authorization
* Bypass capability checks through direct API requests

---

# 60. Audit

Capability changes and important industry-specific business actions should be auditable where appropriate.

Examples:

```text
Capability Enabled
Capability Disabled
Capability Configuration Changed
Batch Adjusted
Expiry Stock Marked
Recipe Configuration Changed
Variant Updated
```

Audit records should capture relevant information such as:

```text
tenantId
storeId
userId
action
entity
entityId
timestamp
reason
requestId
```

Audit logs are separate from technical application logs.

---

# 61. Testing Strategy

Every implemented capability should be tested according to its risk and complexity.

### Unit Tests

Examples:

```text
Expiry calculation
Variant validation
Recipe consumption
Offer calculation
Capability dependency validation
```

### Integration Tests

Test interaction with:

```text
PostgreSQL
Redis where relevant
Core modules
Capability modules
```

### API Tests

Test:

```text
Authentication
RBAC
Capability access
Tenant isolation
Store isolation
Validation
Idempotency
```

### End-to-End Tests

Test complete workflows.

Initial supermarket example:

```text
Create Product
      ↓
Receive Stock
      ↓
Open POS
      ↓
Scan Product
      ↓
Create Sale
      ↓
Process Payment
      ↓
Update Inventory
      ↓
Generate Invoice
      ↓
View Analytics
```

Future industries should receive their own E2E workflows when implemented.

---

# 62. Industry Testing Matrix

The following matrix represents the **architecture target**, not a statement that every capability is already implemented.

| Capability      |     Supermarket | Pharmacy | Clothing | Restaurant |
| --------------- | --------------: | -------: | -------: | ---------: |
| Products        |               ✓ |   Future |   Future |     Future |
| Inventory       |               ✓ |   Future |   Future |     Future |
| POS             |               ✓ |   Future |   Future |     Future |
| Sales           |               ✓ |   Future |   Future |     Future |
| Suppliers       |               ✓ |   Future |   Future |     Future |
| Customers       |               ✓ |   Future |   Future |     Future |
| Barcode         |               ✓ |   Future |   Future |     Future |
| Unit Management |               ✓ |   Future |   Future |     Future |
| Batch Tracking  | Future/Optional |   Future |        — |          — |
| Expiry Tracking | Future/Optional |   Future |        — |   Optional |
| Prescription    |               — |   Future |        — |          — |
| Variants        |        Optional | Optional |   Future |   Optional |
| Offers          |               ✓ |   Future |   Future |     Future |
| Ingredients     |               — |        — |        — |     Future |
| Recipes         |               — |        — |        — |     Future |
| Tables          |               — |        — |        — |     Future |
| Kitchen         |               — |        — |        — |     Future |

**Important:** “Future” means architecturally planned, not currently implemented.

---

# 63. Initial Implementation Priority

Buzzsynx should follow this implementation order:

```text
PHASE 1
Shared Platform Core
        ↓
PHASE 2
Supermarket Product + Inventory
        ↓
PHASE 3
Supermarket Purchasing
        ↓
PHASE 4
Supermarket POS + Sales
        ↓
PHASE 5
Payments + Invoices + Returns
        ↓
PHASE 6
Analytics + Reports
        ↓
PHASE 7
AI Intelligence
        ↓
PHASE 8
Production Hardening
        ↓
PHASE 9
Future Industry Capabilities
```

The objective is to complete one coherent business workflow before expanding horizontally into additional industries.

---

# 64. Definition of Done

The industry capability architecture is considered structurally complete when:

* [ ] Shared core modules are defined
* [ ] Supermarket is identified as the first complete industry
* [ ] Future industries are separated from MVP scope
* [ ] Tenant industry is stored in trusted backend configuration
* [ ] Capabilities are configurable per tenant
* [ ] Store scope is supported
* [ ] Backend capability checks are implemented
* [ ] RBAC and capability checks work together
* [ ] Frontend reflects enabled capabilities
* [ ] Industry-specific data is tenant-scoped
* [ ] Store-specific data is store-scoped where required
* [ ] Capability dependencies are validated
* [ ] Critical business rules remain server-authoritative
* [ ] Industry-specific workflows have appropriate tests
* [ ] AI respects tenant, store and capability context
* [ ] Analytics supports industry-specific metrics
* [ ] New capabilities can be introduced without rewriting the shared core
* [ ] No widespread industry-specific conditional logic exists
* [ ] Documentation exists for implemented capabilities

---

# 65. Architectural Principle

Buzzsynx should not be designed as:

```text
Pharmacy App
Supermarket App
Clothing App
Restaurant App
```

It should be designed as:

```text
                     BUZZSYNX
                        │
              ┌─────────┴─────────┐
              │                   │
         Shared Core        Capabilities
              │                   │
              │          ┌────────┼────────┐
              │          │        │        │
              │     Supermarket Pharmacy Clothing
              │          │        │        │
              │          └────────┼────────┘
              │                   │
              └──────────┬────────┘
                         │
                      Tenants
                         │
                    Stores / Branches
```

The shared core remains stable.

Capabilities evolve independently.

Tenants activate the functionality relevant to their business.

---

# 66. Final Principle

> **Buzzsynx is a capability-driven, multi-tenant business operations SaaS platform designed for multiple industries but implemented incrementally.**

The platform provides a shared business engine for:

```text
Products
Inventory
Purchasing
POS
Sales
Customers
Payments
Invoices
Analytics
AI
```

Industry-specific requirements are implemented as modular capabilities.

The first complete implementation is:

```text
Supermarket / Grocery
```

Future capability sets may include:

```text
Pharmacy
Clothing
Restaurant
Electronics
Other Industries
```

The architecture therefore follows:

```text
One Codebase
      ↓
One Shared Core
      ↓
Tenant + Store Context
      ↓
Configurable Capabilities
      ↓
Industry-Specific Extensions
      ↓
Multiple Independent Tenants
```

while preserving:

* Maintainability
* Tenant isolation
* Store isolation
* Security
* Transactional integrity
* Extensibility
* AI safety
* Future scalability

---

# 67. Source-of-Truth Principle

The industry capability system must follow the broader Buzzsynx architecture principle:

> **The database knows what happened.
> The application enforces what is allowed.
> Capabilities determine which business functions are available.
> AI helps understand what happened and what might happen next.**

AI, frontend configuration, feature flags, or industry labels must never override authoritative business rules.

---

# 68. Closing Architecture Statement

Buzzsynx does not need to build four different products to become a multi-industry platform.

It needs to build **one strong business engine**, prove it through the supermarket vertical, and then extend that engine through carefully isolated capabilities.

> **Build one industry completely. Design the platform for many.**
