# CommerceOS API Overview

This document summarizes how CommerceOS API resources are structured and accessed.

---

## About the API Reference

This reference explains how to integrate with CommerceOS at a practical level. It focuses on workflows, operator syntax, and real examples.

**Audience:** Developers and AI agents building integrations with CommerceOS.

**Scope:** This reference covers HTTP usage, authentication, query operators, content negotiation, and common patterns. For endpoint schemas and model definitions specific to your deployment, use your tenant's `/api-docs`.

**Navigation ladder:**
1. **Start here** - This overview for API basics and authentication
2. **Learn operators** - [`operators.md`](operators.md) for query syntax and recipes
3. **See examples** - [`../guide/examples.md`](../guide/examples.md) for practical curl examples
4. **Dive deeper** - [`mapped-types.md`](mapped-types.md), [`sync-webhooks.md`](sync-webhooks.md), and other reference docs

> **Tip:** Your tenant's `/api-docs` is authoritative for endpoints and schemas. This reference complements it with patterns and examples.

---

## Resource Groups & Examples

The API organizes resources into logical groups. Each group maps to OAuth2 scopes and has practical examples.

| Tag | Scopes (informational) | Key Resources | Examples |
|-----|------------------------|---------------|----------|
| Organization | `org:read` | `/agents`, `/people`, `/companies`, `/stores` | [Organization examples](../guide/examples.md#organization-examples) |
| Products | `products:read`, `products:write` | `/products`, `/product-categories`, `/product-families` | [Products examples](../guide/examples.md#products-examples) |
| Pricing | `prices:read`, `prices:write` | `/prices`, `/currencies`, `/vat-codes` | [Pricing examples](../guide/examples.md#pricing-examples) |
| Orders | `orders.sales:write`, `orders.payments:write` | `/trade-orders`, `/payment-orders` | [Orders examples](../guide/examples.md#orders-examples) |
| Inventory | `stock:read`, `stock:write` | `/stock-places`, `/stock-transactions` | [Inventory examples](../guide/examples.md#inventory-examples) |
| POS | `pos:read`, `pos:write`, `receipts:write` | `/pos-terminals`, `/receipts`, `/payment-methods` | [POS examples](../guide/examples.md#pos-examples) |
| Users | `users:read` | `/users`, `/users/{id}/roleAssignments` | [Users examples](../guide/examples.md#users-examples) |
| Configuration | `config` | `/config`, `/mapped-types`, `/sync-webhooks` | [Configuration examples](../guide/examples.md#configuration-examples) |

> **Note:** The OAuth2 scopes listed above are informational. The deployed API uses coarse scopes (`read:api`, `write:api`). Fine-grained scopes may be introduced in the future.

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
- `gtin` (for products)

**Non-essential fields** (require `~with`):
- Relations: `categories`, `prices`, `supplierAgent`, `customerAgent`
- Extended properties: `hidden`, `plu`, `vatId`

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
- `~map(mappedTypeName)` - Apply mapped type transformation
- `~with(field)` - Expand additional fields

> **Important:** Do not mix query parameters (`?limit=…`) and `~` operators (`~take(…)`) in the same request. Choose one style per request. See [`operators.md`](operators.md) for details.

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

### Response Formats (Accept Header)

| Accept Header | Shorthand | Use Case |
|---------------|-----------|----------|
| `application/json` | `json` | Default, buffered response |
| `application/x-ndjson` | `ndjson` | Streaming, line-delimited |
| `text/csv` | - | Tabular export |
| `text/plain` | - | Plain text output |
| `text/html` | - | HTML-wrapped output |

**Default behavior:** If no Accept header is provided or `*/*` is used, `application/json` is returned.

**Shorthand values:** The Accept header supports shorthand values like `Accept: json` or `Accept: ndjson`.

**Content-Type defaults for input:** When sending data via POST/PUT/PATCH, if no Content-Type header is provided, `application/json` is assumed.

### Request Content-Type Defaults

When sending data:
- If no Content-Type header is provided, `application/json` is assumed
- For NDJSON import, explicitly set `Content-Type: application/x-ndjson`
- NDJSON accepts trailing newlines on import
- CSV import parses header row and supports quoted values

### JSON Serializer Parameters

The JSON content type accepts parameters to control output:

```bash
# Skip null fields (enabled by default)
Accept: application/json; skipNulls=true

# Include null fields (useful with ~with to show explicitly null values)
Accept: application/json; skipNulls=false

# Strip @type annotations
Accept: application/json; skipTypes=true

# Serialize numbers as strings (for precision)
Accept: application/json; numberHandling=string

# Streaming mode - fast first byte (items emitted as available)
Accept: application/json; stream=true
```

**`skipNulls` + `~with` interaction:** When `skipNulls=true` (default), null fields are omitted. However, if a field is explicitly requested via `~with()`, a null value is included to indicate the field exists but is empty.

### Format-Specific Behaviors

**JSON (`application/json`):**
- Empty collections return `[]`
- `~count` returns a number
- POST accepts arrays for bulk creation

**NDJSON (`application/x-ndjson`):**
- One JSON object per line
- Empty collections yield empty output (no lines)
- Trailing newlines are accepted on import

**CSV (`text/csv`):**
- Header row derived from first item's fields
- Commas, quotes, and newlines in values are escaped
- Empty collections yield empty output (no header)
- Single item yields header + 1 data row

**text/plain and text/html:**
- Require string output; non-string values cause error
- Use `~toString` to convert numeric results: `/products~count~toString`
- HTML wraps non-HTML string content in basic HTML structure

### Known Limitations

- **File extension content negotiation not supported:** URLs like `/products.json` or `/products.csv` do not work. The extension is treated as part of the resource identifier. Use the Accept header instead.
- **Unknown Accept header parameters:** Unknown parameters in the Accept header (e.g., `Accept: application/json; unknownParam=foo`) should be ignored while still applying known parameters. However, this is a known issue - some unknown parameters may cause errors. When in doubt, use only the documented parameters: `skipNulls`, `skipTypes`, `numberHandling`, and `stream`.

### Request Content Types

| Content-Type | Use Case |
|--------------|----------|
| `application/json` | Standard JSON body |
| `application/x-ndjson` | Bulk import (one object per line) |
| `text/csv` | Tabular import (header + rows) |

---

## Transaction and Buffering Semantics

All collection responses go through two phases: **buffering** (database read) and **serialization** (output writing).

### Phase 1: Buffering (Transaction-Bounded)

All items are read from the database within a single transaction before any output is written. This guarantees:

- **Snapshot consistency:** The entire result set reflects one point-in-time view of the data
- **Atomicity:** Either all items are returned, or none (on error)
- **No partial results:** Errors during buffering return no data, preventing corrupted or incomplete arrays

### Phase 2: Serialization (Format-Dependent)

After buffering completes, items are serialized according to the requested output format:

| Format | Serialization Behavior | First Byte Timing |
|--------|------------------------|-------------------|
| `application/json` | Buffered - full array built before output | After all items read + serialized |
| `application/json; stream=true` | Streaming - items emitted incrementally | After all items read |
| `application/x-ndjson` | Streaming - one line per item | After all items read |
| `text/csv` | Streaming - one row per item | After all items read |

**Key point:** Even streaming formats buffer all items during the database read phase. The "streaming" behavior applies only to the serialization phase, providing faster first-byte response times for large result sets while maintaining transaction consistency.
