# Query Operators

This reference documents CommerceOS API query operators.

> **Quick Reference:** See [`../guide/examples.md`](../guide/examples.md) for practical usage examples.

---

## Operator Syntax vs Query Parameters

CommerceOS supports two distinct query layers:

- **Basic layer (query params):** `/v1/products?status=Active&limit=10`
- **Advanced layer (operators):** `/v1/products~where(status=Active)~take(10)`

**Do not mix** query parameters and operators in the same request. Pick one layer per request and keep it consistent.

| Layer | Filtering | Pagination | Sorting | Expansion |
|-------|-----------|------------|---------|-----------|
| Query params | `?status=Active` | `?limit=10&offset=20` | `?orderby=name` | `?fields=all` |
| Operators | `~where(status=Active)` | `~take(10)~skip(20)` | `~orderBy(name)` | `~withAll` |

**Why two layers?** Query parameters are familiar and work well for simple requests. Operators are chainable and expressive for complex queries like multi-field filtering, nested expansions, and mapped type transformations.

**Evaluation order:** Operators are evaluated by the engine in a fixed sequence regardless of URL position. Query parameters are translated into that same sequence (see [Operator Application Order](#operator-application-order) below).

### Operator Recipes

Common operator patterns for everyday tasks:

```bash
# Paginated list (page 3, 20 items per page)
GET /v1/products~take(20)~skip(40)

# Filtered search
GET /v1/products~where(status=Active,categoryKey=electronics)

# Expanded view with pagination
GET /v1/products~with(prices,categories)~take(10)

# Sorted and filtered
GET /v1/products~where(status=Active)~orderBy(name)~take(20)

# Transformed export
GET /v1/products~map(com.example.export-format)~take(100)
```

---

## Operator Categories

### Projection
- `~with(selectors)` - Add/replace fields on each object.
- `~withAll` - Include all fields (expands all non-essential fields).
- `~without(selectors)` - Remove fields.
- `~just(selectors)` - Keep only specified fields; clears existing subunits first.
- `~simpleJust(names)` - Keep only given member names (no selector parsing, just property name matching).

### Filtering
- `~where(predicates)` - Filter by predicates.
  - Predicate operators: `=`, `!=`, `>`, `<`, `>=`, `<=`, `=~` (includes), `!~` (not includes)
  - Truthy/falsy: `field` (truthy check) / `!field` (falsy check)
  - Predicate separators: `,` or `&` within `~where(...)` - both are AND
  - Multiple `~where` clauses: Each additional `~where` adds more AND conditions (they combine as AND)
  - **Value parsing rules:**
    - `null` and `undefined` are parsed as their literal values
    - `true` and `false` are parsed as booleans
    - Numbers (including decimals and negative) are parsed numerically
    - ISO 8601 dates (e.g., `2024-12-20T10:00:00Z`) are parsed as Date objects
    - Empty strings: use `field=` (nothing after `=`) to match empty string
    - Nested paths work as keys: `~where(addresses/main/countryCode=SE)`
  - Date coercion: Datetime values on either side are compared via `.getTime()`

**Filtering Examples:**

```bash
# Simple equality
GET /products~where(status=Active)

# Not equals
GET /products~where(status!=Discontinued)

# Includes (contains in array or string)
GET /products~where(gtin=~7312345)

# Truthy check (field exists and is truthy)
GET /products~where(hidden)

# Falsy check (field is null/undefined/false/empty)
GET /products~where(!hidden)

# Nested path filtering
GET /people~where(addresses/main/countryCode=SE)

# Multiple conditions (AND)
GET /products~where(status=Active,hidden=false)

# Chained where clauses (also AND)
GET /products~where(status=Active)~where(hidden=false)
```

### Ordering & Pagination
- `~orderBy(field)` or `~orderBy(field:desc)` - Order objects by selector value.
- `~orderBy(field1,field2:desc)` - Multi-field ordering: sorts by first field, then by second for ties.
- `~order` / `~order(desc)` - Order primitive streams in ascending/descending order.
- `~take(N)` - Take first N items.
- `~skip(N)` - Skip first N items.
- `~first` - Return first item (reducer). Returns `null` if collection is empty.
- `~last` - Return last item (reducer). Returns `null` if collection is empty.
- `~count` - Return item count as number (reducer). Returns `0` for empty collections.

**Ordering Examples:**

```bash
# Ascending order (default)
GET /products~orderBy(name)

# Descending order
GET /products~orderBy(name:desc)

# Multi-field ordering (sort by status, then by name within each status)
GET /products~orderBy(status,name)

# Order by nested field
GET /trade-orders~orderBy(customer/name)

# Pagination: skip 20, take 10
GET /products~skip(20)~take(10)
```

### Distinct
- `~distinct` - Deduplicate streams by value identity. Works on primitives (strings, numbers). For objects, use `~distinctBy`.
- `~distinctBy(selector)` - Deduplicate objects by selector value. Evaluates selector per item, drops subsequent duplicates.

**Distinct Examples:**

```bash
# Deduplicate objects by a field
GET /products~distinctBy(status)

# Deduplicate objects by a nested field
GET /trade-orders~distinctBy(customer/identifiers/key)
```

### Transformation
- `~map(mappedTypeName)` - Apply a registered mapped type by name.
  - **Mapped type lookup:** `~map(com.example.typeName)` looks up a mapped type with that exact name in `/mapped-types`. The name must match the `mappedTypeName` identifier.
  - **Field projection:** Use `~just(...)` for simple field selection, or create a mapped type when you need reshaping.
- `~flat` - Flatten nested arrays one level deep.
- `~entries` - Convert object to `{index, key, value}[]` entries (excludes `@type`).
- `~array` - Wrap single item in an array.
- `~typeless` - Set context flag to strip `@type` from output.
- `~join(separator)` - Join array elements to string; default separator is `,`.
- `~toLower` - Convert string to lowercase; returns `undefined` for non-strings.
- `~toUpper` - Convert string to uppercase; returns `undefined` for non-strings.
- `~toString` - Convert value to string via `.toString()`.
- `~repeat(N)` - Repeat input N times.

---

## Parameter Rules

**Parameterized operators** (require arguments):
- `~where(...)`, `~with(...)`, `~without(...)`, `~just(...)`, `~simpleJust(...)`
- `~orderBy(...)`, `~distinctBy(...)`
- `~map(...)`, `~join(...)`, `~take(...)`, `~skip(...)`, `~repeat(...)`

**Non-parameterized operators** (no parentheses):
- `~withAll`, `~first`, `~last`, `~count`, `~distinct`
- `~flat`, `~entries`, `~array`, `~typeless`
- `~toLower`, `~toUpper`, `~toString`

**Special case**:
- `~order` can be used without parameter (defaults to `asc`) or with `~order(desc)`

---

## Query Parameter Equivalents

Standard query parameters are translated to operators:

| Query Parameter | Operator Equivalent | Example |
|-----------------|---------------------|---------|
| `limit=N` | `~take(N)` | `?limit=10` → `~take(10)` |
| `offset=N` | `~skip(N)` | `?offset=20` → `~skip(20)` |
| `fields=a,b` | `~just(a,b)` + `~simpleJust(a,b)` | `?fields=prices,categories` → `~just(prices,categories)~simpleJust(prices,categories)` |
| `fields=all` | `~withAll` | `?fields=all` → `~withAll` |
| `orderby=field` | `~orderBy(field)` | `?orderby=name` → `~orderBy(name)` |
| `orderby=field:desc` | `~orderBy(field:desc)` | `?orderby=name:desc` → `~orderBy(name:desc)` |
| `format=json` | Accept: application/json | Content type selection |
| `format=ndjson` | Accept: application/x-ndjson | Streaming output |
| `format=csv` | Accept: text/csv | Tabular export |
| `format=txt` | Accept: text/plain | Plain text output |
| `format=html` | Accept: text/html | HTML-wrapped output |

**Fields edge cases:** `fields=none` maps to `~just()` with empty args. `fields=all,extra` emits `~withAll` and `~with(extra)`. `fields=default` is a no-op.

> **Rule: Choose one query layer per request**
>
> Use either standard query parameters (`?limit=…&orderby=…`) **or** `~` operators (`~take(…)~orderBy(…)`) in a single request. Do not combine them. Mixed requests like `GET /products~orderBy(name)?limit=10` can produce unexpected results. Pick one layer and keep the request consistent.

**Examples:**

```bash
# Preferred: query params only
GET /products?orderby=name&limit=10

# Preferred: operators only
GET /products~orderBy(name)~take(10)
```

**Multiple `where` clauses:** Each non-reserved query parameter becomes a `~where` clause. Repeating the same key (e.g., `?status=Active&status=Pending`) produces multiple `~where` clauses that combine as AND.

---

## Operator Application Order

Operators are evaluated in a fixed sequence, regardless of whether they come from query parameters or `~` operators in the path.

1. `format` (content type selection)
2. `fields` / `~with` / `~withAll` (field expansion)
3. `~where` (filtering)
4. `~skip` (offset)
5. `~take` (limit)
6. `~orderBy` (sorting)
7. `~simpleJust` (final projection)

URL order does not change evaluation. The engine applies the fixed order above even if the operators appear in a different sequence in the path.

---

## Evaluation Notes

- Operator execution follows the fixed order listed above; URL order does not change results.
- `~where`, `~orderBy`, `~distinctBy`, `~with`, `~just` evaluate selectors (same selector language as mapped types).
- `~map` resolves a mapped type by name and strips `@type` annotations from output.
- `~map` with `@merge`: When a mapped type contains `"@merge": true`, mapping a collection returns a **single-element array** containing the merged object (not a raw object). This is used when the mapping references the whole collection via `$prior`.
- Comparisons in `~where` and `~orderBy` use numeric comparison for numeric values.

---

## Performance Considerations

- Prefer indexed array paths (e.g., `/receipts/after/-=24`) over `~where(...)` for large datasets.
- Use `~with`/`~just` to limit output and avoid expanding expensive relations.
- `~orderBy` collects all items into memory before sorting—not suitable for very large collections.
