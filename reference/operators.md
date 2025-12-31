# Query Operators

This reference documents CommerceOS API query operators.

> **Quick Reference:** See [`../guide/examples.md`](../guide/examples.md) for practical usage examples.

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
  - Truthy/falsy: `field` / `!field`
  - Predicate separators: `,` or `&` within `~where(...)`
  - Date coercion: Datetime values on either side are compared via `.getTime()`

### Ordering & Pagination
- `~orderBy(field)` or `~orderBy(field:desc)` - Order objects by selector value.
- `~order` / `~order(desc)` - Order primitive streams in ascending/descending order.
- `~take(N)` - Take first N items.
- `~skip(N)` - Skip first N items.
- `~first` - Return first item (reducer).
- `~last` - Return last item (reducer).
- `~count` - Return item count as number (reducer).

### Distinct
- `~distinct` - Deduplicate streams by value identity. Works on primitives and objects.
- `~distinctBy(selector)` - Deduplicate objects by selector value. Evaluates selector per item, drops subsequent duplicates.

### Transformation
- `~map(selectorOrMappedType)` - Map fields via inline selector OR apply a registered mapped type.
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

## Evaluation Notes

- Operators chain left-to-right over the data stream.
- `~where`, `~orderBy`, `~distinctBy`, `~with`, `~just` evaluate selectors (same selector language as mapped types).
- `~map` with a mapped type name looks up the type by name and strips `@type`.
- `~map` with `@merge` returns a single merged object instead of a collection.
- Comparisons in `~where` and `~orderBy` use numeric comparison for numeric values.

---

## Performance Considerations

- Prefer indexed array paths (e.g., `/receipts/after/-=24`) over `~where(...)` for large datasets.
- Use `~with`/`~just` to limit output and avoid expanding expensive relations.
- `~orderBy` collects all items into memory before sorting—not suitable for very large collections.
