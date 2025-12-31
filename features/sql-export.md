# API to SQL Output

**Version:** 1.0
**Status:** Proposed (Not Yet Implemented)
**Last Updated:** 2024-12-18
**Related:** HEADS-5739, GT-193

> This is a forward-looking specification. The SQL export features described here are not available in the current API and should be treated as design guidance only.

## 1. Overview

This document defines a proposed SQL-compatible output serialization for the CommerceOS API, enabling data export to MSSQL Server and other SQL databases.
In the proposed design, SQL support is implemented purely as a serializer that consumes the intermediary format; it is not exposed as a query operator and does not enforce dialect rules.

### 1.1 Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Intermediary Format | JS object array | Decouples data transformation from SQL generation |
| Primary Format | RFC 4180 CSV | Best performance + SQL Server 2017+ native support |
| Secondary Format | SQL INSERT statements | Human readable, direct execution |
| Tertiary Format | SQL MERGE statements | Upsert/sync scenarios |
| Nested Handling | Flatten (default) | Simpler SQL schema, better query performance |
| Array Handling | JSON column (default) | Preserves data, SQL Server JSON support |
| Streaming | Row-by-row for CSV, batched for INSERT | Memory efficient |
| Batch Size | 1000 rows (default) | Balance of performance factors |
| REPLACE INTO | Not supported (SQL Server) | SQL Server has no REPLACE INTO; use MERGE or sync mode |
| Operator | None | SQL output is provided via serializer only |
| Dialect handling | None | Serializer emits statements from the intermediary format as-is |

### 1.2 Scope

**In Scope:**
- SQL-compatible serialization formats (CSV, INSERT, MERGE)
- Multi-table export for related entities
- Schema generation
- Configurable batching and commit frequency
- Type mapping from API types to SQL types
- Streaming output for large datasets

**Out of Scope:**
- Direct ODBC/database connections from API (file/HTTP output only)
- Real-time CDC replication
- Bi-directional sync

---

## 2. Intermediary Format

The SQL serializer consumes an intermediary format: an array of JS objects representing SQL statements. Each object has a key indicating the statement type.
In the proposed design, the serializer is dialect-agnostic and does not interpret or rewrite statement intent beyond literal rendering.

### 2.1 Statement Types

```typescript
type SqlStatement =
  | { "DELETE FROM": string; "WHERE": Record<string, unknown[]> }
  | { "INSERT INTO": string; "VALUES": Record<string, unknown>[] }
  | { "MERGE INTO": string; "ON": string[]; "VALUES": Record<string, unknown>[] }
  | { "UPDATE": string; "SET": Record<string, unknown>; "WHERE": Record<string, unknown[]> }
  | { "GO": true }
  | { "--": string };
```

### 2.2 Statement Examples

**DELETE:**
```javascript
{ "DELETE FROM": "HeadsReceipt", "WHERE": { "ReceiptObjectID": [1001, 1002] } }
// → DELETE FROM [HeadsReceipt] WHERE [ReceiptObjectID] IN (1001, 1002)
```

**INSERT:**
```javascript
{ "INSERT INTO": "HeadsReceipt", "VALUES": [
  { ReceiptObjectID: 1001, SellerID: "S001", Total: 1250.00 },
  { ReceiptObjectID: 1002, SellerID: "S001", Total: 890.50 }
]}
// → INSERT INTO [HeadsReceipt] ([ReceiptObjectID], [SellerID], [Total]) VALUES (1001, N'S001', 1250.00), (1002, N'S001', 890.50)
```

**MERGE (upsert):**
```javascript
{ "MERGE INTO": "HeadsReceipt", "ON": ["ReceiptObjectID"], "VALUES": [
  { ReceiptObjectID: 1001, SellerID: "S001", Total: 1250.00 }
]}
// → MERGE INTO [HeadsReceipt] AS t USING (VALUES ...) AS s ON t.[ReceiptObjectID] = s.[ReceiptObjectID] WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT ...
```

**UPDATE:**
```javascript
{ "UPDATE": "HeadsReceipt", "SET": { "Total": 1300.00 }, "WHERE": { "ReceiptObjectID": [1001] } }
// → UPDATE [HeadsReceipt] SET [Total] = 1300.00 WHERE [ReceiptObjectID] IN (1001)
```

**Batch separator:**
```javascript
{ "GO": true }
// → GO
```

**Comment:**
```javascript
{ "--": "Exported from CommerceOS API" }
// → -- Exported from CommerceOS API
```

### 2.3 Delete-then-Insert Pattern

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

### 2.4 Type Inference

The serializer infers SQL types from JS values at runtime:

| JS Type | SQL Output |
|---------|------------|
| `string` | `N'escaped value'` |
| `number` (integer) | `123` |
| `number` (decimal) | `123.4500` |
| `boolean` | `1` or `0` |
| `null` | `NULL` |
| `Date` | `'2024-12-18T10:30:00.000'` |

Column names and order are derived from `Object.keys()` of the first row in `VALUES`.

### 2.5 Serializer Implementation

```typescript
function serializeSqlStatements(statements: SqlStatement[]): string {
  return statements.map(stmt => {
    if ("DELETE FROM" in stmt) return serializeDelete(stmt);
    if ("INSERT INTO" in stmt) return serializeInsert(stmt);
    if ("MERGE INTO" in stmt) return serializeMerge(stmt);
    if ("UPDATE" in stmt) return serializeUpdate(stmt);
    if ("GO" in stmt) return "GO";
    if ("--" in stmt) return `-- ${stmt["--"]}`;
    throw new Error("Unknown statement type");
  }).join("\n");
}
```

When implemented, the SQL serializer is invoked via content negotiation (Accept header) and reads the intermediary format as input.
The proposed design does not include a `~sql` pipe/operator and does not branch on database dialect.

---

## 3. Data Model

### 3.1 Receipt Export Model

Based on existing `GekasReceiptODBCExporter` patterns:

```
HeadsReceipt (parent)
├── HeadsProductReceiptItem (1:N)
├── HeadsPaymentReceiptItem (1:N)
├── HeadsMoneyReceiptItem (1:N)
├── HeadsReceiptCustomer (1:1)
└── HeadsReceiptRowDiscountDistribution (1:N via RowObjectId)
```

### 3.2 Table Schemas

**HeadsReceipt:**

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| ReceiptObjectID | BIGINT | No | Primary key |
| SellerID | VARCHAR(128) | Yes | Seller identifier |
| SellerName | VARCHAR(255) | Yes | Seller display name |
| UserID | VARCHAR(128) | Yes | Cashier user ID |
| UserFullName | VARCHAR(255) | Yes | Cashier name |
| WorkstationID | VARCHAR(128) | Yes | Terminal/register ID |
| ReceiptID | VARCHAR(128) | Yes | Receipt number |
| ReceiptDate | DATETIME2 | Yes | Transaction timestamp |
| Roundoff | DECIMAL(18,4) | Yes | Rounding amount |
| Total | DECIMAL(18,4) | Yes | Receipt total |
| TotalDiscount | DECIMAL(18,4) | Yes | Total discounts |
| TotalTAX | DECIMAL(18,4) | Yes | Total tax |
| TotalCost | DECIMAL(18,4) | Yes | Total cost |
| Currency | VARCHAR(3) | Yes | ISO currency code |

**HeadsProductReceiptItem:**

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| ReceiptItemObjectID | BIGINT | No | Primary key |
| ReceiptObjectID | BIGINT | No | FK to HeadsReceipt |
| RowIndex | INT | Yes | Line number |
| ProductID | VARCHAR(128) | Yes | Product SKU |
| ProductName | VARCHAR(255) | Yes | Product description |
| Quantity | DECIMAL(18,4) | Yes | Quantity sold |
| Unit | VARCHAR(50) | Yes | Unit of measure |
| TaxPercent | DECIMAL(18,4) | Yes | VAT rate |
| Tax | DECIMAL(18,4) | Yes | Tax amount |
| Discount | DECIMAL(18,4) | Yes | Discount amount |
| Total | DECIMAL(18,4) | Yes | Line total |
| Currency | VARCHAR(3) | Yes | ISO currency code |

**HeadsPaymentReceiptItem:**

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| ReceiptItemObjectID | BIGINT | No | Primary key |
| ReceiptObjectID | BIGINT | No | FK to HeadsReceipt |
| RowIndex | INT | Yes | Line number |
| PaymentType | VARCHAR(128) | Yes | Payment method |
| PaymentID | VARCHAR(128) | Yes | Payment reference |
| Currency | VARCHAR(3) | Yes | ISO currency code |
| Value | DECIMAL(18,4) | Yes | Amount |

---

## 4. Output Formats

### 4.1 CSV Format

**Content-Type:** `application/vnd.ms-sqlserver.csv`

- RFC 4180 compliant
- Header row with column names
- NULL as empty fields
- Dot (.) decimal separator
- ISO 8601 dates

```csv
ReceiptObjectID,SellerID,SellerName,ReceiptDate,Total,Currency
1001,"S001","Store Stockholm","2024-12-18T10:30:00.000",1250.00,"SEK"
1002,"S001","Store Stockholm","2024-12-18T11:45:00.000",890.50,"SEK"
```

**SQL Server Import:**
```sql
BULK INSERT [target_table]
FROM 'data.csv'
WITH (FORMAT = 'CSV', FIELDQUOTE = '"', FIRSTROW = 2, CODEPAGE = '65001');
```

### 4.2 SQL INSERT Format

**Content-Type:** `application/sql`

```sql
INSERT INTO [dbo].[HeadsReceipt] (
    [ReceiptObjectID], [SellerID], [SellerName], [ReceiptDate], [Total], [Currency]
) VALUES
(1001, N'S001', N'Store Stockholm', '2024-12-18T10:30:00.000', 1250.0000, N'SEK'),
(1002, N'S001', N'Store Stockholm', '2024-12-18T11:45:00.000', 890.5000, N'SEK');
GO
```

### 4.3 SQL DELETE-INSERT Format (Sync)

**Content-Type:** `application/sql` with `mode=sync`

```sql
DELETE FROM [dbo].[HeadsReceipt] WHERE [ReceiptObjectID] IN (1001, 1002);
GO

INSERT INTO [dbo].[HeadsReceipt] (...) VALUES (...);
GO
```

### 4.4 SQL MERGE Format

**Content-Type:** `application/sql` with `mode=merge`

```sql
MERGE INTO [dbo].[HeadsReceipt] AS target
USING (VALUES
    (1, N'Product A', 29.99),
    (2, N'Product B', 49.99)
) AS source ([id], [name], [price])
ON target.[id] = source.[id]
WHEN MATCHED THEN UPDATE SET target.[name] = source.[name], target.[price] = source.[price]
WHEN NOT MATCHED THEN INSERT ([id], [name], [price]) VALUES (source.[id], source.[name], source.[price]);
```

### 4.5 REPLACE INTO (Not Supported in SQL Server)

SQL Server does not support `REPLACE INTO` (MySQL-only). In the proposed design, use:
- `mode=merge` for upsert behavior, or
- `mode=sync` for delete-then-insert.

```sql
-- Prefer MERGE for upsert in SQL Server
MERGE INTO [dbo].[HeadsReceipt] AS target
USING (VALUES (1001, N'S001', 1250.00)) AS source ([ReceiptObjectID], [SellerID], [Total])
ON target.[ReceiptObjectID] = source.[ReceiptObjectID]
WHEN MATCHED THEN UPDATE SET target.[SellerID] = source.[SellerID], target.[Total] = source.[Total]
WHEN NOT MATCHED THEN INSERT ([ReceiptObjectID], [SellerID], [Total])
VALUES (source.[ReceiptObjectID], source.[SellerID], source.[Total]);
```

---

## 5. Type Mapping

### 5.1 API to SQL Server Types

| API Type | SQL Server Type | Notes |
|----------|-----------------|-------|
| `string` | `NVARCHAR(MAX)` | Unicode |
| `strict string` | `NVARCHAR(255)` | Length-limited |
| `number` (int) | `INT` / `BIGINT` | Based on range |
| `number` (decimal) | `DECIMAL(18,4)` | Financial precision |
| `boolean` / `flag` | `BIT` | 0 or 1 |
| `date-time` | `DATETIME2(3)` | Millisecond precision |
| `date` | `DATE` | Date only |
| `uuid` / `guid` | `UNIQUEIDENTIFIER` | Standard format |
| `enum` | `NVARCHAR(50)` | String value |
| `null` | `NULL` | Literal NULL |

### 5.2 Nested Object Handling

**Flatten (default):**
```json
{ "identifiers": { "key": 123, "seedID": "abc" } }
```
→
```csv
identifiers_key,identifiers_seedID
123,abc
```

**JSON column (for depth > 2):**
```csv
identifiers
"{\"key\":123,\"seedID\":\"abc\"}"
```

### 5.3 Array Handling

**JSON column (default):**
```csv
categories
"[{\"id\":1,\"name\":\"Electronics\"}]"
```

**Separate table:**
```csv
# products.csv
id,name
1,"Widget"

# products_categories.csv
product_id,category_id,category_name
1,1,"Electronics"
```

---

## 6. API Endpoints (Planned)

> The endpoints in this section are proposed and not yet implemented.

### 6.1 SQL Export via Mapped Types

The SQL serializer consumes `SqlStatement[]` as input. Table names, schema references, and statement structure are defined within the statement objects themselves—not via serializer parameters. This design keeps the serializer dialect-agnostic and lets mapped types control the SQL output structure.

```
GET /api/v1/receipts~map(receipt-sql-export)
Accept: application/sql
```

The mapped type `receipt-sql-export` produces `SqlStatement[]` containing properly structured INSERT/MERGE/DELETE statements with embedded table names.

**Serializer settings (via Accept header parameters):**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `mode` | enum | `insert` | Hint to mapped types: `insert`, `sync`, `merge`. The serializer emits statements as-is. |
| `batchSize` | int | 1000 | Rows per batch for streaming output |

**Example:**
```
Accept: application/sql; mode=sync; batchSize=500
```

> **Note:** `tableName` and `schema` settings were considered but **intentionally omitted**. The intermediary `SqlStatement` format already contains fully-qualified table references (e.g., `"INSERT INTO": "dbo.HeadsReceipt"`), making serializer-level table/schema settings redundant and potentially conflicting.

### 6.2 Multi-Table Export (Planned)

```
GET /api/v1/receipts~sqlexport
Accept: application/zip
```

Returns ZIP archive:
- `HeadsReceipt.csv`
- `HeadsProductReceiptItem.csv`
- `HeadsPaymentReceiptItem.csv`
- `import.sql` (orchestration script)

### 6.3 Schema Export (Planned)

```
GET /api/v1/receipts~sqlschema
Accept: application/sql
```

Returns CREATE TABLE statements.

---

## 7. Streaming Architecture

### 7.1 CSV Streaming (Row-by-Row)

```
Observable<T>
  → emit(header) on first item
  → map(item => serializeRow(item))
  → toAsyncIterable()
  → HTTP Response stream
```

Memory: O(1) per row.

### 7.2 SQL INSERT Streaming (Batched)

```
Observable<T>
  → bufferCount(batchSize)
  → map(batch => serializeBatch(batch))
  → toAsyncIterable()
  → HTTP Response stream
```

Memory: O(batchSize × rowSize).

### 7.3 Batch Sizes

| Dataset Size | Batch Size | Rationale |
|--------------|------------|-----------|
| < 1,000 rows | All rows | Single statement |
| 1,000 - 100,000 | 1,000 | SQL Server optimal |
| > 100,000 | 1,000 | Prevent memory exhaustion |

---

## 8. Security

### 8.1 SQL Injection Prevention

**String escaping:**
```typescript
function escapeSqlString(value: string): string {
  return value.replace(/'/g, "''");
}
```

**Identifier escaping:**
```typescript
function escapeSqlIdentifier(name: string): string {
  return `[${name.replace(/\]/g, ']]')}]`;
}
```

### 8.2 Rules

- All string values escaped with `''`
- All identifiers wrapped in `[brackets]`
- System does NOT execute generated SQL (output only)
- Export respects API authorization

---

## 9. Error Handling

### 9.1 Error Codes

| Error Code | Description | HTTP Status |
|------------|-------------|-------------|
| `SQL_EXPORT_TYPE_ERROR` | Type conversion failed | 400 |
| `SQL_EXPORT_TRUNCATION` | Value exceeds column length | 400 |
| `SQL_EXPORT_NULL_VIOLATION` | NULL in non-nullable column | 400 |

### 9.2 Partial Export

On error mid-stream:
```sql
-- Export completed: 1000 rows
-- Error at row 1001: Type conversion failed for column 'Total'
-- Remaining rows skipped: 500
```

---

## 10. Performance Targets

| Metric | Target |
|--------|--------|
| CSV serialization | 100,000 rows/sec |
| INSERT generation | 50,000 rows/sec |
| Memory usage | O(batch size) |

---

## 11. Implementation Order

1. **Intermediary format types** - Type definitions
2. **Serializer functions** - DELETE, INSERT, MERGE, UPDATE
3. **Type inference** - JS value → SQL literal
4. **CSV serializer** - Streaming row-by-row
5. **SQL serializer** - Batched INSERT/MERGE
6. **Content-type registration** - Add to protocol
7. **API endpoints** - Single/multi-table export
8. **Tests** - Implementation + integration tests

---

## 12. References

- [RFC 4180 - CSV Format](https://tools.ietf.org/html/rfc4180)
- [SQL Server BULK INSERT](https://learn.microsoft.com/en-us/sql/t-sql/statements/bulk-insert-transact-sql)
- [SQL Server MERGE](https://learn.microsoft.com/en-us/sql/t-sql/statements/merge-transact-sql)
- [SQL Server JSON Support](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server)

---

## 13. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-12-18 | CommerceOS Team | Initial document |
