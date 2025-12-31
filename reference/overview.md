# CommerceOS API Overview

This document summarizes how CommerceOS API resources are structured and accessed.

---

## API Basics

The CommerceOS API is a RESTful Web API using:
- **OAuth 2.0 / OpenID Connect** for authentication
- **OpenAPI 3.1** for specification
- **JSON** for data exchange (also supports NDJSON, CSV)

**Base URL**: `/api/v1`

---

## Authentication

### API Key
```bash
# Via header
curl -H "X-Api-Key: MySecretKey" localhost:5000/api/v1/void

# Via Basic Auth (password only)
curl -u ":MySecretKey" localhost:5000/api/v1/void
```

### OAuth 2.0
```bash
# Get token
curl -X POST localhost:5000/oauth2/v1/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=ID&client_secret=SECRET"

# Use token
curl -H "Authorization: Bearer TOKEN" localhost:5000/api/v1/void
```

---

## Resource Model and Endpoint Tree

Resources follow a nested structure that mirrors the data model. Any JSON member in a response can be navigated to by extending the URL path.

Key behaviors:
- `@type` discriminator on polymorphic types (agents, product nodes)
- External identifiers use reverse-domain keys (e.g., `com.example.id`)
- Endpoint tree follows schema structure: collections, members, sub-collections

### Path Casing Rules

- **Top-level collections**: kebab-case (e.g., `/trade-orders`, `/product-categories`)
- **Nested member paths**: camelCase (e.g., `/customerRelations`, `/childCategories`)

Example: `/agents/com.heads.seedID=store1/customerRelations` (NOT `customer-relations`)

---

## Common Endpoints

### Agents (People, Companies, Stores)
| Endpoint | Description |
|----------|-------------|
| `/agents` | All agents |
| `/people` | People (persons) |
| `/companies` | Companies |
| `/stores` | Stores |
| `/people/com.example.id=123` | Person by external ID |

### Products
| Endpoint | Description |
|----------|-------------|
| `/products` | All products |
| `/product-categories` | Product categories |
| `/product-families` | Product families |
| `/product-groups` | Product groups |

### Orders & Commerce
| Endpoint | Description |
|----------|-------------|
| `/trade-orders` | Trade orders (sales/purchase) |
| `/trade-orders/{key}/items` | Order items |
| `/payment-orders` | Payment orders |
| `/shipment-orders` | Shipment orders |
| `/receipts` | POS receipts |

### Inventory
| Endpoint | Description |
|----------|-------------|
| `/stock-places` | Stock locations |
| `/stock-places/{key}/entries` | Stock entries at location |

### Configuration
| Endpoint | Description |
|----------|-------------|
| `/config/{key}` | System configuration singletons |
| `/pos-terminals` | POS terminals |
| `/payment-methods` | Payment methods |
| `/mapped-types` | Mapped types for transformations |
| `/sync-webhooks` | Automated data sync tasks |

---

## External Identifiers

Objects with `common identifiers` can hold dynamic external identifiers:

```bash
# Create person with external ID
POST /people '{"identifiers": {"com.myapp.userId": "123"}}'

# Or use PUT (creates if not exists)
PUT /people/com.myapp.userId=123 '{"givenName": "John"}'

# Access by external ID
GET /people/com.myapp.userId=123
```

---

## Common Identifiers and Dynamic Properties

`commonIdentifiers` exposes `identifiers.key` plus all dynamic, namespaced external IDs:
- Supports namespaced keys (e.g., `com.example.id`)
- Enables lookup by external identifier

`definedProperties` exposes:
- `declared`: Static members from schema definitions
- `dynamic`: Properties with `propertyType`, `description`, `requiredOnCreate`

---

## Field Expansion

Most resources return essential fields only. Non-essential fields are accessed via:
- `~with(...)` / `~withAll`
- `fields=...` or `fields=all` query parameters

**Essential fields** (always returned):
- `identifiers` (including `key`)
- `name`
- `@type` (for polymorphic types)
- `status` (for products)

**Non-essential fields** (require `~with`):
- Relations: `categories`, `prices`, `supplierAgent`, `customerAgent`
- Extended properties: `hidden`, `plu`, `gtin`, `vatId`

```bash
# Include specific non-essential fields
GET /products/com.example.sku=ABC~with(hidden,plu,gtin)

# Include ALL fields
GET /products/com.example.sku=ABC~withAll
```

---

## Query Operators

The API supports query operators that modify responses:
- `~where(condition)` - Filter collections
- `~take(N)` and `~skip(N)` - Pagination
- `~orderBy(field)` - Sorting
- `~map(name)` - Apply mapped type transformation
- `~with(field)` - Expand additional fields

See [`operators.md`](operators.md) for the full operator reference.

---

## HTTP Methods

| Method | Usage |
|--------|-------|
| `GET` | Read resources |
| `POST` | Create new resources |
| `PUT` | Create or update (upsert) |
| `PATCH` | Partial update |
| `DELETE` | Remove resources |

---

## Content Types

| Format | Content-Type | Use Case |
|--------|--------------|----------|
| JSON | `application/json` | Default, buffered response |
| NDJSON | `application/x-ndjson` | Streaming large datasets |
| CSV | `text/csv` | Tabular export |
