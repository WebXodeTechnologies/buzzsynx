# Buzzsynx — Security Architecture

**Document:** `docs/07-security.md`
**Project:** Buzzsynx
**Version:** 2.0
**Status:** Architecture Specification
**Architecture:** Multi-Tenant Modular Monolith
**Initial Industry:** Supermarket / Grocery Retail

---

# 1. Purpose

Security is a first-class architectural requirement of Buzzsynx.

Buzzsynx is a multi-tenant SaaS platform where multiple independent businesses use the same application and infrastructure.

The security architecture must protect:

* User accounts
* Tenant data
* Store / branch data
* Business transactions
* Inventory
* Payments
* Customer information
* AI processing
* Uploaded files
* APIs
* Background jobs
* Database
* Infrastructure
* Logs
* Secrets

The primary security objective is:

> **A user, tenant, service, or process must only access resources it is explicitly authorized to access.**

The security architecture must also assume that:

* Client requests can be manipulated.
* Authenticated users can be malicious.
* Accounts can be compromised.
* Background jobs can fail or be replayed.
* External providers can send duplicate or invalid callbacks.
* Internal services can contain implementation bugs.
* Infrastructure credentials can be compromised.

Security therefore follows a **defense-in-depth** model.

---

# 2. Security Principles

Buzzsynx follows these principles:

1. **Never trust client input.**
2. **Never trust client-supplied tenant identity.**
3. **Validate store context before using it.**
4. **Authenticate before accessing protected resources.**
5. **Authorize every protected operation.**
6. **Apply least privilege.**
7. **Use defense in depth.**
8. **Secure by default.**
9. **Validate at system boundaries.**
10. **Keep business invariants server-authoritative.**
11. **Protect sensitive data.**
12. **Audit important business actions.**
13. **Treat background jobs as security-sensitive.**
14. **Keep secrets outside source code.**
15. **Do not expose internal implementation details.**
16. **Make security controls testable.**
17. **Minimize collected and processed data.**
18. **Fail securely.**
19. **Keep tenant and store boundaries explicit.**
20. **Assume external integrations can be manipulated or replayed.**

---

# 3. Security Architecture

The general request security flow is:

```text
Client
  ↓
HTTPS
  ↓
Reverse Proxy / Nginx / Load Balancer
  ↓
Rate Limiting
  ↓
Authentication
  ↓
User Resolution
  ↓
Tenant Membership
  ↓
Tenant Context
  ↓
Store Scope Resolution
  ↓
RBAC / Permissions
  ↓
Capability Check
  ↓
Input Validation
  ↓
Business Rules
  ↓
Tenant + Store Scoped Data Access
  ↓
PostgreSQL
```

Supporting security layers include:

```text
Secrets Management
Audit Logging
Security Monitoring
Redis Protection
Queue Security
File Storage Security
Payment Verification
Webhook Verification
AI Isolation
Infrastructure Security
Backup Security
```

Security must not depend on one middleware or one authorization check.

---

# 4. Security Boundaries

Buzzsynx has several important security boundaries.

### 4.1 Identity boundary

Determines:

> Who is the authenticated user?

### 4.2 Tenant boundary

Determines:

> Which business does this request operate within?

### 4.3 Store boundary

Determines:

> Which store / branch can this user access?

### 4.4 Permission boundary

Determines:

> What actions can this user perform?

### 4.5 Capability boundary

Determines:

> Is the required business functionality enabled for this tenant?

### 4.6 Data boundary

Determines:

> Does the requested resource belong to the authorized tenant and store scope?

### 4.7 Infrastructure boundary

Protects:

* Database
* Redis
* Queues
* Storage
* Containers
* AWS resources
* Secrets

---

# 5. Threat Model

Buzzsynx should consider threats from several sources.

## 5.1 External attackers

Potential attacks include:

* Credential stuffing
* Brute-force attacks
* Account takeover
* API abuse
* Injection
* IDOR
* Session theft
* Malicious file uploads
* Payment manipulation
* Webhook replay
* Scraping
* Automated attacks
* Denial-of-service attempts

## 5.2 Malicious authenticated users

A legitimate user may attempt to:

* Access another tenant
* Access another store
* Access another user's records
* Perform restricted actions
* Modify financial records
* Manipulate inventory
* Access hidden capabilities
* Export unauthorized data
* Abuse AI functionality

## 5.3 Compromised accounts

If a user account is compromised, authorization boundaries should limit the attacker's access.

Example:

```text
Compromised CASHIER
       ↓
Can access permitted POS operations
       ↓
Cannot automatically access
tenant administration
financial configuration
other stores
platform administration
```

## 5.4 Compromised infrastructure

The architecture should minimize the blast radius of compromised:

* Containers
* API credentials
* AWS roles
* CI/CD credentials
* Worker processes
* External API keys

---

# 6. Authentication

Authentication answers:

> **Who are you?**

Buzzsynx may support:

* Email/password
* Google OAuth
* Session management
* Password reset
* Email verification
* Account activation/deactivation

Authentication must be handled by trusted server-side infrastructure.

Authentication does not grant business access by itself.

After authentication, Buzzsynx must resolve:

```text
User
 ↓
Tenant Membership
 ↓
Tenant
 ↓
Store Scope
 ↓
Permissions
 ↓
Capabilities
```

---

# 7. Authentication Strategy

The exact session mechanism must be implemented consistently across the application.

For browser-based authentication, a secure server-managed session or securely managed access/refresh-token strategy may be used.

If cookie-based authentication is used:

```text
HttpOnly
Secure
SameSite
```

must be configured appropriately.

If tokens are used:

* Tokens must be securely issued.
* Expiration must be enforced.
* Refresh mechanisms must be protected.
* Token storage must be carefully designed.
* Revocation/session invalidation must be supported where required.

The API and frontend must not mix incompatible authentication models without an explicit security design.

---

# 8. Password Security

Passwords must never be stored in plaintext.

The system must use a modern password hashing algorithm with an appropriate cost configuration.

Preferred option:

```text
Argon2id
```

An appropriately configured bcrypt implementation may also be used where required by the current implementation.

Only the resulting password hash should be stored:

```text
passwordHash
```

Never store:

```text
password
plainPassword
```

Passwords must not appear in:

* Logs
* Analytics
* Error messages
* API responses
* Audit metadata

---

# 9. Password Requirements

Password requirements should prioritize resistance to guessing while remaining usable.

Controls may include:

* Minimum password length
* Password confirmation
* Common-password detection
* Breached-password checks where appropriate
* Secure password reset

Avoid unnecessarily complicated composition rules that encourage predictable passwords.

---

# 10. Password Reset

Password reset workflow:

```text
User Requests Reset
        ↓
Generate Cryptographically Secure Token
        ↓
Store Hashed Token
        ↓
Set Short Expiration
        ↓
Send Reset Link
        ↓
User Submits New Password
        ↓
Validate Token
        ↓
Hash New Password
        ↓
Invalidate Token
        ↓
Revoke Existing Sessions Where Appropriate
```

Reset tokens must:

* Be cryptographically random
* Expire
* Be single-use
* Not be logged
* Not be exposed through API responses
* Be stored securely if persistence is required

Password-reset responses should avoid revealing whether an account exists.

---

# 11. OAuth Security

For Google OAuth:

* Validate the OAuth provider response.
* Validate identity information.
* Validate redirect URIs.
* Prevent open redirects.
* Protect the OAuth state/CSRF mechanism where applicable.
* Link accounts carefully.
* Never trust client-supplied identity claims.

OAuth establishes authenticated identity.

Buzzsynx must still independently resolve:

```text
User
 ↓
Tenant Membership
 ↓
Store Scope
 ↓
Permissions
 ↓
Capabilities
```

---

# 12. Session Security

Sessions must be protected against:

* Session theft
* Session fixation
* Session replay
* Cross-site attacks
* Unauthorized reuse

Where cookies are used:

```text
HttpOnly
Secure
SameSite
```

should be appropriately configured.

Session management should support:

* Expiration
* Revocation
* Logout
* Session invalidation after sensitive security events
* Optional session/device visibility in future releases

---

# 13. Authentication vs Authorization

These concepts must remain separate.

### Authentication

```text
Who is the user?
```

### Authorization

```text
What is the user allowed to do?
```

Example:

```text
User authenticated
       ↓
Member of Tenant A
       ↓
Assigned to Store A
       ↓
Role = CASHIER
       ↓
Permission = sales:create
       ↓
Sale allowed
```

The same user should not automatically be able to operate Store B unless their membership/store access permits it.

---

# 14. Multi-Tenant Security

Tenant isolation is one of the highest-priority security requirements.

Every tenant-owned resource must have enforceable tenant ownership.

Examples:

```text
Product
tenantId

Sale
tenantId

Customer
tenantId

Inventory
tenantId

Supplier
tenantId
```

Store-owned records should additionally include appropriate store scope:

```text
tenantId
storeId
```

The backend must never use client-supplied `tenantId` as the security authority.

---

# 15. Tenant Context

After authentication, the backend resolves a trusted context.

Conceptually:

```js
const context = {
  userId,
  tenantId,
  activeStoreId,
  membership,
  roles,
  permissions,
  capabilities
};
```

This context must be derived from trusted server-side state.

`tenantId` is always server-derived.

`storeId` may be supplied by the client as a requested active store/context selector, but it must be validated against:

* Authenticated user
* Tenant membership
* Store status
* Store access
* Required permissions

---

# 16. Tenant Switching

If a user belongs to multiple tenants:

```text
Tenant A
Tenant B
Tenant C
```

switching tenants must re-resolve:

* Tenant membership
* Permissions
* Active store
* Store access
* Capabilities
* Relevant configuration

Previously selected tenant/store state must not be blindly trusted after switching.

Browser state is never authoritative.

---

# 17. Store / Branch Security

Store scope is an additional authorization boundary.

Example:

```text
Tenant A
│
├── Store A
└── Store B
```

A cashier assigned to Store A must not automatically access Store B.

Queries for store-owned resources must validate:

```text
tenantId
+
storeId
+
resource ownership
+
user permissions
```

Cross-store access should only be available when explicitly permitted.

---

# 18. Never Trust Client tenantId

Never treat these as authoritative:

```js
req.body.tenantId
```

```js
req.query.tenantId
```

```js
req.headers["x-tenant-id"]
```

The trusted flow is:

```text
Authenticated User
       ↓
Tenant Membership
       ↓
Trusted Tenant Context
```

A client may request a resource within a tenant context, but the server determines whether that context is valid.

---

# 19. Tenant-Scoped Queries

Tenant-owned queries must always enforce tenant scope.

Unsafe:

```js
prisma.product.findUnique({
  where: { id }
});
```

Safer:

```js
prisma.product.findFirst({
  where: {
    id,
    tenantId
  }
});
```

For store-owned resources:

```js
prisma.inventory.findFirst({
  where: {
    id,
    tenantId,
    storeId
  }
});
```

Updates and deletes must apply the same ownership checks.

The exact Prisma query shape may vary according to schema and unique constraints, but **resource ownership must always be enforced**.

---

# 20. Repository and Service Security

Services must not expose unrestricted data-access methods for tenant-owned resources.

Avoid generic patterns such as:

```js
getById(id)
```

when the caller can use the method without tenant/store context.

Prefer scoped access patterns such as:

```text
getProduct(context, productId)
```

or:

```text
getProduct({
  tenantId,
  storeId,
  productId
})
```

where appropriate.

The service layer remains responsible for business authorization and transaction boundaries.

---

# 21. IDOR Protection

Buzzsynx must protect against **Insecure Direct Object Reference (IDOR)**.

Example:

```text
Tenant A

GET /api/v1/products/product-123
```

Attacker changes:

```text
product-123
```

to:

```text
product-999
```

If `product-999` belongs to Tenant B, the request must not expose the resource.

Authorization must validate:

```text
Resource ID
+
Tenant Ownership
+
Store Scope Where Applicable
+
User Permission
```

For resources that should not be disclosed, a safe `404` response may be preferable to revealing that another tenant's resource exists.

---

# 22. Defense in Depth

Tenant and store security should use multiple layers:

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
Scoped Data Access
      ↓
Database Constraints
      ↓
Audit / Monitoring
```

No single layer should be considered sufficient.

---

# 23. RBAC

Current Buzzsynx roles are:

```text
SUPER_ADMIN
OWNER
ADMIN_MANAGER
CASHIER
ACCOUNTANT
STORE_STAFF
```

These roles should map to permissions.

Roles are not themselves the complete authorization system.

---

# 24. Permissions

Permissions represent business actions.

Examples:

```text
product:view
product:create
product:update
product:archive

inventory:view
inventory:adjust
inventory:transfer

sale:create
sale:view
sale:return

payment:view
payment:create
payment:refund

report:view

member:view
member:invite
member:update

store:view
store:create
store:update
```

The exact permission catalog should evolve with the implementation.

---

# 25. Super Admin Authorization

`SUPER_ADMIN` is a platform-level role.

Super Admin access is not equivalent to an ordinary tenant membership.

Platform operations may include:

* Tenant administration
* Tenant status changes
* Platform configuration
* Subscription/platform controls
* Operational support

Platform access must be:

* Explicit
* Least-privileged
* Audited
* Separated from ordinary tenant operations

A platform administrator accessing tenant data should generate appropriate audit records.

---

# 26. Capability Authorization

RBAC and capabilities are different.

### RBAC answers:

> Can this user perform this action?

### Capability answers:

> Is this business functionality enabled for this tenant?

Example:

```text
User:
STORE_STAFF

Permission:
inventory:view

Tenant:
Pharmacy

Capability:
EXPIRY_TRACKING
```

A protected operation may require both:

```text
inventory:view
+
EXPIRY_TRACKING
```

Capability configuration itself must also be permission-protected.

---

# 27. Capability Is Not Security by Itself

A capability does not grant permission.

For example:

```text
BATCH_TRACKING = ENABLED
```

does not mean every tenant user can edit batches.

Authorization still requires:

```text
Tenant
+
Store
+
Permission
+
Capability
+
Business Validation
```

---

# 28. Input Validation

All external input must be validated.

Input sources include:

* Request bodies
* Query parameters
* Route parameters
* Headers
* Cookies
* File metadata
* Webhooks
* OAuth callbacks
* Imported files
* External provider payloads

Use schema validation.

Conceptually:

```js
{
  name: string,
  sku: string,
  price: number,
  categoryId: string
}
```

Validation should reject:

* Invalid types
* Invalid formats
* Unexpected fields where appropriate
* Impossible values
* Oversized input

---

# 29. Mass Assignment Protection

Never blindly pass client objects into database writes.

Unsafe pattern:

```js
prisma.product.update({
  data: req.body
});
```

The client may attempt to modify protected fields such as:

```text
tenantId
storeId
createdBy
status
ownerId
internal fields
```

Use explicit allowed fields.

Server-controlled fields must be derived by the server.

---

# 30. API Security

Protected APIs must enforce appropriate:

* Authentication
* Tenant context
* Store scope
* Permissions
* Capabilities
* Input validation
* Rate limiting
* Request limits
* Safe error handling

Sensitive operations should also generate audit events where appropriate.

---

# 31. Rate Limiting

Rate limiting should protect high-risk and resource-intensive endpoints.

Examples:

```text
Login
Password Reset
Registration
OAuth
Public APIs
Search
AI endpoints
File uploads
Payment operations
Webhook endpoints
```

Redis may be used for distributed rate limiting.

Keys should be designed according to the endpoint and security context.

Examples:

```text
rate-limit:ip:{ip}:{endpoint}
```

```text
rate-limit:user:{userId}:{endpoint}
```

```text
rate-limit:tenant:{tenantId}:{endpoint}
```

Do not blindly include high-cardinality identifiers in every rate-limit key without considering storage and cleanup behavior.

---

# 32. Brute-Force Protection

Authentication endpoints should protect against repeated failed attempts.

Controls may include:

* Rate limiting
* Progressive delay
* Temporary lockout where appropriate
* Suspicious activity monitoring
* CAPTCHA/challenge where justified

Avoid account-enumeration leaks.

For example, password-reset requests should generally use a generic response.

---

# 33. SQL Injection Protection

Prisma parameterized queries should be preferred.

Avoid:

```js
const query =
  `SELECT * FROM products WHERE name = '${name}'`;
```

Prefer parameterized ORM operations.

If raw SQL is required:

* Parameterize values
* Validate inputs
* Restrict raw SQL usage
* Review security implications
* Avoid dynamic SQL identifiers unless explicitly allowlisted

---

# 34. Query / Filter Injection

Client-controlled query objects must not be passed directly into database APIs.

Validate:

```text
sort
filter
pagination
search
field selection
date ranges
```

against explicit schemas and allowlists.

Never allow a client to arbitrarily control database query structure.

---

# 35. XSS Protection

Treat user-generated content as untrusted.

Potential sources include:

* Product descriptions
* Customer names
* Notes
* Comments
* Rich text
* Uploaded metadata
* AI-generated content
* Imported data

Use appropriate output encoding and sanitization.

If rich text is supported, use a trusted sanitization strategy and restrict unsafe HTML.

---

# 36. CSRF Protection

If browser authentication uses cookies, state-changing operations must be protected against CSRF.

Possible controls include:

* SameSite cookies
* CSRF tokens
* Origin validation
* Referer validation where appropriate
* Strict CORS policy

CSRF strategy must match the actual authentication architecture.

---

# 37. CORS

CORS must explicitly define trusted origins.

Avoid:

```text
Access-Control-Allow-Origin: *
```

for authenticated APIs.

Production configuration should allow only approved frontend origins.

CORS is not an authentication mechanism.

---

# 38. HTTP Security Headers

Production responses should use appropriate security headers.

Potential headers include:

```text
Content-Security-Policy
Strict-Transport-Security
X-Content-Type-Options
Referrer-Policy
Permissions-Policy
```

The exact CSP should be tested against:

* Next.js
* Authentication
* Images
* External providers
* Analytics
* Payment integrations

Do not deploy a restrictive CSP blindly without validating application behavior.

---

# 39. HTTPS

Production traffic must use HTTPS.

Architecture:

```text
Client
  ↓
HTTPS
  ↓
Nginx / Load Balancer
  ↓
Application
```

Plain HTTP should not be used for sensitive production communication.

HTTP-to-HTTPS redirection should be configured where appropriate.

---

# 40. Secrets Management

Secrets must never be committed to Git.

Examples:

```text
DATABASE_URL
JWT_SECRET
SESSION_SECRET
GOOGLE_CLIENT_SECRET
RAZORPAY_KEY_SECRET
REDIS_PASSWORD
AI_API_KEY
AWS credentials
```

Development may use:

```text
.env.local
```

Production should use a secure secrets mechanism such as:

```text
AWS Secrets Manager
AWS Systems Manager Parameter Store
```

or an equivalent managed secret store.

Secrets must not be baked into Docker images.

---

# 41. Secret Rotation

Security-sensitive secrets should support rotation.

Examples:

* Database credentials
* OAuth secrets
* API keys
* JWT/session secrets where architecture permits
* AWS credentials
* Payment provider credentials

If a secret is exposed:

> **Treat it as compromised, revoke/rotate it, and investigate its usage.**

Deleting it from the latest commit is not sufficient.

---

# 42. Environment Separation

Buzzsynx should separate:

```text
Development
Staging
Production
```

Each environment should have independent:

* Databases
* Redis
* Storage
* OAuth credentials
* API keys
* Secrets
* Monitoring configuration

Production credentials must never be casually copied into local development.

---

# 43. Database Security

PostgreSQL should use:

* Strong credentials
* TLS where appropriate
* Restricted network access
* Least-privilege users
* Encrypted storage where supported
* Automated backups
* Restore testing
* Monitoring

The application must not connect using a PostgreSQL superuser.

---

# 44. Database Access Separation

Where practical, separate:

```text
Application DB Access
        ↓
Required runtime operations

Administrative DB Access
        ↓
Migrations
Maintenance
Administration
```

Runtime credentials should not automatically have unnecessary administrative privileges.

---

# 45. Database Constraints

Application authorization should be supported by database integrity constraints where appropriate.

Examples:

* Unique tenant-scoped identifiers
* Unique store-scoped identifiers
* Foreign-key relationships
* Required ownership fields
* Valid status values
* Transactional consistency

Database constraints do not replace authorization, but they reduce the impact of application mistakes.

---

# 46. Sensitive Data

Buzzsynx may process business and customer information.

Sensitive data should be:

* Minimized
* Access-controlled
* Protected
* Encrypted where appropriate
* Excluded from unnecessary logs
* Retained only as required
* Deleted or anonymized according to policy

Do not collect data merely because the schema can store it.

---

# 47. Payment Security

Buzzsynx should use trusted payment providers for payment processing.

Buzzsynx must not store raw card data.

Example:

```text
Buzzsynx
   ↓
Payment Provider
   ↓
Payment
   ↓
Provider Confirmation / Webhook
   ↓
Verified Payment State
```

Frontend payment success callbacks must never be treated as the sole source of truth.

---

# 48. Payment State vs Sale Transaction

Payment processing requires special handling because external providers operate outside the PostgreSQL transaction.

The architecture should distinguish:

```text
Sale State
```

from:

```text
Payment State
```

For example:

```text
Sale Created
      ↓
Payment Pending
      ↓
Provider Confirmation
      ↓
Verified Payment
      ↓
Sale Finalization / Completion
```

The exact state machine depends on the payment workflow.

Do not hold a PostgreSQL transaction open while waiting for an external payment provider.

---

# 49. Webhook Security

Payment and external-service webhooks must:

* Verify provider signatures
* Validate event payloads
* Validate expected event types
* Prevent or tolerate replay
* Be idempotent
* Record processing state
* Handle duplicate delivery
* Avoid trusting frontend-generated status

Workflow:

```text
Webhook Received
      ↓
Verify Signature
      ↓
Validate Event
      ↓
Check Idempotency
      ↓
Process
      ↓
Persist Result
      ↓
Return Success
```

Webhook processing should be designed so repeated delivery does not create duplicate financial effects.

---

# 50. Idempotency

Idempotency is required for security-sensitive and retryable operations where duplicate execution could create incorrect business results.

Examples:

* Payment webhooks
* Sale finalization where retries are possible
* Refunds
* Inventory receiving
* Background jobs
* External provider callbacks

Conceptually:

```text
Idempotency Key
      ↓
Check Previous Result
      ↓
Already Processed?
      ├── Yes → Return Existing Result
      └── No  → Process + Record
```

---

# 51. Transaction Security

Critical database operations must use appropriate PostgreSQL transactions.

For a synchronous internal transaction:

```text
Validate Request
      ↓
Begin Transaction
      ↓
Validate Current State
      ↓
Create Business Records
      ↓
Create Stock / Financial Records
      ↓
Create Audit Record Where Required
      ↓
Commit
```

If a critical database operation fails:

```text
ROLLBACK
```

The transaction must be short-lived.

External network calls should not be performed inside the transaction.

---

# 52. POS Transaction Integrity

For a supermarket POS sale, the critical database operation may include:

```text
Validate Cart
      ↓
Validate Current Stock
      ↓
Calculate Authoritative Totals
      ↓
Create Sale
      ↓
Create Sale Items
      ↓
Create Payment Allocation / State
      ↓
Create Stock Movement
      ↓
Update Stock Balance
      ↓
Create Invoice Record
      ↓
Commit
```

The exact ordering may vary with the payment model.

The key requirement is:

> **Financial and inventory state must remain internally consistent.**

Invoice PDF rendering, notifications, analytics and AI should occur after commit.

---

# 53. Inventory Concurrency

Inventory operations must handle concurrent requests.

Example:

```text
Available Stock = 1

User A → attempts sale
User B → attempts sale
```

The database must prevent both transactions from incorrectly consuming the same stock.

Possible mechanisms include:

* Database transactions
* Atomic conditional updates
* Appropriate row locking
* Correct transaction isolation
* Database constraints
* Idempotency

The exact strategy should be selected according to the inventory schema and transaction design.

---

# 54. Redis Security

Redis may be used for:

* Caching
* Rate limiting
* Sessions where applicable
* Temporary data
* Queue support
* Distributed coordination

Redis must never become the authoritative source for:

* Financial records
* Sales
* Inventory history
* Inventory truth

Sensitive Redis data must be protected.

Tenant/store scope must be reflected in keys where applicable.

Examples:

```text
tenant:{tenantId}:products
```

```text
tenant:{tenantId}:store:{storeId}:analytics
```

```text
rate-limit:user:{userId}:endpoint
```

---

# 55. Cache Security

Cached data must follow the same authorization boundaries as source data.

A cache key must not allow:

```text
Tenant A
```

to receive:

```text
Tenant B
```

data.

Store-specific cache entries must include appropriate store scope.

Sensitive data should not be cached unnecessarily.

Authorization must not depend solely on the presence of a cached object.

---

# 56. Background Job Security

BullMQ jobs are security-sensitive.

Jobs should carry the minimum required trusted context.

Example:

```js
{
  tenantId,
  storeId,
  jobType: "GENERATE_SALES_ANALYSIS",
  entityId
}
```

Workers must:

1. Validate the job payload.
2. Resolve the tenant context.
3. Resolve store scope where applicable.
4. Query only authorized tenant/store data.
5. Validate referenced entities.
6. Perform idempotent processing where required.
7. Log execution safely.
8. Handle failures without leaking sensitive information.

Never create a worker that performs unrestricted global tenant queries simply because it runs internally.

---

# 57. Queue Isolation

Background jobs must not assume that an internal queue is automatically trusted.

Workers should validate:

```text
tenantId
storeId
entity ownership
job type
payload schema
```

Jobs should not accept arbitrary authorization information such as:

```text
role: "OWNER"
```

from untrusted job producers and treat it as authoritative.

The worker should derive what it needs from trusted application state.

---

# 58. AI Security

AI functionality must follow tenant and store isolation.

Preferred flow:

```text
User Request
      ↓
Authentication
      ↓
Tenant / Store Authorization
      ↓
Capability Check
      ↓
Tenant-scoped Data Retrieval
      ↓
Data Minimization
      ↓
AI Processing
      ↓
Validate AI Output
      ↓
Tenant Response
```

AI must not have unrestricted database access.

---

# 59. AI Data Minimization

Only the data necessary for the AI task should be provided.

Avoid sending unnecessary:

* Customer information
* Personal information
* Payment details
* Internal secrets
* Authentication data
* Unrelated tenant data

Where possible, use aggregated or derived business information.

---

# 60. AI Prompt Injection

User-controlled content must be treated as untrusted input.

Potential sources:

```text
Product Description
Customer Note
Imported Document
Comment
Uploaded File
External Content
```

Such content must not be allowed to override:

* System instructions
* Authorization rules
* Tenant boundaries
* Tool restrictions
* Business rules

AI output must also be treated as untrusted application data.

---

# 61. AI Tool / Action Security

If AI is eventually allowed to call application tools, every tool must have explicit authorization.

Do not allow:

```text
AI
 ↓
Unrestricted Database
```

Prefer:

```text
AI
 ↓
Approved Tool
 ↓
Tool Authorization
 ↓
Tenant / Store Scope
 ↓
Validation
 ↓
Business Service
```

Critical operations such as:

* Changing inventory
* Issuing refunds
* Changing prices
* Modifying permissions
* Deleting data

must not be performed solely because an AI model requested them.

AI should primarily provide intelligence and recommendations unless an explicitly designed, authorized workflow exists.

---

# 62. File Upload Security

Potential uploads include:

* Product images
* Invoices
* Documents
* Prescriptions
* Business files

Security controls include:

* File-size limits
* MIME validation
* Extension validation
* Filename sanitization
* Malware scanning where appropriate
* Private storage for sensitive files
* Controlled access
* Signed URLs where applicable

Never execute uploaded files.

Do not trust the file extension alone.

---

# 63. File Ownership

Uploaded files must be associated with the correct:

```text
tenantId
storeId where applicable
entity type
entity ID
```

Access must be authorized before generating download URLs.

A signed URL must not become a substitute for application authorization.

---

# 64. Cloud Storage Security

If AWS S3 is used:

```text
Application
     ↓
Authenticated Request
     ↓
Authorization
     ↓
Generate Controlled Upload / Download
     ↓
Private S3 Bucket
```

Buckets should not be publicly writable.

Sensitive files should remain private.

Storage paths may use tenant/store prefixes:

```text
tenants/{tenantId}/stores/{storeId}/...
```

Do not allow the client to freely choose a storage path outside its authorized namespace.

---

# 65. Import / Export Security

Data imports and exports are security-sensitive.

Imports must not allow uploaded data to redefine ownership.

Never trust:

```text
tenantId
storeId
ownerId
```

from an uploaded file.

Ownership must come from the authenticated request context.

Exports must verify:

* Tenant scope
* Store scope
* User permission
* Data sensitivity
* Export limits

Large exports should be processed asynchronously and protected appropriately.

---

# 66. Logging Security

Logs must never contain:

```text
Passwords
Access Tokens
Refresh Tokens
API Secrets
Payment Secrets
Database Credentials
Private Keys
```

Avoid unnecessarily logging sensitive customer information.

Useful security context may include:

```text
requestId
userId
tenantId
storeId
action
resource
result
timestamp
```

Sensitive identifiers should be masked or omitted where appropriate.

---

# 67. Application Logs vs Audit Logs

These are different.

### Application logs

Used for:

* Debugging
* Errors
* Performance
* Operational monitoring

### Audit logs

Used for:

* Business accountability
* Security investigation
* Important administrative actions
* Sensitive state changes

Audit records should be protected from ordinary modification.

---

# 68. Audit Logging

Important actions should be auditable.

Examples:

```text
USER_CREATED
MEMBERSHIP_UPDATED
ROLE_CHANGED
STORE_CREATED
PRODUCT_CREATED
PRODUCT_UPDATED
INVENTORY_ADJUSTED
SALE_CREATED
SALE_RETURNED
PAYMENT_UPDATED
REFUND_CREATED
CAPABILITY_CHANGED
SETTINGS_CHANGED
```

Audit records should include relevant information:

```text
tenantId
storeId
userId
action
resourceType
resourceId
timestamp
requestId
reason where applicable
metadata
```

Do not store secrets inside audit metadata.

---

# 69. Error Handling

Production API responses must not expose internal implementation details.

Avoid:

```text
PrismaClientKnownRequestError...
PostgreSQL connection failed...
/server/modules/inventory/...
```

Use controlled responses:

```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Resource not found",
    "requestId": "req_123"
  }
}
```

Detailed technical information belongs in secure server logs.

---

# 70. Enumeration Protection

Avoid revealing unnecessary information about:

* Users
* Tenants
* Customers
* Invoice numbers
* Resource IDs
* Account existence
* Internal system state

Where appropriate, use generic responses.

However, enumeration protection should not interfere with legitimate application workflows or produce misleading security behavior.

---

# 71. API Response Security

API responses should return only the fields required by the client.

Never accidentally return:

```text
passwordHash
private tokens
secret configuration
database credentials
internal secrets
unnecessary personal data
```

Use explicit response schemas.

---

# 72. Pagination and Resource Limits

APIs must protect against expensive requests.

Controls should include:

```text
Maximum page size
Maximum request body size
Maximum file size
Maximum search length
Maximum export size
Query timeout where appropriate
```

Never allow unlimited client-controlled database queries.

---

# 73. Dependency Security

Dependencies should be reviewed regularly.

Use:

```bash
npm audit
```

and appropriate dependency/security tooling.

Important dependencies include:

* Next.js
* React
* Express
* Prisma
* Redis clients
* Authentication libraries
* Payment SDKs
* AI SDKs

Security updates should be prioritized according to severity and compatibility.

Do not blindly upgrade production dependencies without testing.

---

# 74. Docker Security

Containers should follow least privilege.

Prefer:

* Minimal base images
* Non-root users
* Minimal installed packages
* Read-only filesystem where practical
* No unnecessary Linux capabilities
* No secrets baked into images
* Explicit exposed ports
* Regular image updates

Never place production secrets inside:

```text
Dockerfile
docker-compose.yml
Git repository
Docker image
```

---

# 75. Nginx Security

Nginx may provide an additional security boundary.

Responsibilities may include:

* HTTPS termination
* Request size limits
* Rate limiting
* Reverse proxying
* Security headers
* Route restrictions
* Hiding internal service ports

Internal services should not be directly exposed to the public internet unless explicitly required.

---

# 76. AWS Security

When deployed to AWS, Buzzsynx should use:

* IAM least privilege
* Security Groups
* Private networking where appropriate
* Secrets Manager / Parameter Store
* CloudWatch
* Encryption
* S3 bucket policies
* Database network restrictions
* HTTPS
* Backup policies

AWS root credentials must not be used for routine application operations.

---

# 77. IAM

AWS IAM should follow least privilege.

Avoid giving application services:

```text
AdministratorAccess
```

unless there is a documented and exceptional requirement.

Use separate roles/identities for:

```text
Application
Deployment
CI/CD
Database Administration
Monitoring
Infrastructure Management
```

Where possible, prefer short-lived credentials and role-based access.

---

# 78. CI/CD Security

GitHub Actions must protect:

* AWS access
* Database credentials
* API keys
* Deployment secrets
* Signing credentials

Prefer OIDC-based AWS authentication where appropriate instead of long-lived AWS access keys.

CI pipeline:

```text
Install
  ↓
Lint
  ↓
Test
  ↓
Security Checks
  ↓
Build
  ↓
Deploy
```

Production deployments should use appropriate environment protection and approval controls.

---

# 79. Git Security

Never commit:

```text
.env
.env.local
.env.production
private keys
certificates
database dumps
credentials
```

Use:

* `.gitignore`
* Secret scanning
* Repository protection
* Pre-commit checks where useful

If a secret is accidentally committed:

> **Treat it as compromised and rotate it.**

---

# 80. Security Monitoring

Production monitoring should detect relevant events such as:

* Failed logins
* Account lockouts
* Permission failures
* Suspicious API activity
* Rate-limit violations
* Payment failures
* Webhook failures
* Database errors
* Queue failures
* Authentication anomalies
* Infrastructure anomalies

Potential tooling:

```text
Pino / Application Logs
Sentry
CloudWatch
Infrastructure Monitoring
```

Security monitoring should focus on actionable signals.

---

# 81. Metrics and Tenant Data

Tenant identifiers should not automatically be placed into high-cardinality metrics labels.

For example, avoid creating unrestricted metrics such as:

```text
request_count{tenantId="every-tenant-id"}
```

for large tenant populations.

Use logs or traces for detailed tenant-level investigation where appropriate.

Metrics should remain operationally useful and bounded.

---

# 82. Security Events

Example:

```text
50 Failed Login Attempts
        ↓
Rate Limit Triggered
        ↓
Security Event Recorded
        ↓
Account Protection
        ↓
Monitoring / Alert
```

Other events may include:

```text
Repeated Permission Denials
Unexpected Cross-Store Access
Webhook Signature Failures
Repeated Invalid Tokens
Unusual Export Activity
```

---

# 83. Backup Security

Backups must be:

* Automated
* Encrypted
* Access-controlled
* Retained according to policy
* Monitored
* Tested

Backup restoration must be tested periodically.

> **A backup that cannot be restored is not a reliable backup.**

Backups must receive the same tenant-data security considerations as primary storage.

---

# 84. Data Retention and Deletion

Deletion must consider:

```text
Primary Database
Cache
Files
Search Indexes
Background Jobs
Analytics
Audit Records
Backups
```

Business records may require retention instead of immediate physical deletion.

Retention policies should distinguish:

```text
User / Account Data
Operational Data
Financial Records
Audit Records
Uploaded Files
Backups
```

Deletion should be controlled and auditable where required.

---

# 85. Tenant Suspension

When a tenant is suspended:

* Business mutations should generally be blocked.
* Authentication may remain available depending on policy.
* Controlled read-only access may be permitted.
* Platform administration must remain available.
* Background jobs should respect tenant status.
* Scheduled operations must not continue unrestricted.

The exact suspension behavior must be defined per business requirement.

---

# 86. Tenant Deletion

Tenant deletion should not be implemented as an immediate destructive API call.

A controlled workflow should be used:

```text
Deletion Requested
      ↓
Authorization
      ↓
Confirmation / Policy Check
      ↓
Tenant Marked for Deletion
      ↓
Controlled Background Process
      ↓
Data / File Cleanup
      ↓
Audit Record
```

Financial and legally retained data may require different treatment.

---

# 87. Background Jobs and Tenant Lifecycle

Scheduled jobs must respect tenant status.

Example:

```text
Tenant SUSPENDED
       ↓
Scheduled AI Analysis
       ↓
Check Tenant Status
       ↓
Skip / Stop According to Policy
```

A worker must never assume that a tenant that was active when the job was created is still active when the job executes.

---

# 88. Security of Notifications

Notifications may contain sensitive business information.

Examples:

* Low-stock alerts
* Payment reminders
* Invoice information
* AI insights
* Expiry alerts

Notifications must respect:

* Tenant scope
* Store scope
* User permissions
* Notification preferences
* Data minimization

Do not put sensitive information into URLs or messages unnecessarily.

---

# 89. Security of Exports and Reports

Reports may contain highly sensitive business data.

Export/report endpoints must validate:

```text
User
Tenant
Store
Permission
Date range
Data scope
```

Generated files should:

* Have controlled access
* Expire where appropriate
* Not be publicly accessible
* Be deleted according to retention policy

---

# 90. Security Testing Strategy

Security must be tested continuously.

## Unit Tests

Test:

* Permission rules
* Capability rules
* Business authorization
* Input validation
* Sensitive business invariants

## Integration Tests

Test:

* Authentication
* Tenant isolation
* Store isolation
* Database access
* Payment workflows
* Webhooks
* Queue processing

## E2E Tests

Test:

* Login
* Role restrictions
* Tenant switching
* Store switching
* Critical POS workflows
* Inventory operations

## Security Tests

Test:

* IDOR
* Cross-tenant access
* Cross-store access
* Invalid permissions
* Injection attempts
* Rate limits
* Invalid tokens
* CSRF where applicable
* Malicious uploads
* Webhook signature failures
* Replay/idempotency
* Export authorization

---

# 91. Tenant Isolation Test

Minimum scenario:

```text
Tenant A
  Store A1
    Product A

Tenant B
  Store B1
    Product B
```

Authenticate as Tenant A.

Expected:

```text
Get Product A       → SUCCESS
Update Product A    → SUCCESS if authorized

Get Product B       → DENIED
Update Product B    → DENIED
Delete Product B    → DENIED
```

Also test cross-store access:

```text
Tenant A
  Store A1
  Store A2
```

User assigned only to Store A1:

```text
Store A1 data → ALLOWED
Store A2 data → DENIED
```

This test pattern should be applied to major tenant/store-owned domains.

---

# 92. Security Checklist

## Authentication

* [ ] Passwords hashed securely
* [ ] Secure session/token handling
* [ ] OAuth validation
* [ ] Password reset protected
* [ ] Account verification where required
* [ ] Brute-force protection
* [ ] Session revocation/invalidation

## Authorization

* [ ] RBAC implemented
* [ ] Permissions enforced server-side
* [ ] Capability checks enforced server-side
* [ ] Store scope enforced
* [ ] Platform admin access protected
* [ ] Administrative actions audited

## Multi-Tenancy

* [ ] Trusted tenant context
* [ ] No client-controlled tenant authority
* [ ] Tenant-scoped queries
* [ ] Store-scoped queries
* [ ] IDOR protection
* [ ] Cross-tenant tests
* [ ] Cross-store tests
* [ ] Tenant-aware cache
* [ ] Tenant-aware queues
* [ ] Tenant-aware AI
* [ ] Tenant-aware file storage

## API

* [ ] Input validation
* [ ] Mass-assignment protection
* [ ] Rate limiting
* [ ] Request size limits
* [ ] CORS configured
* [ ] Security headers
* [ ] Safe error responses
* [ ] Pagination/resource limits

## Database

* [ ] Least-privilege credentials
* [ ] Secure database connectivity
* [ ] Database constraints
* [ ] Transactional critical operations
* [ ] Query validation
* [ ] Encrypted backups
* [ ] Restore testing

## Payments

* [ ] No raw card storage
* [ ] Webhook signature verification
* [ ] Idempotency
* [ ] Payment state validation
* [ ] Reconciliation strategy
* [ ] No frontend-only payment trust

## Files

* [ ] MIME/type validation
* [ ] File size limits
* [ ] Filename sanitization
* [ ] Malware scanning where required
* [ ] Private storage
* [ ] Authorized signed URLs
* [ ] Tenant/store ownership

## AI

* [ ] Tenant isolation
* [ ] Store scope where relevant
* [ ] Data minimization
* [ ] Prompt-injection protection
* [ ] Tool authorization
* [ ] No unrestricted DB access
* [ ] AI output validation

## Infrastructure

* [ ] HTTPS
* [ ] Secure Docker images
* [ ] Non-root containers
* [ ] Protected AWS resources
* [ ] IAM least privilege
* [ ] Secrets management
* [ ] Network restrictions

## Monitoring

* [ ] Application logs
* [ ] Audit logs
* [ ] Security events
* [ ] Error monitoring
* [ ] Infrastructure monitoring
* [ ] Alerting
* [ ] Secret exposure monitoring

---

# 93. Security Definition of Done

A security-sensitive feature is not complete until:

* Authentication requirements are defined.
* Authorization requirements are defined.
* Tenant scope is enforced.
* Store scope is enforced where applicable.
* Input is validated.
* Client-controlled ownership fields are rejected/ignored.
* Sensitive data is protected.
* Errors are handled safely.
* Audit requirements are considered.
* Security tests exist.
* Logs do not expose secrets.
* Relevant rate limits exist.
* External integrations are verified.
* Idempotency is implemented where required.
* Documentation is updated.

---

# 94. Security Architecture Summary

The primary security flow is:

```text
                    INTERNET
                        │
                      HTTPS
                        │
                 NGINX / LB
                        │
                 Rate Limiting
                        │
                 Authentication
                        │
                Tenant Membership
                        │
                 Tenant Context
                        │
                  Store Scope
                        │
                 RBAC / Permissions
                        │
                Capability Check
                        │
                 Input Validation
                        │
                 Business Rules
                        │
             Scoped Data Access
                        │
                   PostgreSQL
                        │
               Audit / Monitoring
```

Supporting security layers:

```text
┌───────────────────────┐
│        Redis          │
│ Cache / Rate Limits   │
│ Temporary Data        │
└───────────────────────┘

┌───────────────────────┐
│       BullMQ          │
│ Secure Background     │
│ Jobs / Workers        │
└───────────────────────┘

┌───────────────────────┐
│         AI            │
│ Tenant + Store Scoped │
│ Approved Data / Tools │
└───────────────────────┘

┌───────────────────────┐
│    File Storage       │
│ Private + Controlled  │
│ Access                │
└───────────────────────┘

┌───────────────────────┐
│      AWS / IAM        │
│ Infrastructure        │
│ Security              │
└───────────────────────┘
```

---

# 95. Security Priority by Layer

Buzzsynx should prioritize security in the following order:

```text
1. Identity
      ↓
2. Tenant Isolation
      ↓
3. Store Isolation
      ↓
4. Authorization
      ↓
5. Business Transaction Integrity
      ↓
6. Input / API Security
      ↓
7. External Integration Security
      ↓
8. Data / File Security
      ↓
9. Infrastructure Security
      ↓
10. Monitoring / Response
```

This does not mean lower layers are optional.

It means the application must first establish trustworthy identity and authorization boundaries before allowing business operations.

---

# 96. Architecture vs Implementation Status

This document defines the **security architecture and required controls**.

Not every control must already be implemented at the current development stage.

Implementation should be tracked separately.

For example:

```text
Architecture Requirement
        ↓
Implementation
        ↓
Automated Test
        ↓
Production Verification
```

A documented security control must not be treated as implemented merely because it appears in this document.

---

# 97. Final Security Principle

> **Security in Buzzsynx is not a single authentication feature.**

It is a defense-in-depth system spanning:

```text
Identity
Tenant Isolation
Store Isolation
Authorization
Validation
Business Rules
Database Access
Transactions
Payments
Redis
Queues
AI
Files
Containers
AWS
Secrets
Logging
Monitoring
Testing
```

The most important rule remains:

> **Never trust the client. Resolve identity and tenant context on the server, validate store scope, authorize every operation, scope every tenant-owned resource, protect critical transactions, verify external integrations, and use multiple independent security layers.**

---

# 98. Final Architecture Statement

Buzzsynx security follows the principle:

```text
WHO?
 ↓
Authentication

WHICH BUSINESS?
 ↓
Tenant Membership

WHICH STORE?
 ↓
Store Scope

WHAT CAN THEY DO?
 ↓
RBAC / Permissions

IS THE FUNCTIONALITY ENABLED?
 ↓
Capability

IS THE REQUEST VALID?
 ↓
Validation

IS THE BUSINESS OPERATION ALLOWED?
 ↓
Business Rules

DOES THE RESOURCE BELONG HERE?
 ↓
Tenant + Store Scoped Data Access

CAN THE OPERATION BE TRUSTED?
 ↓
Transaction / Constraints / Idempotency

CAN WE TRACE IT?
 ↓
Audit / Monitoring
```

Therefore:

> **Buzzsynx security is built around trusted identity, strict tenant and store isolation, least-privilege authorization, server-side business enforcement, transactional integrity, secure integrations, protected infrastructure, and continuous verification.**
