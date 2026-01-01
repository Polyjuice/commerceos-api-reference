# CommerceOS API Resource Patterns

This document highlights repeatable resource patterns observed in the CommerceOS API.

---

## Agents Overview

Agents represent entities that can participate in commerce: individuals (people), businesses (companies), and retail locations (stores).

### Endpoints

| Endpoint | Description |
|----------|-------------|
| `/agents` | All agents (polymorphic) |
| `/people` | Person entities |
| `/companies` | Company entities |
| `/stores` | Store entities |

### Person Name Derivation

For Person entities, `name` is a **computed field** derived from `givenName + familyName`:

```bash
# WRONG - name is not included by default
GET /people~orderBy(name)~take(10)

# RIGHT - explicitly include computed name field
GET /people~with(name)~orderBy(name)~take(10)
```

**Key behaviors:**
- `name` requires `~with(name)` to include in response
- Setting `name` directly may be overwritten by the computed value
- Use `givenName` and `familyName` for reliable filtering

### Agent Finder

The agent finder (`/agents/@find`) uses heuristic lookup:

```bash
# Find by email
PUT /agents/@find/results
{"email": "john@example.com"}

# Find by phone
PUT /agents/@find/results
{"phone": "+46701234567"}

# Find by national ID
PUT /agents/@find/results
{"nationalId": "1990010112345"}

# Navigate to first result's name
PUT /agents/@find/results/0/name
{"email": "john@example.com"}
```

Name filtering uses metaphone (phonetic) matching for approximate name search.

---

## Agent Members

Agents (person, company, store) share common members:

| Member | Description | Write Semantics |
|--------|-------------|-----------------|
| `identifiers` | External IDs + key | Via common identifiers |
| `name` | Mapped from fullName property | Direct set |
| `nationality` | Agent nationality | Direct set |
| `languages` | Spoken languages | Direct set |
| `vatId` | VAT identification number | Direct set |
| `addresses` | Object with `main`, `home`, `invoice`, `delivery`, `visiting` | Each address settable |
| `contactMethods` | Object with `landlinePhone`, `mobilePhone`, `workPhone`, `email` | Each method settable |
| `confirmationAttempts` | Confirmation attempts for agent | Read-only |
| `customerGroups` | Groups owned by agent | Add/remove |
| `customerRelations` | Trade relationships where agent is supplier | Read-only |
| `supplierRelations` | Trade relationships where agent is customer | Read-only |
| `manufacturerRelations` | Manufacturer relationships | Read-only |
| `labels` | Assigned labels | Add/remove |
| `timeline` | Receipts for buyer | Read-only |
| `stockRoots` | Stock places attached to agent | Add/remove |
| `assortmentOwner` | Owner agent for assortment inheritance | Writable |
| `assortmentRoots` | Root product nodes in assortment | Add/remove |
| `assortment` | All product nodes in assortment | Add/remove |
| `preferredCurrency` | Preferred currency | Set by key |

Notes:
- `timeline` fetches receipts where agent is the buyer.
- `stockRoots` provides stock owner interface.
- `assortmentOwner` writes config at the agent node.
- Phone numbers are converted on set.
- Email addresses are converted on set.

---

## Product Node Hierarchy

Products exist in a hierarchy with different node types:

| Resource | Endpoint | Description |
|----------|----------|-------------|
| Products | `/products` | Concrete sellable items |
| Product Families | `/product-families` | Abstract product groupings |
| Product Groups | `/product-groups` | Groupings with hierarchy (variant containers) |
| Product Categories | `/product-categories` | Categorization with membership |
| Product Nodes | `/product-nodes` | Unified view of all node types |

The `/product-nodes` endpoint provides a unified view of all product node types with `assortmentOwners`, `assortmentContexts`, and `xrefs` expansions.

---

## Product Node Members

Product nodes share common members:

| Member | Description | Write Semantics |
|--------|-------------|-----------------|
| `identifiers` | External IDs + key | Via common identifiers |
| `name` | Product node name | Direct set |
| `gtin` | Global Trade Item Numbers | Direct set |
| `plu` | Price Look-Up codes | Direct set |
| `hidden` | Hidden flag | Direct set |
| `createdAt` | Creation timestamp | Read-only |
| `createdBy` | Creating agent | Read-only |
| `promotionTitle` | Localized promotion title | Localized set |
| `promotionBanner` | Localized promotion banner | Localized set |
| `promotionDescription` | Localized promotion description | Localized set |
| `notesForPicking` | Picker instructions | Direct set |
| `images` | Image files | **Full replace on set** |
| `assortmentOwners` | Agents owning this node | Add/remove |
| `assortmentContexts` | Assortment relation contexts | **Nested fields writable** (see below) |
| `categories` | Category assignments with weight | Add/remove; weight settable per assignment |
| `labels` | Assigned labels | Add/remove |
| `prices` | Price rules for this product | **Replace on set** |
| `xrefs.compatibles` | Compatible products | Add/remove |
| `xrefs.alternatives` | Alternative products | Add/remove |
| `application.webshop` | Webshop settings | Nested fields writable |

### assortmentContexts

The collection itself is read-only, but **each context element has writable nested fields**:

| Nested Field | Description | Write Semantics |
|--------------|-------------|-----------------|
| `owner` | Owning agent | Read-only |
| `articleNumber` | Custom article number for this owner | Settable |
| `minimumOrderQuantity` | Minimum order quantity | Settable |
| `primarySupplier` | Primary supplier company | Settable |

When setting nested fields, the relation is created if it doesn't exist.

### categories

Uses a wrapper object pattern:
- `category`: The assigned category
- `weight`: Child weight in the category

Weights stored on the category relation.

### prices

Gets prices specific to this node.
- Replace on set means setting the array replaces all existing prices
- On add, price patterns are updated
- On remove, if no other products remain, price is purged entirely

---

## Products and Categories

### Products

Products represent sellable items with status, identifiers, and various attributes.

**Default Fields (returned without `~with`):**
- `status`: One of `Active`, `Inactive`, `Pending`
- `gtin`: Array of barcode identifiers (string array supporting index access)
- `name`: Product name (always returned)

**Non-essential Fields (require `~with`):**
- `plu`: Array of price look-up codes
- `hidden`: Boolean flag for visibility

**GTIN/PLU Array Access:**

```bash
# Access first GTIN
GET /products/{id}/gtin/0

# Access last GTIN
GET /products/{id}/gtin/-1

# Count GTINs
GET /products/{id}/gtin/count
```

**Localized Fields (require `~with`):**
- `promotionTitle`, `receiptText`, `signText`

```bash
# Filter active products with localized fields
GET /products~where(status=Active)~with(promotionTitle,receiptText)~take(20)

# Filter by name pattern
GET /products~where(name=~Widget)~take(10)
```

### Categories

| Endpoint | Description |
|----------|-------------|
| `/product-categories` | Category collection |
| `/product-categories/{id}/members` | Products in category |
| `/products/{id}/categories` | Categories a product belongs to |

**Category Fields:**
- Localized promotion fields (require `~with`)
- `images`, `labels` expansions
- `childCategories` for hierarchical navigation

```bash
# Categories with expansions
GET /product-categories~with(images,labels)~take(20)

# Members of a category with pagination
GET /product-categories/{id}/members~take(50)~skip(100)

# Count products in category
GET /product-categories/{id}/members/count
```

### Category Membership

Products belong to categories through the `categories` member:

```bash
# Product's categories
GET /products/{id}/categories

# Category's parent categories
GET /product-categories/{id}/parentCategories

# Navigate via parentCategories relation
GET /products/{id}/categories/0/category/parentCategories
```

### Product Groups and Families

| Endpoint | Description |
|----------|-------------|
| `/product-groups` | Product groups (variant containers) |
| `/product-families` | Product families |

Groups define variant dimensions, default VAT codes, instance types, and age restrictions.

---

## Assortment Contexts

Assortment contexts link products to owning agents with owner-specific metadata.

### Structure

| Field | Description |
|-------|-------------|
| `owner` | Owning agent (company/store) |
| `articleNumber` | Owner-specific article number |
| `minimumOrderQuantity` | Minimum order quantity for this owner |
| `primarySupplier` | Primary supplier (company reference) |

### Endpoints

```bash
# Assortment contexts for a product
GET /products/{id}/assortmentContexts

# Access by owner ID
GET /products/{id}/assortmentContexts/{owner-id}

# Agent's assortment
GET /companies/{id}/assortment
GET /stores/{id}/assortment

# Agent's assortment roots
GET /companies/{id}~with(assortmentRoots)
```

### Product expansions

```bash
# Product with assortment owners expanded
GET /products/{id}~with(assortmentOwners)

# Product node with xrefs expanded
GET /products/{id}~with(xrefs)
```

---

## Deep Indexing Patterns

The API supports deep navigation into resources using URL path segments.

### External Identifier Lookup

Objects with `commonIdentifiers` can be accessed by any external identifier namespace:

```bash
# Access by namespace-qualified external ID
GET /people/com.myapp.userId=123

# Multiple identifiers on same object work interchangeably
GET /products/com.legacy.sku=ABC-001
GET /products/com.newapp.productId=xyz789
# Both resolve to the same product if it has both identifiers
```

### Database Key Lookup

Every resource has an internal `key` in its identifiers:

```bash
# Access by database key
GET /products/key=abc123def456
GET /agents/key=987654321
```

### Array Index Access

Array members support numeric index access:

```bash
# Access first element (0-indexed)
GET /products/com.test.sku=ABC/gtin/0

# Access last element with -1
GET /products/com.test.sku=ABC/gtin/-1

# Get array length
GET /products/com.test.sku=ABC/gtin/count
```

### Sub-Collection Navigation

Collections expose their elements as navigable paths:

```bash
# Stock place transactions sub-collection
GET /stock-places/com.test.placeId=WH1/transactions

# Product categories sub-collection
GET /products/com.test.sku=ABC/categories

# Category members (products in category)
GET /product-categories/com.test.catId=ELEC/members

# Agent customer relations
GET /companies/com.test.id=CORP/customerRelations
```

### Nested Relation Navigation

Navigate through relations to reach deeply nested data:

```bash
# Navigate through product to its category to that category's parent
GET /products/com.test.sku=ABC/categories/0/category/parent

# Navigate from order to customer's addresses
GET /trade-orders/com.test.orderId=ORD1/customer/addresses/main

# Navigate to count of items in a collection
GET /trade-orders/com.test.orderId=ORD1/items/count
```

### Receipt Date Indexing

Receipts support efficient date-based access via indexed paths:

```bash
# Receipts after specific datetime
GET /receipts/after/2024-12-01T00:00:00Z

# Receipts before specific datetime
GET /receipts/before/2024-12-31T23:59:59Z

# Combined range
GET /receipts/after/2024-12-01T00:00:00Z/before/2024-12-31T23:59:59Z

# Relative syntax: hours ago
GET /receipts/after/-=24        # Last 24 hours
GET /receipts/after/-=168       # Last week (168 hours)

# Relative syntax: hours and minutes
GET /receipts/after/-=0:30      # Last 30 minutes
```

---

## Structured Slot Patterns

### Addresses

Agent addresses use a structured slot pattern with predefined keys:

| Slot | Description |
|------|-------------|
| `main` | Primary/default address |
| `home` | Home address |
| `invoice` | Billing address |
| `delivery` | Shipping address |
| `visiting` | Physical visit address |

Access and update via PATCH:

```bash
# Read main address
GET /people/com.test.id=123~with(addresses/main)

# Update delivery address
PATCH /people/com.test.id=123
{"addresses": {"delivery": {"line1": "123 Main St", "cityName": "Stockholm", "countryCode": "SE"}}}
```

### Contact Methods

Contact methods follow the same slot pattern:

| Slot | Description |
|------|-------------|
| `landlinePhone` | Landline telephone |
| `mobilePhone` | Mobile telephone |
| `workPhone` | Work telephone |
| `email` | Email address |

**Important:** Use `contactMethods`, not `contactPoints`.

```bash
# Read contact methods
GET /people/com.test.id=123~with(contactMethods)

# Update email
PATCH /people/com.test.id=123
{"contactMethods": {"email": "new@example.com"}}
```

---

## Trade Orders

Trade orders represent sales or purchase transactions between agents.

### Required Fields

- **`items`**: Must be non-empty (at least 1 item required)
- **`sellers`**: Must be non-empty (at least 1 seller required)

Orders missing either of these will fail validation.

### Product References

Product identifiers in trade order items require a **fully qualified namespace**:

```json
// WRONG - bare key
{"product": {"identifiers": {"sku": "PHONE-001"}}}

// RIGHT - namespaced key
{"product": {"identifiers": {"com.example.sku": "PHONE-001"}}}
```

### Instance Tracking

For serialized products (devices, SIM cards), use `instances` instead of `quantity`:

| Instance Type | Instance Field | Description |
|---------------|----------------|-------------|
| MobileDevice | `imei` | Device IMEI number |
| MobilePlan | `phoneImei` | References a device IMEI from an earlier item |

**Order matters:** A MobilePlan item with `phoneImei` must reference an IMEI from a MobileDevice item that appears earlier in the items array.

> **Tip:** Products need the correct `instanceType` set during product creation (not on the order). This is handled by product setup, not order creation.

### Order Actions

| Action | Effect |
|--------|--------|
| `tryApprove` | Sets order status to `Committed` |
| `createPayment` | Creates a payment record with `transactionId`, `timestamp`, `method`, `currency`, `items`, `amount` |

### Expansions

| Expansion | Description |
|-----------|-------------|
| `~with(manualDiscounts)` | Manual discounts applied to order |
| `~with(payments)` | Payment records for this order |

Each order item has a `discountable` flag indicating whether it accepts discounts.

---

## Trade Relationships

Trade relationships define B2B or B2C connections between agents.

### Required Fields

- **`supplierAgent`**: The agent providing goods/services
- **`customerAgent`**: The agent receiving goods/services

### Optional Fields

| Field | Description |
|-------|-------------|
| `creditAllowed` | Whether credit is allowed |
| `allowsBackOrder` | Whether backorders are allowed |
| `supplierId` / `customerId` | External IDs |
| `intervalStart` / `intervalEnd` | Validity period |
| `acceptedMembershipTerms` | Membership acceptance flag |
| `acceptedPromotionalMaterial` | Marketing consent flag |

### Agent-Side Access

```bash
# Company's customers
GET /companies/{id}/customerRelations

# Company's suppliers
GET /companies/{id}/supplierRelations
```

Use `~with` to include non-essential fields in the response.

---

## Trade Rules

Trade rules define discounting and pricing logic.

### Endpoints

| Endpoint | Description |
|----------|-------------|
| `/discount-phases` | Discount phase definitions |
| `/discount-reasons` | Reason codes for discounts |
| `/discount-rules` | Automatic discount rules |
| `/discount-rule-effects` | Effects applied by rules |
| `/price-rules` | Pricing rules |
| `/return-reasons` | Reason codes for returns |

### Manual Discount Types

| Type | Description |
|------|-------------|
| Percentage | Percentage reduction |
| FixedReduction | Fixed amount reduction |
| FixedPrice | Override to fixed price |

Edge cases handled: zero priority, 100% discount, zero reduction amount.

---

## Prices and Currencies

### Price Fields

| Field | Description |
|-------|-------------|
| `amount` | Price amount (string for precision) |
| `currency` | Currency reference |
| `products` | Products this price applies to |
| `sellers` | Selling agents (company or store) |
| `from` / `to` | Validity period |

### Sub-Collections

```bash
# Prices for a specific product
GET /products/{id}/prices

# Product with prices expanded
GET /products/{id}~with(prices)
```

### Currencies

Currencies are resources at `/currencies`:

```bash
GET /currencies
GET /currencies/currencyCode=SEK
```

---

## Stock Management

### Stock Places

Stock places represent physical or logical storage locations.

| Member | Description |
|--------|-------------|
| `owner` | Agent owning the stock place |
| `effectiveAddress` | Computed address (read-only) |
| `parent` / `children` | Hierarchical structure |
| `entries` | Current stock levels |
| `transactions` | Stock transaction history |

### Stock Transactions

```bash
# Transactions at a stock place
GET /stock-places/{id}/transactions
```

Transactions support positive, negative, and zero quantities with product and stock place references.

### Stock Adjustment Reasons

Adjustment reasons are managed at `/stock-adjustment-reasons`.

### Agent Stock Roots

Agents expose their stock places via `stockRoots`:

```bash
GET /companies/{id}~with(stockRoots)
GET /stores/{id}~with(stockRoots)
```

---

## Receipts

Receipts are read-only records of POS transactions.

### Read-Only Nature

Receipts are primarily read-only after creation. Use projections and ordering to query:

```bash
# Recent receipts ordered by timestamp
GET /receipts~orderBy(timestamp:desc)~take(20)

# Receipts with expansions
GET /receipts~with(items,payments)~take(10)
```

### Date-Indexed Access

Use `/receipts/after/` and `/receipts/before/` for efficient date queries (uses database indexes):

```bash
GET /receipts/after/-=24          # Last 24 hours
GET /receipts/after/2024-12-01T00:00:00Z/before/2024-12-31T23:59:59Z
```

### Properties Endpoint

```bash
GET /receipts/{id}/properties
```

---

## Shipments

Shipments track order delivery and fulfillment.

### Resources

| Endpoint | Description |
|----------|-------------|
| `/shipment-orders` | Shipment order requests |
| `/shipment-records` | Actual shipment records |
| `/shipment-items` | Individual items in shipments |
| `/delivery-terms` | Incoterm delivery terms |

### Sub-Collections

```bash
# Items in a shipment order
GET /shipment-orders/{id}/items
```

### Linking

Shipment records can link back to shipment orders. Use `incotermCode` for delivery terms classification.

---

## Payments and POS

### Payment Methods

| Member | Description |
|--------|-------------|
| `supportsIncoming` | Accepts incoming payments |
| `supportsOutgoing` | Supports outgoing payments |
| `supportsReversal` | Supports reversal transactions |
| `requiresTerminal` | Requires a payment terminal |
| `requiresSpecification` | Requires additional specification |
| `consumerFriendlyTitle` | Display title |
| `consumerFriendlyBanner` | Display banner |

### Payment Orders

| Member | Description |
|--------|-------------|
| `amount` | Payment amount |
| `currency` | Payment currency |
| `method` | Payment method |
| `payer` / `payee` | Payment parties |

Sub-collection: `/payment-orders/{id}/records`

### POS Terminals

| Member | Description |
|--------|-------------|
| `posTerminalName` | Terminal identifier |
| `associatedNode` | Associated product node (not `store`) |

### POS Resources

| Endpoint | Description |
|----------|-------------|
| `/pos-terminals` | Terminal definitions |
| `/pos-profiles` | Terminal profiles (`posProfileId`) |
| `/pos-functions` | Terminal functions |
| `/pos-reports` | Z/X and cash register reports |
| `/pos-devices` | Hardware devices |
| `/pos-device-roles` | Device roles |
| `/pos-printers` | Receipt printers |

---

## Users and Credentials

### User Resources

| Member | Description |
|--------|-------------|
| `agent` | Linked agent reference |
| `name` | User display name (PATCH updatable) |

```bash
# User by key
GET /users/key={userKey}

# Update user name
PATCH /users/{id} '{"name": "New Name"}'
```

### Credentials Sub-Collections

| Sub-Collection | Description | Key Field |
|----------------|-------------|-----------|
| `localCredentials` | Email-based auth | `email` |
| `retailCredentials` | Username-based auth | `username` |
| `pinCredentials` | PIN-based auth | - |
| `apikeyCredentials` | API key auth | - |
| `oauth2Clients` | OAuth2 clients | - |

### Roles and Permissions

```bash
# Role assignments for a user
GET /users/{id}/roleAssignments

# Available user roles
GET /user-roles

# Permissions collection
GET /user-permissions
```
