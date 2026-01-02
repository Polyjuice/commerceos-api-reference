# CommerceOS API Examples

Comprehensive curl examples demonstrating the full capabilities of the CommerceOS API.

**Base URL:** `http://localhost:5000/api/v1`
**API Key:** `banana` (passed via Basic Auth with empty username: `-u ":banana"`)

> **See also:** The [reference documentation](../reference/) for detailed API specifications.

---

## Examples Index

Quick links to examples by resource group. Use these anchors for stable linking.

| Tag | Examples | Key Operations |
|-----|----------|----------------|
| [Organization](#organization-examples) | Agents, People, Companies, Stores | CRUD, relations, addresses |
| [Products](#products-examples) | Products, Categories, Families, Groups | CRUD, GTIN/PLU lookup, labels, images |
| [Pricing](#pricing-examples) | Prices, Currencies, VAT Codes | Price creation, validity periods, buyer restrictions |
| [Orders](#orders-examples) | Trade Orders, Trade Relationships, Shipments, Payments | Order lifecycle, fulfillment, payments |
| [Inventory](#inventory-examples) | Stock Places, Stock Entries, Transactions | Stock queries, adjustments, reservations |
| [POS](#pos-examples) | Terminals, Receipts, Payment Methods | Receipt creation, payment processing |
| [Users](#users-examples) | Users, Authentication, Roles | User management, role assignments |
| [Configuration](#configuration-examples) | Config, Reference Data, Integrations | System settings, mapped types, webhooks |
| [Operators](#operators-examples) | Query operators | Filtering, pagination, expansion, projection |
| [Advanced](#advanced-examples) | Complex queries | Chaining, bulk operations, exports |

---

## Table of Contents

1. [Core Agents & Organizations](#1-core-agents--organizations)
2. [Products & Catalog](#2-products--catalog)
3. [Prices & Pricing](#3-prices--pricing)
4. [Trade Orders & Fulfillment](#4-trade-orders--fulfillment)
5. [Stock & Inventory](#5-stock--inventory)
6. [Point of Sale (POS)](#6-point-of-sale-pos)
7. [Users & Authentication](#7-users--authentication)
8. [Configuration & Reference Data](#8-configuration--reference-data)
9. [Query Operators Reference](#9-query-operators-reference)
10. [Advanced Query Patterns](#10-advanced-query-patterns)
11. [SQL Export Serialization](#11-sql-export-serialization)

---

<a id="organization-examples"></a>

## 1. Core Agents & Organizations

Agents are the foundational entities representing people, companies, and stores.

### 1.1 People

```bash
# List all people
curl -X GET -u ":banana" "localhost:5000/api/v1/people"

# Get person by external ID
curl -X GET -u ":banana" "localhost:5000/api/v1/people/com.myapp.customerId=CUST-001"

# Get person with addresses included
curl -X GET -u ":banana" "localhost:5000/api/v1/people/com.myapp.customerId=CUST-001~with(addresses)"

# Get person's customer relationships
curl -X GET -u ":banana" "localhost:5000/api/v1/people/com.myapp.customerId=CUST-001/customerRelations"

# Create a person
curl -X POST -u ":banana" "localhost:5000/api/v1/people" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.customerId": "CUST-001"},
    "givenName": "John",
    "familyName": "Doe"
  }'

# Create person with full details
curl -X POST -u ":banana" "localhost:5000/api/v1/people" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.customerId": "CUST-002"},
    "givenName": "Jane",
    "familyName": "Smith",
    "addresses": {
      "main": {
        "line1": "Kungsgatan 1",
        "postalCode": "11143",
        "cityName": "Stockholm",
        "countryCode": "SE"
      }
    },
    "contactMethods": {
      "email": "jane@example.com",
      "mobilePhone": "+46701234567"
    }
  }'

# Update person (partial)
curl -X PATCH -u ":banana" "localhost:5000/api/v1/people/com.myapp.customerId=CUST-001" \
  -H "Content-Type: application/json" \
  -d '{"familyName": "Doe-Smith"}'

# Delete person
curl -X DELETE -u ":banana" "localhost:5000/api/v1/people/com.myapp.customerId=CUST-001"
```

### 1.2 Companies

```bash
# List all companies
curl -X GET -u ":banana" "localhost:5000/api/v1/companies"

# Get company by external ID
curl -X GET -u ":banana" "localhost:5000/api/v1/companies/com.heads.seedID=ourcompany"

# Get company with supplier relations
curl -X GET -u ":banana" "localhost:5000/api/v1/companies/com.heads.seedID=ourcompany~with(supplierRelations)"

# Get company's assortment roots (product categories it owns)
curl -X GET -u ":banana" "localhost:5000/api/v1/companies/com.heads.seedID=ourcompany/assortmentRoots"

# Create a company
curl -X POST -u ":banana" "localhost:5000/api/v1/companies" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.companyId": "COMP-001"},
    "name": "Acme Corporation",
    "organizationNumber": "556123-4567"
  }'

# Create company with parent (subsidiary)
curl -X POST -u ":banana" "localhost:5000/api/v1/companies" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.companyId": "COMP-002"},
    "name": "Acme Subsidiary",
    "parent": {"identifiers": {"com.myapp.companyId": "COMP-001"}}
  }'

# Update company
curl -X PATCH -u ":banana" "localhost:5000/api/v1/companies/com.myapp.companyId=COMP-001" \
  -H "Content-Type: application/json" \
  -d '{"name": "Acme Corp International"}'
```

### 1.3 Stores

```bash
# List all stores
curl -X GET -u ":banana" "localhost:5000/api/v1/stores"

# Get store by external ID
curl -X GET -u ":banana" "localhost:5000/api/v1/stores/com.heads.seedID=store1"

# Get store with opening hours
curl -X GET -u ":banana" "localhost:5000/api/v1/stores/com.heads.seedID=store1~with(openingHours)"

# Get store's assortment (products it carries)
curl -X GET -u ":banana" "localhost:5000/api/v1/stores/com.heads.seedID=store1/assortment"

# Get store's stock roots (warehouses/stock places)
curl -X GET -u ":banana" "localhost:5000/api/v1/stores/com.heads.seedID=store1/stockRoots"

# Create a store (NOTE: uses "owner" not "parent")
curl -X POST -u ":banana" "localhost:5000/api/v1/stores" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.storeId": "STORE-001"},
    "name": "Downtown Store",
    "owner": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "addresses": {
      "main": {
        "line1": "Drottninggatan 50",
        "postalCode": "11121",
        "cityName": "Stockholm",
        "countryCode": "SE"
      }
    }
  }'

# Create store with organization number
curl -X POST -u ":banana" "localhost:5000/api/v1/stores" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.storeId": "STORE-002"},
    "name": "Mall Store",
    "owner": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "organizationNumber": "556789-0123"
  }'

# Update store
curl -X PATCH -u ":banana" "localhost:5000/api/v1/stores/com.myapp.storeId=STORE-001" \
  -H "Content-Type: application/json" \
  -d '{"name": "Downtown Flagship Store"}'
```

### 1.4 Generic Agents

```bash
# List all agents (people, companies, stores combined)
curl -X GET -u ":banana" "localhost:5000/api/v1/agents"

# Filter agents by type
curl -X GET -u ":banana" "localhost:5000/api/v1/agents~where(@type=person)"
curl -X GET -u ":banana" "localhost:5000/api/v1/agents~where(@type=company)"
curl -X GET -u ":banana" "localhost:5000/api/v1/agents~where(@type=store)"

# Get agent by database key
curl -X GET -u ":banana" "localhost:5000/api/v1/agents/key=abc123def456"

# Get agent's addresses
curl -X GET -u ":banana" "localhost:5000/api/v1/agents/com.heads.seedID=ourcompany/addresses"
curl -X GET -u ":banana" "localhost:5000/api/v1/agents/com.heads.seedID=ourcompany/addresses/main"

# Get agent's contact methods
curl -X GET -u ":banana" "localhost:5000/api/v1/agents/com.heads.seedID=ourcompany/contactMethods"

# Get agent's labels
curl -X GET -u ":banana" "localhost:5000/api/v1/agents/com.heads.seedID=ourcompany/labels"
```

---

<a id="products-examples"></a>

## 2. Products & Catalog

### 2.1 Products

```bash
# List all products
curl -X GET -u ":banana" "localhost:5000/api/v1/products"

# Get product by external ID
curl -X GET -u ":banana" "localhost:5000/api/v1/products/com.heads.seedID=iphone16"

# Get product by GTIN (use ~where, NOT direct indexer)
curl -X GET -u ":banana" "localhost:5000/api/v1/products~where(gtin=7311250449246)~first"

# Get product by PLU
curl -X GET -u ":banana" "localhost:5000/api/v1/products~where(plu=4011)~first"

# Get product with prices
curl -X GET -u ":banana" "localhost:5000/api/v1/products/com.heads.seedID=iphone16~with(prices)"

# Get product with categories
curl -X GET -u ":banana" "localhost:5000/api/v1/products/com.heads.seedID=iphone16~with(categories)"

# Get product with assortment contexts
curl -X GET -u ":banana" "localhost:5000/api/v1/products/com.heads.seedID=iphone16/assortmentContexts"

# Get product's specific assortment context (by owner)
curl -X GET -u ":banana" "localhost:5000/api/v1/products/com.heads.seedID=iphone16/assortmentContexts/com.heads.seedID=ourcompany"

# Get product's images
curl -X GET -u ":banana" "localhost:5000/api/v1/products/com.heads.seedID=iphone16/images"

# Get product's labels
curl -X GET -u ":banana" "localhost:5000/api/v1/products/com.heads.seedID=iphone16/labels"

# Create a simple product
curl -X POST -u ":banana" "localhost:5000/api/v1/products" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.sku": "SKU-001"},
    "name": "Basic T-Shirt",
    "status": "Active"
  }'

# Create product with GTIN and PLU
curl -X POST -u ":banana" "localhost:5000/api/v1/products" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.sku": "SKU-002"},
    "name": "Premium Headphones",
    "gtin": ["7312345678901"],
    "plu": ["1234"],
    "status": "Active"
  }'

# Create product with assortment context
curl -X POST -u ":banana" "localhost:5000/api/v1/products" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.sku": "SKU-003"},
    "name": "Organic Coffee",
    "status": "Active",
    "assortmentContexts": [
      {
        "owner": {"identifiers": {"com.heads.seedID": "ourcompany"}},
        "articleNumber": "ART-12345"
      }
    ]
  }'

# Update product
curl -X PATCH -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001" \
  -H "Content-Type: application/json" \
  -d '{"name": "Premium T-Shirt", "status": "Active"}'

# Assign product to category
curl -X POST -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001/categories" \
  -H "Content-Type: application/json" \
  -d '{"category": {"identifiers": {"com.myapp.catId": "CAT-001"}}}'

# Add product to assortment owner
curl -X POST -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001/assortmentOwners" \
  -H "Content-Type: application/json" \
  -d '{"@type": "store", "identifiers": {"com.heads.seedID": "store1"}}'

# Delete product
curl -X DELETE -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001"
```

### 2.2 Product Categories

```bash
# List all product categories
curl -X GET -u ":banana" "localhost:5000/api/v1/product-categories"

# Get category by external ID
curl -X GET -u ":banana" "localhost:5000/api/v1/product-categories/com.heads.seedID=electronics"

# Get category with child categories
curl -X GET -u ":banana" "localhost:5000/api/v1/product-categories/com.heads.seedID=electronics~with(childCategories)"

# Get category members (products)
curl -X GET -u ":banana" "localhost:5000/api/v1/product-categories/com.heads.seedID=electronics/members"

# Create a root category
curl -X POST -u ":banana" "localhost:5000/api/v1/product-categories" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.catId": "CAT-001"},
    "name": "Electronics"
  }'

# Create a child category
curl -X POST -u ":banana" "localhost:5000/api/v1/product-categories" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.catId": "CAT-002"},
    "name": "Smartphones",
    "parent": {"identifiers": {"com.myapp.catId": "CAT-001"}}
  }'

# Update category
curl -X PATCH -u ":banana" "localhost:5000/api/v1/product-categories/com.myapp.catId=CAT-001" \
  -H "Content-Type: application/json" \
  -d '{"name": "Electronics & Gadgets"}'
```

### 2.3 Product Families (Variants)

```bash
# List all product families
curl -X GET -u ":banana" "localhost:5000/api/v1/product-families"

# Get product family with variants
curl -X GET -u ":banana" "localhost:5000/api/v1/product-families/com.heads.seedID=iphone-family~with(variants)"

# Create a product family
curl -X POST -u ":banana" "localhost:5000/api/v1/product-families" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.familyId": "FAM-001"},
    "name": "iPhone 16 Family"
  }'

# Add variant dimension
curl -X PATCH -u ":banana" "localhost:5000/api/v1/product-families/com.myapp.familyId=FAM-001" \
  -H "Content-Type: application/json" \
  -d '{
    "variantDimensions": [
      {"name": "Color", "values": ["Black", "White", "Blue"]},
      {"name": "Storage", "values": ["128GB", "256GB", "512GB"]}
    ]
  }'
```

### 2.4 Product Groups

```bash
# List all product groups
curl -X GET -u ":banana" "localhost:5000/api/v1/product-groups"

# Get product group with members
curl -X GET -u ":banana" "localhost:5000/api/v1/product-groups/com.myapp.groupId=GRP-001~with(members)"

# Get group's children (sub-groups)
curl -X GET -u ":banana" "localhost:5000/api/v1/product-groups/com.myapp.groupId=GRP-001/children"

# Create a product group
curl -X POST -u ":banana" "localhost:5000/api/v1/product-groups" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.groupId": "GRP-001"},
    "name": "Summer Collection"
  }'

# Create child group
curl -X POST -u ":banana" "localhost:5000/api/v1/product-groups" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.groupId": "GRP-002"},
    "name": "Beach Wear",
    "parent": {"identifiers": {"com.myapp.groupId": "GRP-001"}}
  }'

# Assign product to group (via parentGroup on the product, NOT group's members)
curl -X PATCH -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001" \
  -H "Content-Type: application/json" \
  -d '{"parentGroup": {"identifiers": {"com.myapp.groupId": "GRP-001"}}}'
```

### 2.5 Labels

```bash
# List all labels
curl -X GET -u ":banana" "localhost:5000/api/v1/labels"

# Get label by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/labels/com.myapp.labelId=sale"

# Create a label
curl -X POST -u ":banana" "localhost:5000/api/v1/labels" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.labelId": "sale"},
    "title": "On Sale",
    "description": "Items currently on sale",
    "color": "#FF0000"
  }'

# Create label restricted to specific types
curl -X POST -u ":banana" "localhost:5000/api/v1/labels" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.labelId": "vip-customer"},
    "title": "VIP Customer",
    "color": "#FFD700",
    "applicableOnlyTo": ["person", "company"]
  }'

# Assign label to entity
curl -X POST -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001/labels" \
  -H "Content-Type: application/json" \
  -d '{"identifiers": {"com.myapp.labelId": "sale"}}'
```

### 2.6 Images

```bash
# Get product's images
curl -X GET -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001/images"

# Add image to product
curl -X POST -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001/images" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/images/product-001.jpg",
    "altText": "Product front view"
  }'

# Add image with sort order
curl -X POST -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001/images" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/images/product-001-back.jpg",
    "altText": "Product back view",
    "sortOrder": 2
  }'
```

---

<a id="pricing-examples"></a>

## 3. Prices & Pricing

### 3.1 Prices

```bash
# List all prices
curl -X GET -u ":banana" "localhost:5000/api/v1/prices"

# Get prices for a product
curl -X GET -u ":banana" "localhost:5000/api/v1/products/com.myapp.sku=SKU-001/prices"

# Get prices with currency info
curl -X GET -u ":banana" "localhost:5000/api/v1/prices~with(currency)~take(20)"

# Create a price
curl -X POST -u ":banana" "localhost:5000/api/v1/prices" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.priceId": "PRICE-001"},
    "products": [{"identifiers": {"com.myapp.sku": "SKU-001"}}],
    "sellers": [{"identifiers": {"com.heads.seedID": "ourcompany"}}],
    "amount": 199.00,
    "currency": {"identifiers": {"currencyCode": "SEK"}}
  }'

# Create price with validity period
curl -X POST -u ":banana" "localhost:5000/api/v1/prices" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.priceId": "PRICE-002"},
    "products": [{"identifiers": {"com.myapp.sku": "SKU-001"}}],
    "sellers": [{"identifiers": {"com.heads.seedID": "ourcompany"}}],
    "amount": 149.00,
    "currency": {"identifiers": {"currencyCode": "SEK"}},
    "from": "2024-12-01T00:00:00Z",
    "to": "2024-12-31T23:59:59Z"
  }'

# Create price with buyer restriction
curl -X POST -u ":banana" "localhost:5000/api/v1/prices" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.priceId": "PRICE-003"},
    "products": [{"identifiers": {"com.myapp.sku": "SKU-001"}}],
    "sellers": [{"identifiers": {"com.heads.seedID": "ourcompany"}}],
    "buyers": [{"identifiers": {"com.myapp.customerId": "CUST-001"}}],
    "amount": 179.00,
    "currency": {"identifiers": {"currencyCode": "SEK"}}
  }'

# Update price
curl -X PATCH -u ":banana" "localhost:5000/api/v1/prices/com.myapp.priceId=PRICE-001" \
  -H "Content-Type: application/json" \
  -d '{"amount": 189.00}'

# Delete price
curl -X DELETE -u ":banana" "localhost:5000/api/v1/prices/com.myapp.priceId=PRICE-001"
```

### 3.2 Currencies

```bash
# List all currencies
curl -X GET -u ":banana" "localhost:5000/api/v1/currencies"

# Get currency by code
curl -X GET -u ":banana" "localhost:5000/api/v1/currencies/currencyCode=SEK"

# Get currency with denominations
curl -X GET -u ":banana" "localhost:5000/api/v1/currencies/currencyCode=SEK~with(denominations)"

# Create/update currency
curl -X PUT -u ":banana" "localhost:5000/api/v1/currencies/currencyCode=USD" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"currencyCode": "USD"},
    "name": "US Dollar",
    "symbol": "$",
    "decimalDigits": 2
  }'
```

### 3.3 VAT Codes

```bash
# List all VAT codes
curl -X GET -u ":banana" "localhost:5000/api/v1/vat-codes"

# Get VAT code by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/vat-codes/vatCodeId=SE25"

# Create VAT code
curl -X POST -u ":banana" "localhost:5000/api/v1/vat-codes" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"vatCodeId": "SE25"},
    "name": "Swedish VAT 25%",
    "percentage": 25.0,
    "country": {"identifiers": {"countryCode": "SE"}}
  }'
```

---

<a id="orders-examples"></a>

## 4. Trade Orders & Fulfillment

### 4.1 Trade Orders

```bash
# List all trade orders
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders"

# Get trade order by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001"

# Get trade order with items
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001~with(items)"

# Get trade order with customer and supplier
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001~with(customer,supplier)"

# Get trade order items
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/items"

# Get specific item by index
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/items/0"

# Get trade order payments
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/payments"

# Get trade order shipments
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/shipments"

# Create a sales order
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-orders" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.orderId": "ORD-2024-001"},
    "supplier": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "customer": {"identifiers": {"com.myapp.customerId": "CUST-001"}},
    "currency": {"identifiers": {"currencyCode": "SEK"}},
    "items": [
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
        "quantity": 2,
        "unitAmountInclVat": 199.00
      }
    ]
  }'

# Create purchase order (supplier and customer swapped perspective)
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-orders" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.orderId": "PO-2024-001"},
    "supplier": {"identifiers": {"com.myapp.supplierId": "SUPP-001"}},
    "customer": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "currency": {"identifiers": {"currencyCode": "SEK"}},
    "items": [
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
        "quantity": 100
      }
    ]
  }'

# Add item to existing order
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/items" \
  -H "Content-Type: application/json" \
  -d '{
    "product": {"identifiers": {"com.myapp.sku": "SKU-002"}},
    "quantity": 1,
    "unitAmountInclVat": 299.00
  }'

# Update order item quantity
curl -X PATCH -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/items/0" \
  -H "Content-Type: application/json" \
  -d '{"quantity": 3}'

# Order actions - confirm order
curl -X PATCH -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/actions" \
  -H "Content-Type: application/json" \
  -d '{"confirm": true}'

# Order actions - cancel order
curl -X PATCH -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/actions" \
  -H "Content-Type: application/json" \
  -d '{"cancel": true}'

# Delete order item
curl -X DELETE -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/items/0"

# Delete order
curl -X DELETE -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001"
```

### 4.2 Trade Relationships

```bash
# List all trade relationships
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-relationships"

# Get trade relationship by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-relationships/com.heads.seedID=rel-001"

# Get relationship with agents
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-relationships/com.heads.seedID=rel-001~with(supplierAgent,customerAgent)"

# Create a trade relationship (supplier-customer)
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-relationships" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.relId": "REL-001"},
    "supplierAgent": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "customerAgent": {"identifiers": {"com.myapp.customerId": "CUST-001"}},
    "defaultCurrency": {"identifiers": {"currencyCode": "SEK"}}
  }'

# Create relationship with payment and delivery terms
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-relationships" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.relId": "REL-002"},
    "supplierAgent": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "customerAgent": {"identifiers": {"com.myapp.companyId": "COMP-001"}},
    "defaultCurrency": {"identifiers": {"currencyCode": "SEK"}},
    "creditAllowed": true,
    "allowsBackOrder": true
  }'

# Update relationship
curl -X PATCH -u ":banana" "localhost:5000/api/v1/trade-relationships/com.myapp.relId=REL-001" \
  -H "Content-Type: application/json" \
  -d '{"creditAllowed": true}'
```

### 4.3 Shipment Orders

```bash
# List all shipment orders
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders"

# Get shipment order by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001"

# Get shipment with items
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001~with(items)"

# Get shipment's related trade orders
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001/orders"

# Get shipment records
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001/records"

# Create a shipment order
curl -X POST -u ":banana" "localhost:5000/api/v1/shipment-orders" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.shipmentId": "SHIP-001"},
    "sender": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "receiver": {"identifiers": {"com.myapp.customerId": "CUST-001"}},
    "source": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}},
    "items": [
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
        "quantity": 2
      }
    ]
  }'

# Search shipments by sender (using finder)
curl -X POST -u ":banana" "localhost:5000/api/v1/shipment-orders/find" \
  -H "Content-Type: application/json" \
  -d '{
    "sender": {"identifiers": {"com.heads.seedID": "ourcompany"}}
  }'

# Release shipment order
curl -X PATCH -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001/actions" \
  -H "Content-Type: application/json" \
  -d '{"release": true}'
```

### 4.4 Picking Orders

```bash
# List all picking orders
curl -X GET -u ":banana" "localhost:5000/api/v1/picking-orders"

# Get picking order by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/picking-orders/pickingOrderId=PICK-001"

# Create a picking order
curl -X POST -u ":banana" "localhost:5000/api/v1/picking-orders" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"pickingOrderId": "PICK-001"},
    "issuer": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "source": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}},
    "destination": {"identifiers": {"com.myapp.stockPlaceId": "STORE-001"}}
  }'
```

### 4.5 Payment Orders

```bash
# List all payment orders
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-orders"

# Get payment order by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-orders/com.myapp.paymentOrderId=PAY-001"

# Get payment orders for a trade order
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/payments"

# Create a payment order
curl -X POST -u ":banana" "localhost:5000/api/v1/payment-orders" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.paymentOrderId": "PAY-001"},
    "payer": {"identifiers": {"com.myapp.customerId": "CUST-001"}},
    "payee": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "amount": 398.00,
    "currency": {"identifiers": {"currencyCode": "SEK"}}
  }'
```

### 4.6 Trade Rules & Discounts

```bash
# List all discount rules
curl -X GET -u ":banana" "localhost:5000/api/v1/discount-rules"

# Get discount rule by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/discount-rules/com.myapp.ruleId=DISC-001"

# Create a percentage discount rule
curl -X POST -u ":banana" "localhost:5000/api/v1/discount-rules" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.ruleId": "DISC-001"},
    "name": "Summer Sale 20%",
    "discountPercentage": 20.0,
    "validFrom": "2024-06-01T00:00:00Z",
    "validTo": "2024-08-31T23:59:59Z"
  }'

# List trade restrictions
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-restrictions"

# List trade restriction reasons
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-restriction-reasons"

# Create trade restriction reason
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-restriction-reasons" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.reasonId": "AGE-RESTRICTED"},
    "name": "Age Restricted Product",
    "description": "Requires age verification before purchase"
  }'
```

---

<a id="inventory-examples"></a>

## 5. Stock & Inventory

### 5.1 Stock Places

```bash
# List all stock places
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-places"

# Get stock place by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-places/com.heads.seedID=warehouse1"

# Get stock place with children (sub-locations)
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-places/com.heads.seedID=warehouse1~with(children)"

# Get stock place entries (current stock levels)
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-places/com.heads.seedID=warehouse1/entries"

# Get stock transactions
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-places/com.heads.seedID=warehouse1/transactions"

# Create a stock place (warehouse)
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-places" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.stockPlaceId": "WH-001"},
    "name": "Main Warehouse",
    "owner": {"identifiers": {"com.heads.seedID": "ourcompany"}}
  }'

# Create child stock place (zone within warehouse)
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-places" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.stockPlaceId": "WH-001-A"},
    "name": "Zone A",
    "parent": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}}
  }'

# Update stock place
curl -X PATCH -u ":banana" "localhost:5000/api/v1/stock-places/com.myapp.stockPlaceId=WH-001" \
  -H "Content-Type: application/json" \
  -d '{"name": "Central Distribution Warehouse"}'
```

### 5.2 Stock Adjustments

```bash
# List all stock adjustments
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-adjustments"

# List stock adjustment reasons
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-adjustment-reasons"

# Create stock adjustment reason
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-adjustment-reasons" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.reasonId": "DAMAGED"},
    "name": "Damaged Goods",
    "description": "Product damaged and cannot be sold"
  }'

# Create stock adjustment (increase)
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-adjustments" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.adjustmentId": "ADJ-001"},
    "stockPlace": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}},
    "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
    "quantity": 100,
    "reason": {"identifiers": {"com.myapp.reasonId": "RESTOCK"}}
  }'

# Create stock adjustment (decrease)
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-adjustments" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.adjustmentId": "ADJ-002"},
    "stockPlace": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}},
    "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
    "quantity": -5,
    "reason": {"identifiers": {"com.myapp.reasonId": "DAMAGED"}}
  }'
```

### 5.3 Stock Resets

```bash
# List all stock resets
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-resets"

# Create stock reset (set absolute quantity)
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-resets" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.resetId": "RESET-001"},
    "stockPlace": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}},
    "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
    "quantity": 50
  }'
```

---

<a id="pos-examples"></a>

## 6. Point of Sale (POS)

### 6.1 POS Terminals

```bash
# List all POS terminals
curl -X GET -u ":banana" "localhost:5000/api/v1/pos-terminals"

# Get POS terminal by name
curl -X GET -u ":banana" "localhost:5000/api/v1/pos-terminals/posTerminalName=Kassa%201"

# Get terminal with profile
curl -X GET -u ":banana" "localhost:5000/api/v1/pos-terminals/posTerminalName=Kassa%201~with(profile)"

# Create a POS terminal
curl -X POST -u ":banana" "localhost:5000/api/v1/pos-terminals" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"posTerminalName": "Kassa 2"},
    "name": "Kassa 2",
    "profile": {"identifiers": {"posProfileId": "default"}},
    "associatedNode": {"identifiers": {"com.heads.seedID": "store1"}}
  }'

# Update POS terminal
curl -X PATCH -u ":banana" "localhost:5000/api/v1/pos-terminals/posTerminalName=Kassa%202" \
  -H "Content-Type: application/json" \
  -d '{"name": "Express Checkout"}'
```

### 6.2 POS Profiles

```bash
# List all POS profiles
curl -X GET -u ":banana" "localhost:5000/api/v1/pos-profiles"

# Get POS profile by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/pos-profiles/posProfileId=default"

# Get profile with functions
curl -X GET -u ":banana" "localhost:5000/api/v1/pos-profiles/posProfileId=default~with(functions)"

# Create a POS profile
curl -X POST -u ":banana" "localhost:5000/api/v1/pos-profiles" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"posProfileId": "quick-service"},
    "name": "Quick Service Profile",
    "defaultCurrency": {"identifiers": {"currencyCode": "SEK"}}
  }'

# Update POS profile
curl -X PATCH -u ":banana" "localhost:5000/api/v1/pos-profiles/posProfileId=quick-service" \
  -H "Content-Type: application/json" \
  -d '{"name": "Fast Food Profile"}'
```

### 6.3 POS Functions

```bash
# List all POS functions
curl -X GET -u ":banana" "localhost:5000/api/v1/pos-functions"

# Get POS function by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/pos-functions/posFunctionId=open-drawer"

# Create a POS function
curl -X POST -u ":banana" "localhost:5000/api/v1/pos-functions" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"posFunctionId": "void-sale"},
    "name": "Void Sale",
    "description": "Void the current sale"
  }'
```

### 6.4 Receipts

```bash
# List all receipts
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts"

# Get receipt by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/receiptID=RCP-2024-00001"

# Get receipt with items
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/receiptID=RCP-2024-00001~with(items)"

# Get receipt items
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/receiptID=RCP-2024-00001/items"

# Get specific receipt item
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/receiptID=RCP-2024-00001/items/0"

# Get receipt payments
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/receiptID=RCP-2024-00001/payments"

# Receipts from last 24 hours
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/after/-=24"

# Receipts from last 30 minutes
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/after/-=0:30"

# Receipts from last week with item count
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/after/-=168~with(items/count)"

# Receipts in date range
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/after/2024-12-01T00:00:00Z/before/2024-12-31T23:59:59Z"

# Map receipts to custom format
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/after/-=24~map(com.heads.receipt-csv)"

# Create a receipt (typically done by POS system)
curl -X POST -u ":banana" "localhost:5000/api/v1/receipts" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"receiptID": "RCP-2024-00002"},
    "timestamp": "2024-12-14T10:30:00Z",
    "seller": {"identifiers": {"com.heads.seedID": "store1"}},
    "terminal": {"identifiers": {"posTerminalName": "Kassa 1"}},
    "items": [
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
        "quantity": 1,
        "totalAmountInclVat": 199.00
      }
    ],
    "payments": [
      {
        "method": {"identifiers": {"methodId": "com.heads.cash"}},
        "amount": 199.00
      }
    ]
  }'
```

### 6.5 Devices

```bash
# List all devices
curl -X GET -u ":banana" "localhost:5000/api/v1/devices"

# Get device by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/devices/com.myapp.deviceId=DEV-001"

# Get device with role assignments
curl -X GET -u ":banana" "localhost:5000/api/v1/devices/com.myapp.deviceId=DEV-001~with(roleAssignments)"

# Create a device
curl -X POST -u ":banana" "localhost:5000/api/v1/devices" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.deviceId": "DEV-001"},
    "name": "Store 1 Tablet",
    "status": "Active"
  }'

# Update device status
curl -X PATCH -u ":banana" "localhost:5000/api/v1/devices/com.myapp.deviceId=DEV-001" \
  -H "Content-Type: application/json" \
  -d '{"status": "Inactive"}'
```

### 6.6 Device Roles

```bash
# List all device roles
curl -X GET -u ":banana" "localhost:5000/api/v1/device-roles"

# Get device role by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/device-roles/deviceRoleId=pos-terminal"

# Create a device role
curl -X POST -u ":banana" "localhost:5000/api/v1/device-roles" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"deviceRoleId": "backoffice"},
    "name": "Backoffice Device"
  }'
```

### 6.7 Printers

```bash
# List all printers
curl -X GET -u ":banana" "localhost:5000/api/v1/printers"

# Get printer by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/printers/com.heads.seedID=receipt-printer-1"

# Create Star WebPRNT printer
curl -X POST -u ":banana" "localhost:5000/api/v1/star-webPRNT-printers" \
  -H "Content-Type: application/json" \
  -d '{
    "@type": "star webPRNT printer",
    "identifiers": {"com.myapp.printerId": "STAR-001"},
    "name": "Receipt Printer 1",
    "url": "http://192.168.1.100:8080/StarWebPRNT/SendMessage",
    "associatedNode": {"identifiers": {"com.heads.seedID": "store1"}}
  }'

# Create Epson ePOS printer
curl -X POST -u ":banana" "localhost:5000/api/v1/epson-ePOS-printers" \
  -H "Content-Type: application/json" \
  -d '{
    "@type": "epson ePOS printer",
    "identifiers": {"com.myapp.printerId": "EPSON-001"},
    "name": "Kitchen Printer",
    "url": "http://192.168.1.101:8080",
    "deviceID": "local_printer",
    "associatedNode": {"identifiers": {"com.heads.seedID": "store1"}}
  }'

# Create serial printer
curl -X POST -u ":banana" "localhost:5000/api/v1/serial-printers" \
  -H "Content-Type: application/json" \
  -d '{
    "@type": "serial printer",
    "identifiers": {"com.myapp.printerId": "SERIAL-001"},
    "name": "Label Printer",
    "port": "COM1",
    "baudRate": 9600,
    "dataBits": 8,
    "stopBits": 1,
    "parity": "None"
  }'
```

### 6.8 Payment Terminals

```bash
# List all payment terminals
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-terminals"

# Get payment terminal by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-terminals/com.myapp.terminalId=TERM-001"

# Create a payment terminal
curl -X POST -u ":banana" "localhost:5000/api/v1/payment-terminals" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.terminalId": "TERM-001"},
    "name": "Card Terminal 1",
    "associatedNode": {"identifiers": {"com.heads.seedID": "store1"}}
  }'
```

---

<a id="users-examples"></a>

## 7. Users & Authentication

### 7.1 Users

```bash
# List all users
curl -X GET -u ":banana" "localhost:5000/api/v1/users"

# Get user by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/users/com.heads.seedID=admin"

# Get user with role assignments
curl -X GET -u ":banana" "localhost:5000/api/v1/users/com.heads.seedID=admin~with(roleAssignments)"

# Get user's OAuth2 clients
curl -X GET -u ":banana" "localhost:5000/api/v1/users/com.heads.seedID=admin/oauth2Clients"

# Create a user
curl -X POST -u ":banana" "localhost:5000/api/v1/users" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.userId": "USER-001"},
    "name": "John Cashier"
  }'

# Update user
curl -X PATCH -u ":banana" "localhost:5000/api/v1/users/com.myapp.userId=USER-001" \
  -H "Content-Type: application/json" \
  -d '{"name": "John Smith"}'

# Delete user
curl -X DELETE -u ":banana" "localhost:5000/api/v1/users/com.myapp.userId=USER-001"
```

### 7.2 User Roles

```bash
# List all user roles
curl -X GET -u ":banana" "localhost:5000/api/v1/user-roles"

# Get user role by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/user-roles/userRoleID=admin"

# Create a user role
curl -X POST -u ":banana" "localhost:5000/api/v1/user-roles" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"userRoleID": "cashier"},
    "name": "Cashier",
    "description": "Basic cashier role with POS access"
  }'

# Update user role
curl -X PATCH -u ":banana" "localhost:5000/api/v1/user-roles/userRoleID=cashier" \
  -H "Content-Type: application/json" \
  -d '{"description": "Cashier role with POS and returns access"}'
```

### 7.3 Role Assignments

```bash
# Get user's role assignments
curl -X GET -u ":banana" "localhost:5000/api/v1/users/com.myapp.userId=USER-001/roleAssignments"

# Assign role to user
curl -X POST -u ":banana" "localhost:5000/api/v1/users/com.myapp.userId=USER-001/roleAssignments" \
  -H "Content-Type: application/json" \
  -d '{
    "role": {"identifiers": {"userRoleID": "cashier"}},
    "scope": {"identifiers": {"com.heads.seedID": "store1"}}
  }'

# Remove role assignment
curl -X DELETE -u ":banana" "localhost:5000/api/v1/users/com.myapp.userId=USER-001/roleAssignments/key=assignment-key"
```

### 7.4 OAuth2 Clients

```bash
# Get user's OAuth2 clients
curl -X GET -u ":banana" "localhost:5000/api/v1/users/com.heads.seedID=admin/oauth2Clients"

# Create OAuth2 client for user
curl -X POST -u ":banana" "localhost:5000/api/v1/users/com.heads.seedID=admin/oauth2Clients" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"clientID": "my-integration-client"},
    "redirectURIs": ["https://myapp.example.com/callback"],
    "scopes": ["read", "write"],
    "isConfidential": true
  }'

# Delete OAuth2 client
curl -X DELETE -u ":banana" "localhost:5000/api/v1/users/com.heads.seedID=admin/oauth2Clients/key=client-key"
```

---

<a id="configuration-examples"></a>

## 8. Configuration & Reference Data

### 8.1 Countries & Geography

```bash
# List all countries
curl -X GET -u ":banana" "localhost:5000/api/v1/countries"

# Get country by code
curl -X GET -u ":banana" "localhost:5000/api/v1/countries/countryCode=SE"

# Get country with cities
curl -X GET -u ":banana" "localhost:5000/api/v1/countries/countryCode=SE~with(cities)"

# List all cities
curl -X GET -u ":banana" "localhost:5000/api/v1/cities"

# Create/update country
curl -X PUT -u ":banana" "localhost:5000/api/v1/countries/countryCode=NO" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"countryCode": "NO"},
    "name": "Norway"
  }'
```

### 8.2 Languages

```bash
# List all languages
curl -X GET -u ":banana" "localhost:5000/api/v1/languages"

# Get language by code
curl -X GET -u ":banana" "localhost:5000/api/v1/languages/languageCode=sv"

# Create/update language
curl -X PUT -u ":banana" "localhost:5000/api/v1/languages/languageCode=nb" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"languageCode": "nb"},
    "name": "Norwegian Bokmål"
  }'
```

### 8.3 Templates

```bash
# List all templates
curl -X GET -u ":banana" "localhost:5000/api/v1/templates"

# Get template by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/templates/templateId=default-receipt"

# Create a template
curl -X POST -u ":banana" "localhost:5000/api/v1/templates" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"templateId": "custom-receipt"},
    "name": "Custom Receipt Template",
    "text": "RECEIPT\n========\n{{items}}\n--------\nTotal: {{total}}"
  }'

# Update template
curl -X PATCH -u ":banana" "localhost:5000/api/v1/templates/templateId=custom-receipt" \
  -H "Content-Type: application/json" \
  -d '{"text": "RECEIPT v2\n========\n{{items}}\n--------\nTotal: {{total}}\nThank you!"}'
```

### 8.4 Mapped Types

```bash
# List all mapped types
curl -X GET -u ":banana" "localhost:5000/api/v1/mapped-types"

# Get mapped type by name
curl -X GET -u ":banana" "localhost:5000/api/v1/mapped-types/mappedTypeName=com.heads.receipt-csv"

# Create a mapped type
curl -X POST -u ":banana" "localhost:5000/api/v1/mapped-types" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"mappedTypeName": "com.myapp.product-export"},
    "body": {
      "sku": "identifiers/com.myapp.sku",
      "name": "name",
      "price": "prices~first/amount"
    }
  }'

# Use mapped type in query
curl -X GET -u ":banana" "localhost:5000/api/v1/products~map(com.myapp.product-export)~take(10)"

# Map a receipts bundle (default mapped type)
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~map(com.heads.receipts-zip)"

# Map request bodies on write
curl -X PUT -u ":banana" "localhost:5000/api/v1/products" \
  -H "Content-Type: application/json" \
  -H "X-Request-Map: com.myapp.product-import" \
  -d '[{"sku":"P-001","title":"Mapped Product","state":"Active"}]'
```

### 8.5 Payment Methods

```bash
# List all payment methods
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-methods"

# Get payment method by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-methods/methodId=com.heads.cash"

# Create a payment method
curl -X POST -u ":banana" "localhost:5000/api/v1/payment-methods" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"methodId": "com.myapp.gift-card"},
    "name": "Gift Card",
    "description": "Store gift card payment"
  }'

# Update payment method
curl -X PATCH -u ":banana" "localhost:5000/api/v1/payment-methods/methodId=com.myapp.gift-card" \
  -H "Content-Type: application/json" \
  -d '{"name": "Store Gift Card"}'
```

### 8.6 Discount & Return Reasons

```bash
# List discount reasons
curl -X GET -u ":banana" "localhost:5000/api/v1/discount-reasons"

# Create discount reason
curl -X POST -u ":banana" "localhost:5000/api/v1/discount-reasons" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.reasonId": "EMPLOYEE"},
    "name": "Employee Discount",
    "description": "Discount for store employees"
  }'

# List return reasons
curl -X GET -u ":banana" "localhost:5000/api/v1/return-reasons"

# Create return reason
curl -X POST -u ":banana" "localhost:5000/api/v1/return-reasons" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.reasonId": "DEFECTIVE"},
    "name": "Defective Product",
    "description": "Product is defective or damaged"
  }'
```

### 8.7 Delivery & Payment Terms

```bash
# List delivery terms
curl -X GET -u ":banana" "localhost:5000/api/v1/delivery-terms"

# Create incoterm delivery term
curl -X POST -u ":banana" "localhost:5000/api/v1/incoterm-delivery-terms" \
  -H "Content-Type: application/json" \
  -d '{
    "@type": "incoterm delivery term",
    "identifiers": {"com.myapp.termId": "FOB-STOCKHOLM"},
    "incotermCode": "FOB",
    "location": {
      "name": "Stockholm Port"
    }
  }'

# List payment terms
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-terms"

# Create payment term
curl -X POST -u ":banana" "localhost:5000/api/v1/payment-terms" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.termId": "NET30"},
    "name": "Net 30",
    "description": "Payment due within 30 days"
  }'
```

### 8.8 Sales Channels

```bash
# List all sales channels
curl -X GET -u ":banana" "localhost:5000/api/v1/sales-channels"

# Get sales channel by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/sales-channels/com.myapp.channelId=webshop"

# Get sales channel products
curl -X GET -u ":banana" "localhost:5000/api/v1/sales-channels/com.myapp.channelId=webshop/products"

# Create a sales channel
curl -X POST -u ":banana" "localhost:5000/api/v1/sales-channels" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.channelId": "webshop"},
    "name": "Online Webshop"
  }'
```

### 8.9 Customer Groups

```bash
# List all customer groups
curl -X GET -u ":banana" "localhost:5000/api/v1/customer-groups"

# Get customer group by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/customer-groups/com.myapp.groupId=vip"

# Get customer group members
curl -X GET -u ":banana" "localhost:5000/api/v1/customer-groups/com.myapp.groupId=vip/members"

# Create a customer group
curl -X POST -u ":banana" "localhost:5000/api/v1/customer-groups" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.groupId": "vip"},
    "name": "VIP Customers",
    "owner": {"identifiers": {"com.heads.seedID": "ourcompany"}}
  }'
```

### 8.10 Sync Webhooks

```bash
# List all sync webhooks
curl -X GET -u ":banana" "localhost:5000/api/v1/sync-webhooks"

# Get sync webhook by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/sync-webhooks/com.myapp.webhookId=product-sync"

# Create a sync webhook
curl -X POST -u ":banana" "localhost:5000/api/v1/sync-webhooks" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.webhookId": "product-sync"},
    "name": "Product Sync to ERP",
    "description": "Syncs new products to external ERP system",
    "when": "api/v1/now/0_0_*_*_*",
    "repeat": true,
    "in": {
      "method": "GET",
      "url": "api/v1/products~where(status=Active)~take(100)"
    },
    "out": {
      "method": "POST",
      "url": "https://erp.example.com/api/products",
      "auth": {
        "basic": {
          "username": "api-user",
          "password": "api-secret"
        }
      }
    }
  }'

# Update sync webhook
curl -X PATCH -u ":banana" "localhost:5000/api/v1/sync-webhooks/com.myapp.webhookId=product-sync" \
  -H "Content-Type: application/json" \
  -d '{"repeat": false}'
```

### 8.11 Shortened Links

```bash
# List all shortened links
curl -X GET -u ":banana" "localhost:5000/api/v1/shortened-links"

# Create shortened link
curl -X POST -u ":banana" "localhost:5000/api/v1/shortened-links" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.linkId": "promo-2024"},
    "targetUrl": "https://shop.example.com/promotions/summer-2024"
  }'
```

### 8.12 Config API (System Settings)

```bash
# Get API config (singleton)
curl -X GET -u ":banana" "localhost:5000/api/v1/config/api"

# Update API config
curl -X PUT -u ":banana" "localhost:5000/api/v1/config/api" \
  -H "Content-Type: application/json" \
  -d '{
    "publicBaseURL": "https://api.example.com",
    "verboseLogging": true,
    "requestLogPath": "/var/log/commerceos/api"
  }'

# Get webshop config
curl -X GET -u ":banana" "localhost:5000/api/v1/config/webshop"

# Update webshop config
curl -X PUT -u ":banana" "localhost:5000/api/v1/config/webshop" \
  -H "Content-Type: application/json" \
  -d '{
    "siteURL": "https://shop.example.com",
    "siteTitle": {"en": "Example Shop"},
    "paymentMethods": []
  }'
```

### 8.13 EPI Integrations & Configurations

```bash
# 1) Create a payment integration (name + baseUrl required)
curl -X POST -u ":banana" "localhost:5000/api/v1/payment-integrations" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"name": "Mock"},
    "baseUrl": "https://mock-payment.example.com"
  }'

# 2) Create OAuth2 client/user for the integration (manual step outside this example)
# (Required before install)

# 3) Install the integration
curl -X PATCH -u ":banana" "localhost:5000/api/v1/epi-integrations/name=Mock" \
  -H "Content-Type: application/json" \
  -d '{"install": true}'

# 4) Create EPI configuration for a specific node
curl -X POST -u ":banana" "localhost:5000/api/v1/epi-configurations" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.configId": "company-mock-config"},
    "integration": {"identifiers": {"name": "Mock"}},
    "node": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "configuration": {
      "message": "You are Company Mock!"
    }
  }'

# 5) Configure the integration for first-time setup
curl -X PATCH -u ":banana" "localhost:5000/api/v1/epi-configurations/com.myapp.configId=company-mock-config" \
  -H "Content-Type: application/json" \
  -d '{"configure": true}'
```

---

<a id="operators-examples"></a>

## 9. Query Operators Reference

### 9.1 Collection Operators

```bash
# ~take(n) - Limit results
curl -X GET -u ":banana" "localhost:5000/api/v1/products~take(10)"

# ~skip(n) - Skip first n results
curl -X GET -u ":banana" "localhost:5000/api/v1/products~skip(20)~take(10)"

# ~first - Get first element (NO parentheses!)
curl -X GET -u ":banana" "localhost:5000/api/v1/products~first"

# ~last - Get last element (NO parentheses!)
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~last"

# ~count - Count elements (NO parentheses!)
curl -X GET -u ":banana" "localhost:5000/api/v1/products~count"

# ~distinct - Unique primitive values (NO parentheses!)
curl -X GET -u ":banana" "localhost:5000/api/v1/products~map(status)~distinct"

# ~distinctBy(field) - Unique objects by field
curl -X GET -u ":banana" "localhost:5000/api/v1/products~distinctBy(status)~take(10)"

# ~flat - Flatten nested arrays (NO parentheses!)
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~map(items)~flat~take(100)"

# ~orderBy(field) - Sort ascending
curl -X GET -u ":banana" "localhost:5000/api/v1/products~orderBy(name)"

# ~orderBy(field:desc) - Sort descending
curl -X GET -u ":banana" "localhost:5000/api/v1/products~orderBy(name:desc)"
```

### 9.2 Projection Operators

```bash
# ~just(...) - Select specific fields
curl -X GET -u ":banana" "localhost:5000/api/v1/products~just(name,identifiers)"

# ~just with path aliasing
curl -X GET -u ":banana" "localhost:5000/api/v1/products~just(sku:identifiers/key,productName:name)"

# ~with(...) - Include related entities
curl -X GET -u ":banana" "localhost:5000/api/v1/products~with(prices,categories)"

# ~with nested take
curl -X GET -u ":banana" "localhost:5000/api/v1/products~with(prices~take(3))~take(10)"

# ~without(...) - Exclude fields
curl -X GET -u ":banana" "localhost:5000/api/v1/agents~without(addresses)"
```

### 9.3 Filter Operators

```bash
# ~where(condition) - Filter by condition
curl -X GET -u ":banana" "localhost:5000/api/v1/products~where(status=Active)"

# Equality check
curl -X GET -u ":banana" "localhost:5000/api/v1/agents~where(@type=person)"

# Not equal
curl -X GET -u ":banana" "localhost:5000/api/v1/products~where(status!=Inactive)"

# Pattern match (contains)
curl -X GET -u ":banana" "localhost:5000/api/v1/products~where(name=~iPhone)"

# Not contains
curl -X GET -u ":banana" "localhost:5000/api/v1/products~where(name!~test)"

# Greater than / less than
curl -X GET -u ":banana" "localhost:5000/api/v1/prices~where(amount>100)"
curl -X GET -u ":banana" "localhost:5000/api/v1/prices~where(amount<=50)"

# Truthy check (field exists and has value)
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-relationships~where(creditAllowed)"

# Falsy check
curl -X GET -u ":banana" "localhost:5000/api/v1/products~where(!hidden)"

# Nested path filter
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders~where(customer/@type=person)"
```

### 9.4 Transformation Operators

```bash
# ~map(mappedType) - Transform using mapped type
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~map(com.heads.receipt-csv)"

# ~map(field) - Extract field from each element
curl -X GET -u ":banana" "localhost:5000/api/v1/products~map(name)"

# ~typeless - Remove @type annotations (NO parentheses!)
curl -X GET -u ":banana" "localhost:5000/api/v1/products~first~typeless"

# ~entries - Convert object to array of {key, value}
curl -X GET -u ":banana" "localhost:5000/api/v1/agents/com.heads.seedID=ourcompany/addresses~entries"
```

---

<a id="advanced-examples"></a>

## 10. Advanced Query Patterns

### 10.1 Complex Filtering

```bash
# Filter with multiple conditions (AND logic via chaining)
curl -X GET -u ":banana" "localhost:5000/api/v1/products~where(status=Active)~where(name=~Premium)~take(10)"

# Filter by nested property
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders~where(supplier/name=~Acme)"

# Filter by count
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~where(items~count>5)~take(10)"
```

### 10.2 Deep Path Navigation

```bash
# Navigate through relationships
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-001/customer/addresses/main"

# Get nested count
curl -X GET -u ":banana" "localhost:5000/api/v1/product-categories/com.heads.seedID=electronics/members~count"

# Deep with clause
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders~with(items~with(product~just(name)))~take(5)"
```

### 10.3 Aggregations

```bash
# Count by status
curl -X GET -u ":banana" "localhost:5000/api/v1/products~where(status=Active)~count"

# Get distinct statuses
curl -X GET -u ":banana" "localhost:5000/api/v1/products~map(status)~distinct"

# Get one product per status
curl -X GET -u ":banana" "localhost:5000/api/v1/products~distinctBy(status)~take(10)"

# Count items per receipt
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~just(receiptId:identifiers/receiptID,itemCount:items~count)~take(20)"
```

### 10.4 Path Aliasing

```bash
# Simple alias
curl -X GET -u ":banana" "localhost:5000/api/v1/products~just(sku:identifiers/key,productName:name)"

# Nested alias
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders~just(orderId:identifiers/key,customerName:customer/name,supplierName:supplier/name)~take(10)"

# Alias with aggregation
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~just(id:identifiers/receiptID,items:items~count,total:totalAmount)"
```

### 10.5 Body Indicators

Body indicators (`@`) mark where the request body is targeted in the URL path.

```http
PUT /v1/agents/@find
Content-Type: application/json

{ "familyName": "Smith" }
```

Resources with find endpoints:
- `/v1/agents/find`
- `/v1/trade-orders/find`
- `/v1/receipts/find`
- `/v1/payment-orders/find`
- `/v1/shipment-orders/find`
- `/v1/trade-relationships/find`

### 10.6 Mapped Type Patterns

```bash
# Create a mapped type using variables and null coalescing
curl -X POST -u ":banana" "localhost:5000/api/v1/mapped-types" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"mappedTypeName": "com.myapp.receipt-summary"},
    "body": {
      "$buyer": "buyer",
      "receiptId": "identifiers/receiptID",
      "buyerName": "$buyer/name ?? ''",
      "buyerKey": "$buyer/identifiers/key",
      "total": "totalAmount/str"
    }
  }'

# Apply the mapped type to a collection
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~map(com.myapp.receipt-summary)~take(5)"

# Bundle receipts + items with @merge and $prior
curl -X POST -u ":banana" "localhost:5000/api/v1/mapped-types" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"mappedTypeName": "com.myapp.receipt-bundle"},
    "body": {
      "@merge": true,
      "receipts": "$prior~map(com.heads.receipt-csv)",
      "items": "$prior~just(items~map(com.heads.receipt-item-csv))/*items~flat"
    }
  }'

# Use the merged bundle (returns a single object)
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~map(com.myapp.receipt-bundle)"
```

### 10.7 Bulk Operations (NDJSON)

```bash
# Bulk create products (NDJSON format)
curl -X PUT -u ":banana" "localhost:5000/api/v1/products" \
  -H "Content-Type: application/x-ndjson" \
  --data-binary @- << 'EOF'
{"identifiers":{"com.myapp.sku":"BULK-001"},"name":"Bulk Product 1","status":"Active"}
{"identifiers":{"com.myapp.sku":"BULK-002"},"name":"Bulk Product 2","status":"Active"}
{"identifiers":{"com.myapp.sku":"BULK-003"},"name":"Bulk Product 3","status":"Active"}
EOF

# Bulk create prices
curl -X PUT -u ":banana" "localhost:5000/api/v1/prices" \
  -H "Content-Type: application/x-ndjson" \
  --data-binary @- << 'EOF'
{"identifiers":{"com.myapp.priceId":"PRICE-BULK-001"},"products":[{"identifiers":{"com.myapp.sku":"BULK-001"}}],"sellers":[{"identifiers":{"com.heads.seedID":"ourcompany"}}],"amount":99.00,"currency":{"identifiers":{"currencyCode":"SEK"}}}
{"identifiers":{"com.myapp.priceId":"PRICE-BULK-002"},"products":[{"identifiers":{"com.myapp.sku":"BULK-002"}}],"sellers":[{"identifiers":{"com.heads.seedID":"ourcompany"}}],"amount":149.00,"currency":{"identifiers":{"currencyCode":"SEK"}}}
EOF
```

### 10.8 Query Parameter Alternatives

```bash
# Filter and limit with query params
curl -X GET -u ":banana" "localhost:5000/api/v1/products?status=Active&limit=3"

# Sort and paginate with query params
curl -X GET -u ":banana" "localhost:5000/api/v1/products?orderby=name&offset=2&limit=2"

# Sort descending with query params
curl -X GET -u ":banana" "localhost:5000/api/v1/products?orderby=name:desc&limit=3"

# Request all fields
curl -X GET -u ":banana" "localhost:5000/api/v1/products?fields=all&limit=1"
```

### 10.9 System Resources

```bash
# Get current timestamp
curl -X GET -u ":banana" "localhost:5000/api/v1/now"

# Generate UUID
curl -X GET -u ":banana" "localhost:5000/api/v1/uuid"

# Connection test (void)
curl -X GET -u ":banana" "localhost:5000/api/v1/void"
```

### 10.10 Schema Introspection

Use the `/properties` endpoint to discover available fields on any resource type. This is useful for understanding what properties can be read, written, or expanded.

```bash
# Get properties schema for products
curl -X GET -u ":banana" "localhost:5000/api/v1/products/properties"

# Get properties schema for agents
curl -X GET -u ":banana" "localhost:5000/api/v1/agents/properties"

# Get properties schema for product-categories
curl -X GET -u ":banana" "localhost:5000/api/v1/product-categories/properties"

# Get properties schema for receipts
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts/properties"

# Get properties schema for trade-orders
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/properties"

# Get properties schema for prices
curl -X GET -u ":banana" "localhost:5000/api/v1/prices/properties"

# Get properties schema for payment methods
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-methods/properties"
```

The response includes metadata about each property:
- Property names and types
- Whether fields are required or optional
- Which fields are read-only
- Available sub-collections and relationships

---

### 10.11 Metrics: Sync Webhooks

Example metrics emitted per webhook run using the webhook `logName`:

```text
cos_sync_webhook_runs_total{webhook_log_name="WebHook 'product-sync'",status="success"} 12
cos_sync_webhook_runs_total{webhook_log_name="WebHook 'product-sync'",status="failure"} 2
cos_sync_webhook_retries_total{webhook_log_name="WebHook 'product-sync'"} 1
cos_sync_webhook_attempts_exhausted_total{webhook_log_name="WebHook 'product-sync'"} 0
cos_sync_webhook_last_success_timestamp_ms{webhook_log_name="WebHook 'product-sync'"} 1734514825012
cos_sync_webhook_last_failure_timestamp_ms{webhook_log_name="WebHook 'product-sync'"} 1734514750123
cos_sync_webhook_duration_ms_bucket{webhook_log_name="WebHook 'product-sync'",le="1000"} 10
cos_sync_webhook_duration_ms_bucket{webhook_log_name="WebHook 'product-sync'",le="+Inf"} 14
```

Notes:
- Labels use `logName` for human-readable identification.
- Do not include URLs, secrets, or error messages in labels.
- Cardinality of 10-30 per tenant is acceptable.

## 11. SQL Export Serialization

The API supports SQL-compatible output serialization for exporting data to MSSQL Server and other SQL databases. SQL support is implemented as serializers that consume an intermediary format.

> **Specification:** See [api-to-sql.md](./api-to-sql.md) for the complete SQL export specification.

### 11.1 SqlStatement Intermediary Format

The SQL serializer consumes an array of statement objects representing SQL operations:

```typescript
type SqlStatement =
  | { "DELETE FROM": string; "WHERE": Record<string, unknown[]> }
  | { "INSERT INTO": string; "VALUES": Record<string, unknown>[] }
  | { "MERGE INTO": string; "ON": string[]; "VALUES": Record<string, unknown>[] }
  | { "UPDATE": string; "SET": Record<string, unknown>; "WHERE": Record<string, unknown[]> }
  | { "GO": true }
  | { "--": string };
```

### 11.2 Statement Examples

**DELETE Statement:**
```javascript
// Input
{ "DELETE FROM": "HeadsReceipt", "WHERE": { "ReceiptObjectID": [1001, 1002] } }

// Output
DELETE FROM [HeadsReceipt] WHERE [ReceiptObjectID] IN (1001, 1002)
```

**INSERT Statement:**
```javascript
// Input
{ "INSERT INTO": "HeadsReceipt", "VALUES": [
  { ReceiptObjectID: 1001, SellerID: "S001", Total: 1250.00 },
  { ReceiptObjectID: 1002, SellerID: "S001", Total: 890.50 }
]}

// Output
INSERT INTO [HeadsReceipt] ([ReceiptObjectID], [SellerID], [Total])
VALUES (1001, N'S001', 1250.00), (1002, N'S001', 890.50)
```

**MERGE Statement (Upsert):**
```javascript
// Input
{ "MERGE INTO": "HeadsReceipt", "ON": ["ReceiptObjectID"], "VALUES": [
  { ReceiptObjectID: 1001, SellerID: "S001", Total: 1250.00 }
]}

// Output
MERGE INTO [HeadsReceipt] AS t
USING (VALUES (1001, N'S001', 1250.00)) AS s ([ReceiptObjectID], [SellerID], [Total])
ON t.[ReceiptObjectID] = s.[ReceiptObjectID]
WHEN MATCHED THEN UPDATE SET t.[SellerID] = s.[SellerID], t.[Total] = s.[Total]
WHEN NOT MATCHED THEN INSERT ([ReceiptObjectID], [SellerID], [Total])
  VALUES (s.[ReceiptObjectID], s.[SellerID], s.[Total]);
```

**UPDATE Statement:**
```javascript
// Input
{ "UPDATE": "HeadsReceipt", "SET": { "Total": 1300.00 }, "WHERE": { "ReceiptObjectID": [1001] } }

// Output
UPDATE [HeadsReceipt] SET [Total] = 1300.00 WHERE [ReceiptObjectID] IN (1001)
```

**Batch Separator:**
```javascript
// Input
{ "GO": true }

// Output
GO
```

**Comment:**
```javascript
// Input
{ "--": "Exported from CommerceOS API" }

// Output
-- Exported from CommerceOS API
```

### 11.3 Delete-then-Insert Pattern (Sync Mode)

For synchronizing data to an external database, use the delete-then-insert pattern:

```javascript
[
  { "--": "Sync HeadsReceipt" },
  { "DELETE FROM": "HeadsReceipt", "WHERE": { "ReceiptObjectID": [1001, 1002] } },
  { "INSERT INTO": "HeadsReceipt", "VALUES": [
    { ReceiptObjectID: 1001, SellerID: "S001", Total: 1250.00 },
    { ReceiptObjectID: 1002, SellerID: "S001", Total: 890.50 }
  ]},
  { "GO": true },
  { "--": "Sync HeadsProductReceiptItem" },
  { "DELETE FROM": "HeadsProductReceiptItem", "WHERE": { "ReceiptObjectID": [1001, 1002] } },
  { "INSERT INTO": "HeadsProductReceiptItem", "VALUES": [
    { ReceiptItemObjectID: 10001, ReceiptObjectID: 1001, ProductID: "SKU-001", Quantity: 2 },
    { ReceiptItemObjectID: 10002, ReceiptObjectID: 1001, ProductID: "SKU-002", Quantity: 1 }
  ]},
  { "GO": true }
]
```

### 11.4 Type Inference

The serializer infers SQL types from JavaScript values:

| JS Type | SQL Output |
|---------|------------|
| `string` | `N'escaped value'` |
| `number` (integer) | `123` |
| `number` (decimal) | `123.4500` |
| `boolean` | `1` or `0` |
| `null` / `undefined` | `NULL` |
| `Date` | `'2024-12-18T10:30:00.000'` |
| `bigint` | `9007199254740993` |

### 11.5 Content Negotiation

The SQL serializer accepts `SqlStatement[]` as input, not raw API objects. To produce SQL output, you must use a **mapped type** that transforms data into the `SqlStatement[]` intermediary format.

**Important:** Raw API resources like `/receipts` return receipt objects, not SQL statements. Requesting `Accept: application/sql` against raw resources will fail because the serializer expects `SqlStatement[]` input.

**Built-in SQL Mapped Types:**

The following SQL-producing mapped types are available in `seed/default-mapped-types/seed-config.json`:

| Mapped Type | Description |
|-------------|-------------|
| `com.heads.receipt-sql-row` | Flattens receipt to SQL row columns |
| `com.heads.receipt-sql-insert` | Produces INSERT statement for receipts |
| `com.heads.receipt-sql-delete` | Produces DELETE statement for receipts |
| `com.heads.receipt-sql-merge` | Produces MERGE statement (upsert) |
| `com.heads.receipt-sql-sync` | Produces DELETE + INSERT sync pattern |

**Usage examples:**

```bash
# INSERT statement (single statement with all rows)
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~take(10)~map(com.heads.receipt-sql-insert)" \
  -H "Accept: application/sql"

# DELETE statement (deletes by ReceiptObjectID)
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~take(10)~map(com.heads.receipt-sql-delete)" \
  -H "Accept: application/sql"

# MERGE statement (upsert by ReceiptObjectID)
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~take(10)~map(com.heads.receipt-sql-merge)" \
  -H "Accept: application/sql"
```

**CSV format (works with flat objects):**

The CSV serializer (`application/vnd.ms-sqlserver.csv`) accepts flat object arrays and produces RFC 4180 CSV. This can work with existing CSV-producing mapped types:

```bash
# CSV export using existing mapped type
curl -X GET -u ":banana" "localhost:5000/api/v1/receipts~take(10)~map(com.heads.receipt-csv)" \
  -H "Accept: application/vnd.ms-sqlserver.csv"
```

**Creating custom SQL-producing mapped types:**

SQL mapped types use the `@merge` pattern to collect multiple items and emit statement objects.

**Example: INSERT mapped type**
```json
{
  "identifiers": {"mappedTypeName": "com.heads.receipt-sql-insert"},
  "active": true,
  "body": {
    "@merge": true,
    "INSERT INTO": "@dbo.HeadsReceipt",
    "VALUES": "$prior~map(com.heads.receipt-sql-row)"
  }
}
```

**Example: Row mapper for VALUES**
```json
{
  "identifiers": {"mappedTypeName": "com.heads.receipt-sql-row"},
  "active": true,
  "body": {
    "ReceiptObjectID": "objectID",
    "ReceiptID": "identifiers/receiptID",
    "SellerID": "seller/identifiers/sellerId",
    "SellerName": "seller/name",
    "Timestamp": "timestamp",
    "CurrencyCode": "currencyCode",
    "TotalAmount": "totalAmount/num"
  }
}
```

**Example: DELETE mapped type**
```json
{
  "identifiers": {"mappedTypeName": "com.heads.receipt-sql-delete"},
  "active": true,
  "body": {
    "@merge": true,
    "DELETE FROM": "@dbo.HeadsReceipt",
    "WHERE": {
      "ReceiptObjectID": "$prior~just(objectID)/*objectID"
    }
  }
}
```

**Example: MERGE (upsert) mapped type**
```json
{
  "identifiers": {"mappedTypeName": "com.heads.receipt-sql-merge"},
  "active": true,
  "body": {
    "@merge": true,
    "MERGE INTO": "@dbo.HeadsReceipt",
    "ON": ["@ReceiptObjectID"],
    "VALUES": "$prior~map(com.heads.receipt-sql-row)"
  }
}
```

Key patterns:
- `@merge`: Collects all items into a single statement
- `@value`: Creates literal string values (e.g., `@dbo.HeadsReceipt`)
- `$prior~map(...)`: References the parent collection and maps each item
- `$prior~just(field)/*field`: Collects all values of a field into an array

### 11.6 SQL Server CSV Format

The `application/vnd.ms-sqlserver.csv` content type produces RFC 4180 compliant CSV optimized for SQL Server BULK INSERT:

```csv
ReceiptObjectID,SellerID,SellerName,ReceiptDate,Total,Currency
1001,"S001","Store Stockholm","2024-12-18T10:30:00.000",1250.00,"SEK"
1002,"S001","Store Stockholm","2024-12-18T11:45:00.000",890.50,"SEK"
```

Import into SQL Server:
```sql
BULK INSERT [target_table]
FROM 'data.csv'
WITH (FORMAT = 'CSV', FIELDQUOTE = '"', FIRSTROW = 2, CODEPAGE = '65001');
```

### 11.7 Security

All identifiers and string values are properly escaped:

- **String values:** Single quotes escaped with `''`, wrapped in `N'...'` for Unicode
- **Identifiers:** Wrapped in `[brackets]`, closing brackets doubled `]]`
- **SQL injection protection:** The system generates SQL output but does NOT execute it

---

## Quick Reference

### HTTP Methods

| Method | Purpose | Example |
|--------|---------|---------|
| GET | Read | `GET /products` |
| POST | Create | `POST /products {...}` |
| PUT | Upsert (create or replace) | `PUT /products/key=123 {...}` |
| PATCH | Partial update | `PATCH /products/key=123 {"name": "..."}` |
| DELETE | Remove | `DELETE /products/key=123` |

### Parameterless Operators (NO parentheses!)

| Operator | Purpose |
|----------|---------|
| `~first` | First element |
| `~last` | Last element |
| `~count` | Count elements |
| `~distinct` | Unique primitive values (use after `~map`) |
| `~flat` | Flatten arrays |
| `~typeless` | Remove @type |

### Common Identifier Patterns

| Resource | Identifier Pattern |
|----------|-------------------|
| Products | `com.myapp.sku=SKU-001` |
| People | `com.myapp.customerId=CUST-001` |
| Companies | `com.myapp.companyId=COMP-001` |
| Stores | `com.myapp.storeId=STORE-001` |
| Orders | `com.myapp.orderId=ORD-001` |
| Currencies | `currencyCode=SEK` |
| Countries | `countryCode=SE` |
| Languages | `languageCode=sv` |
| POS Terminals | `posTerminalName=Kassa%201` |
| POS Profiles | `posProfileId=default` |
| User Roles | `userRoleID=admin` |
| Templates | `templateId=default-receipt` |
| Mapped Types | `mappedTypeName=com.heads.receipt-csv` |
| Payment Methods | `methodId=com.heads.cash` |
| Receipts | `receiptID=RCP-2024-00001` |

### Relative Date Syntax (for receipts)

| Pattern | Meaning |
|---------|---------|
| `-=24` | 24 hours ago |
| `-=1` | 1 hour ago |
| `-=0:30` | 30 minutes ago |
| `-=168` | 1 week ago (168 hours) |

---

## Notes

1. **External IDs**: Use reverse domain notation for namespacing (e.g., `com.myapp.customerId`)
2. **URL Encoding**: Special characters must be URL-encoded (space = `%20`, etc.)
3. **camelCase Paths**: Nested members use camelCase: `customerRelations`, `childCategories`, `assortmentContexts`, `oauth2Clients`, `roleAssignments`
4. **Agent References**: Use `{"identifiers": {...}}` to reference existing agents in relationships
5. **Store Owner**: Stores use `owner` (not `parent`) to reference the owning company
6. **Product Groups**: Assign products via `parentGroup` on the product, NOT via the group's `members`
7. **DELETE Limitations**: Some sub-collection DELETEs return success but don't persist - always verify
