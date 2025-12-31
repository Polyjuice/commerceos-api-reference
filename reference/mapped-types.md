# Mapped Types

Mapped types are named mapping definitions stored in the API and used to transform data in two places:
- **Response mapping** via the `~map(<mappedTypeName>)` operator.
- **Request mapping** via the `X-Request-Map` header on write operations.

This document covers the storage model, lookup rules, and the selector language used in mapped type bodies.

---

## Storage and Lookup

**Identifiers**
- `identifiers.mappedTypeName` is the canonical name used by `~map(...)` and `X-Request-Map`.

**Body storage**
- `body` is stored as a JSON string.
- On read, `body` is parsed. Array values are treated as collections.

**Lookup**
- `~map(<name>)` resolves the mapped type by name.
- If the mapped type has no body, `NotFound` is thrown with: `There is no active mapped type with name '<name>'`.
- Note: The `active` flag is purely metadata; the lookup only checks for presence of a body.

---

## Map Operator Behavior

**Settings:**
- Output removes `@type` annotations
- Spaces inside selectors become concatenation boundaries
- Output contains only mapped fields (clears existing subunits)
- No implicit field expansion outside the mapping

**Single vs collection**
- Mapping a **single item** returns one mapped object.
- Mapping a **collection** returns an array of mapped objects.
- If the mapped type body contains `@merge`, the collection mapping returns only the first mapped object. This is used when the mapping references the whole collection via `$prior`.

---

## Selector Language (Mapping Body)

Mapped type bodies are objects whose values can be selectors or nested mappings.

### 1) URI selectors and operators

Selectors behave like API paths:
- `identifiers/receiptID`
- `items/count/str`
- `discounts~map(com.heads.receipt-item-discount-line)/*line~join(;)`

Selectors may start with:
- `/` to anchor at the current item
- `/api/v1/...` to anchor at the API root
- `~` to apply operators directly

### 2) Literals

Wrap in single quotes to force a literal string:
- `"buyer/name ?? 'unknown'"`
- `"reason/name+':'+amount/str"`

### 3) Null-coalescing

Use ` ?? ` (note required spaces) to provide a fallback:
- `"buyer/name ?? ''"`

### 4) Ternary expressions

Use `cond ? trueExpr : falseExpr` with paren awareness:
- `"paymentType ? paymentType : 'unknown'"`

### 5) Concatenation

Use `+` to concatenate values and literals:
- `"identifiers/receiptID+':'+timestamp/str"`

If all parts are primitive, the result becomes a string. If any part is non-primitive, a flattened array is returned.

### 6) Context references

Special prefixes navigate the data graph:
- `$this` -> current item
- `$prior` -> parent item (repeatable: `$prior/$prior/...`)
- `$index` -> array index of the current item

Example from default mapped types:
```
"Receipt ID": "$prior/$prior/identifiers/receiptID"
```

### 7) Variables

If a **key** in the mapping object starts with `$`, it defines a variable instead of emitting a field:
```
{
  "$seller": "seller",
  "Seller name": "$seller/name"
}
```

Variables can be reused in later selectors within the same mapping.

### 8) Nested objects and arrays

Mapping values can be objects (nested mappings) or arrays:
```
{
  "address": { "line1": "addresses/main/line1" },
  "tags": ["labels/0", "labels/1"]
}
```

Arrays are emitted as arrays of resolved values.

---

## Request Mapping (X-Request-Map)

Write operations can transform incoming JSON by specifying the `X-Request-Map` header.

Behavior:
- The incoming body is deserialized.
- Each item is mapped using the mapped type body.
- The mapped result is JSON-serialized to materialize computed values before write.

Example:
```
PUT /v1/products
X-Request-Map: com.myapp.product-import

[
  {"sku":"P-1","title":"Widget","state":"Active"}
]
```

Mapped type body:
```
{
  "identifiers": { "com.myapp.sku": "sku" },
  "name": "title",
  "status": "state"
}
```

---

## Default Mapped Types

Key patterns:
- **Receipt CSV export**: `com.heads.receipt-csv`
- **Receipt item CSV export**: `com.heads.receipt-item-csv`
- **Receipt payment CSV export**: `com.heads.receipt-payment-csv`
- **Receipt discount line**: `com.heads.receipt-item-discount-line`
- **Receipt ZIP bundle**: `com.heads.receipts-zip` with `@merge` and `$prior`

Examples:
```
GET /receipts~map(com.heads.receipt-csv)
GET /receipts~just(items~map(com.heads.receipt-item-csv))/*items~flat
GET /receipts~just(payments~map(com.heads.receipt-payment-csv))/*payments~flat
GET /receipts~map(com.heads.receipts-zip)
```

The ZIP mapping uses:
```
{
  "@merge": true,
  "receipts": "$prior~map(com.heads.receipt-csv)",
  "items": "$prior~just(items~map(com.heads.receipt-item-csv))/*items~flat",
  "payments": "$prior~just(payments~map(com.heads.receipt-payment-csv))/*payments~flat"
}
```

`@merge` ensures the mapping returns a single bundle when mapping a collection.
