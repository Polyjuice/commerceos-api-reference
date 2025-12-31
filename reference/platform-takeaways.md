# Platform Takeaways (API Usage)

Concise notes on key CommerceOS API patterns and behavior.

---

## 1) Operators and Selectors

The API supports query operators that chain left-to-right in URL paths:

- `~where(condition)` - Filter collections
- `~with(field)` - Expand non-essential fields
- `~just(fields)` - Project specific fields only
- `~map(name)` - Apply a mapped type transformation
- `~take(N)` and `~skip(N)` - Pagination
- `~orderBy(field:desc)` - Sorting
- `~distinct` and `~distinctBy(key)` - Deduplication

Selector syntax supports:
- `$prior`, `$this`, `$index` for context references
- Concatenation with `+`
- Null coalescing with ` ?? ` (spaces required)
- Ternary expressions `cond ? a : b`
- Type coercion via `->string`, `->number`, `->boolean`

---

## 2) Mapped Types

Mapped types provide reusable transformations for data export:

- Stored in the `mapped-types` resource
- Applied via `~map(<name>)` operator
- `X-Request-Map` header maps incoming request bodies before write
- `@merge` returns the first mapped object when mapping collections

---

## 3) Common Resource Patterns

### Agents

Agents represent people, companies, or stores:

- `identifiers` - External ID support with reverse-domain keys
- `name` - Full name
- `nationality`, `languages`, `vatId`
- `addresses` - Nested object with `main`, `home`, `invoice`, `delivery`, `visiting`
- `contactMethods` - Phone numbers, email addresses
- Relations: `customer`, `supplier`, `manufacturer`
- `timeline` - Receipts for buyer

### Product Nodes

Products in the catalog:

- `identifiers` - External ID support
- `name`, `gtin`, `plu`, `hidden`, `createdAt`, `createdBy`
- `images` - With full replace on set
- `assortmentOwners` - Add/remove pattern
- `labels` - Assigned labels
- `xrefs.compatibles/alternatives` - Cross-references

---

## 4) Essential vs Non-Essential Fields

Most API responses include only "essential" fields by default:

- Use `~with(field)` or `~withAll` to expand additional fields
- Use `~just(a,b,c)` to return only specified fields
- Query params `fields=...` or `fields=all` also work

---

## 5) External Identifiers

Resources support user-defined external identifiers:

- Reverse-domain notation: `com.example.id`
- Access via `identifiers` member
- Multiple external IDs can exist on the same object
- Used for integration and cross-system references
