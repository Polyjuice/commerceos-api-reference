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

## API Response Behaviors

### Empty Collection Results

| Operation | Empty Result |
|-----------|--------------|
| `~count` | `0` |
| `~first` | `null` |
| `~take(N)` | `[]` |

### Person Name Derivation

For Person entities, `name` is derived from `givenName + familyName`:
- Setting `name` directly may be overwritten
- Query using `givenName`/`familyName` for reliable results
