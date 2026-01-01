# Common API Gotchas

Known pitfalls and their solutions when working with the CommerceOS API.

---

## 1. No GTIN/PLU Indexer on Products

Products cannot be indexed by GTIN or PLU directly.

```bash
# WRONG - No GTIN indexer exists
GET /products/gtin=7312345678901

# RIGHT - Use ~where() to filter by GTIN
GET /products~where(gtin=7312345678901)~first
```

---

## 2. Nested Paths Use camelCase

Top-level collections use kebab-case, but nested member paths use camelCase:

```bash
# RIGHT - camelCase for member paths
GET /agents/{key}/customerRelations
GET /product-categories/{key}/childCategories

# WRONG - kebab-case for member paths
GET /agents/{key}/customer-relations  # Will not work
```

---

## 3. ~reverse() and ~any() Don't Exist

```bash
# WRONG - ~reverse() doesn't exist
GET /products~orderBy(name)~reverse()

# RIGHT - Use :desc suffix
GET /products~orderBy(name:desc)

# WRONG - ~any() doesn't exist
GET /products~any(status=Active)

# RIGHT - Use ~where() + ~first or ~count
GET /products~where(status=Active)~first
```

---

## 4. Store Uses `owner`, Not `parent`

Stores use `owner` to reference the owning company:

```bash
# RIGHT
POST /stores '{"owner": {"identifiers": {...}}, ...}'

# WRONG
POST /stores '{"parent": {"identifiers": {...}}, ...}'
```

---

## 5. POS Terminal Uses `associatedNode`, Not `store`

```bash
# RIGHT
POST /pos-terminals '{"associatedNode": {"identifiers": {...}}, ...}'

# WRONG
POST /pos-terminals '{"store": {"identifiers": {...}}, ...}'
```

---

## 6. Trade Relationship Uses `supplierAgent`/`customerAgent`

```bash
# RIGHT
POST /trade-relationships '{
  "supplierAgent": {"identifiers": {...}},
  "customerAgent": {"identifiers": {...}}
}'

# WRONG - supplier/customer without "Agent" suffix
POST /trade-relationships '{"supplier": {...}, "customer": {...}}'
```

---

## 7. Currency Must Be Referenced by Identifier

```bash
# WRONG
POST /prices '{"currency": "SEK", ...}'

# RIGHT
POST /prices '{"currency": {"identifiers": {"currencyCode": "SEK"}}, ...}'
```

---

## 8. Agent References Need @type and identifiers

```bash
# WRONG
POST /stores '{"owner": {"com.myapp.id": "123"}}'

# RIGHT
POST /stores '{"owner": {"@type": "company", "identifiers": {"com.myapp.id": "123"}}}'
```

---

## 9. Parameterless Operators: No Empty Parentheses

Operators without parameters must be used WITHOUT parentheses:

```bash
# WRONG - Empty parentheses break the query
GET /products~first()   # Returns "not found"
GET /products~count()   # Error

# RIGHT - No parentheses
GET /products~first
GET /products~count
GET /products~map(status)~distinct
```

---

## 10. Products Belong to Groups via `parentGroup`

To add a product to a group, set the product's `parentGroup`:

```bash
# WRONG - This doesn't persist
POST /product-groups/{key}/members '{"identifiers": {...}}'

# RIGHT - Set parentGroup on the product
PATCH /products/{key} '{"parentGroup": {"identifiers": {...}}}'
```

---

## 11. Companies Don't Have `stores` Member

Companies do NOT have a `stores` array. Use trade-relationships or query stores:

```bash
# WRONG
GET /companies~with(stores)

# RIGHT - Query stores by owner
GET /stores~where(owner/identifiers/key=COMPANY_KEY)
```

---

## 12. Agents Don't Have `tradeOrders` or `stockTransactions`

Stores and other agents don't have direct order/transaction members:

```bash
# WRONG
GET /stores~with(tradeOrders)

# RIGHT - Query orders directly
GET /trade-orders~where(sellers/identifiers/key=STORE_KEY)
```

---

## 13. ~distinct Only Works on Primitives

```bash
# WRONG - No effect on object streams
GET /products~distinct

# RIGHT - Use ~distinctBy for objects
GET /products~distinctBy(status)

# Or map to primitive first
GET /products~map(status)~distinct
```

---

## 14. Use `contactMethods`, Not `contactPoints`

Agent contact information is under `contactMethods`:

```bash
# RIGHT
GET /people/com.test.id=123~with(contactMethods)
PATCH /people/com.test.id=123 '{"contactMethods": {"email": "new@example.com"}}'

# WRONG - contactPoints doesn't exist
GET /people/com.test.id=123~with(contactPoints)
```

Slots: `landlinePhone`, `mobilePhone`, `workPhone`, `email`

---

## 15. SQL Serializer is Upcoming

The SQL output format (generating INSERT/MERGE statements) is an **upcoming feature**, not currently available. Do not attempt to use `format=sql` or similar parameters.

---

## 16. Body Indicator Navigation Patterns

The `@` marker in a URL path targets the request body to a specific endpoint. Path segments after `@` navigate through the result.

```bash
# Basic pattern: @target/navigation
PUT /agents/@find/results           # Send body to /agents/find, navigate to results

# Navigate into results by index
PUT /agents/@find/results/0         # First match
PUT /agents/@find/results/0/name    # Name of first match

# Operators work after navigation
PUT /agents/@find/results~take(5)~just(name,identifiers)
```

**Common finder endpoints:**
- `/agents/@find` - Agent lookup by email, phone, nationalId, or metaphone-based name filtering
- `/trade-orders/@find` - Order lookup by modifiedTag, customer, supplier, buyer, seller

---

## 17. @merge Returns Single-Element Array

When using a mapped type with `"@merge": true`, the result is a **single-element array**, not a single object:

```bash
# Returns: [{"merged": "object"}]  NOT {"merged": "object"}
GET /receipts~map(com.heads.receipts-zip)
```

This is intentional to keep the response type consistent (always an array when mapping collections).

---

## 18. Ternary Conditions in Mapped Types Test Existence

In mapped type ternary expressions, the condition tests for **existence**, not value comparison:

```json
{
  "result": "someField ? 'has value' : 'empty'"
}
```

This returns `'has value'` if `someField` exists and is truthy, not if it equals a specific value. For value comparisons, use `~where()` filtering before mapping.

---

## 19. Don't Mix Query Parameters and Operators

> **Rule: Choose one query layer per request**
>
> Use either standard query parameters (`?limit=…&orderby=…`) **or** `~` operators (`~take(…)~orderBy(…)`) in a single request. Do not combine them. Mixed requests can produce unexpected results.

**Examples:**

```bash
# WRONG: mixing query params and operators
GET /products~orderBy(name)?limit=10

# RIGHT: query params only
GET /products?orderby=name&limit=10

# RIGHT: operators only
GET /products~orderBy(name)~take(10)
```

---

## 20. Product Identifiers Require Fully Qualified Namespace

Product references in trade orders (and elsewhere) require a **fully qualified namespace** on identifiers. Using bare keys like `"sku"` will fail.

```bash
# WRONG - bare "sku" key is not a valid identifier
{
  "items": [
    {"product": {"identifiers": {"sku": "PHONE-001"}}, "quantity": 1}
  ]
}

# RIGHT - use reverse-domain namespace
{
  "items": [
    {"product": {"identifiers": {"com.example.sku": "PHONE-001"}}, "quantity": 1}
  ]
}
```

Always use a namespaced key like `com.yourcompany.sku`, `com.myapp.productId`, etc.

---

## 21. Instance-Tracked Items: Order and Type Requirements

When using instance tracking (IMEI, serial numbers), the item order matters:

```bash
# WRONG - MobilePlan before MobileDevice
{
  "items": [
    {"product": {"identifiers": {"com.example.sku": "PLAN-001"}}, "instances": [{"phoneImei": "123456789012345"}]},
    {"product": {"identifiers": {"com.example.sku": "PHONE-001"}}, "instances": [{"imei": "123456789012345"}]}
  ]
}

# RIGHT - MobileDevice before MobilePlan
{
  "items": [
    {"product": {"identifiers": {"com.example.sku": "PHONE-001"}}, "instances": [{"imei": "123456789012345"}]},
    {"product": {"identifiers": {"com.example.sku": "PLAN-001"}}, "instances": [{"phoneImei": "123456789012345"}]}
  ]
}
```

The `phoneImei` must reference an `imei` from an item that appears **earlier** in the items array.

> **Tip:** Products with instance tracking need the correct `instanceType` set during product creation (`"MobileDevice"` for devices, `"MobilePlan"` for plans). This is a product setup requirement, not something you specify on the order.

---

## API Response Behaviors

### Empty Collection Results

| Operation | Empty Result |
|-----------|--------------|
| `~count` | `0` |
| `~first` | `null` |
| `~take(N)` | `[]` |

### Person Name Derivation

For Person entities, `name` is a computed field derived from `givenName + familyName`:
- `name` is **not returned by default** - use `~with(name)` to include it
- Setting `name` directly may be overwritten by the computed value
- To sort by name, use `~with(name)~orderBy(name)`
- Query using `givenName`/`familyName` for reliable filtering

```bash
# WRONG - name is not included by default
GET /people~orderBy(name)~take(10)

# RIGHT - explicitly include computed name field
GET /people~with(name)~orderBy(name)~take(10)
```
