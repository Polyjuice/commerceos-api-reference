# Stock & Inventory Examples

Curl examples for stock places, stock adjustments, and stock resets.

**Base URL:** `http://localhost:5000/api/v1`
**API Key:** `banana` (passed via Basic Auth with empty username: `-u ":banana"`)

> **See also:** [Examples Index](../examples.md) | [Reference Documentation](../../reference/)

---

## Stock Places

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
# NOTE: The parent setter requires the database key (identifiers.key), not external IDs.
# Step 1: Get the parent's database key
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-places/com.myapp.stockPlaceId=WH-001/identifiers/key"
# Returns the database key, e.g., "abc123..."

# Step 2: Create the child using the database key
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-places" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.stockPlaceId": "WH-001-A"},
    "name": "Zone A",
    "parent": {"identifiers": {"key": "abc123..."}}
  }'

# Update stock place
curl -X PATCH -u ":banana" "localhost:5000/api/v1/stock-places/com.myapp.stockPlaceId=WH-001" \
  -H "Content-Type: application/json" \
  -d '{"name": "Central Distribution Warehouse"}'
```

> **Important:** The `parent` setter on stock places only accepts the database key via `identifiers.key`. External identifiers (like `com.myapp.stockPlaceId`) are not supported for relationship setters. Always fetch the parent's database key first using `/identifiers/key`.

---

## Stock Adjustments

Stock adjustments use an `items[]` array to adjust one or more products in a single transaction. Each item specifies a `product`, `place`, `reason`, and optional `quantity` (defaults to 1).

```bash
# List all stock adjustments
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-adjustments"

# List stock adjustment reasons
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-adjustment-reasons"

# Create stock adjustment reason
# NOTE: Only identifiers, name, active, and direction are supported - no description field.
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-adjustment-reasons" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.reasonId": "DAMAGED"},
    "name": "Damaged Goods",
    "active": true,
    "direction": "Decrease"
  }'

# Create another adjustment reason (increase direction)
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-adjustment-reasons" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.reasonId": "RESTOCK"},
    "name": "Restock",
    "active": true,
    "direction": "Increase"
  }'

# Create stock adjustment (increase) - single item
# Required fields: timestamp, items[] (each with product, place, reason)
# Optional: owner (auto-inferred from place if omitted), items[].quantity (defaults to 1)
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-adjustments" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.adjustmentId": "ADJ-001"},
    "timestamp": "2024-03-15T10:30:00Z",
    "items": [
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
        "place": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}},
        "reason": {"identifiers": {"com.myapp.reasonId": "RESTOCK"}},
        "quantity": 100
      }
    ]
  }'

# Create stock adjustment (decrease) - negative quantity
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-adjustments" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.adjustmentId": "ADJ-002"},
    "timestamp": "2024-03-15T14:00:00Z",
    "items": [
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
        "place": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}},
        "reason": {"identifiers": {"com.myapp.reasonId": "DAMAGED"}},
        "quantity": -5
      }
    ]
  }'

# Create stock adjustment with multiple items (bulk adjustment)
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-adjustments" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.adjustmentId": "ADJ-003"},
    "timestamp": "2024-03-16T09:00:00Z",
    "owner": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "items": [
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
        "place": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}},
        "reason": {"identifiers": {"com.myapp.reasonId": "RESTOCK"}},
        "quantity": 50
      },
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-002"}},
        "place": {"identifiers": {"com.myapp.stockPlaceId": "WH-001"}},
        "reason": {"identifiers": {"com.myapp.reasonId": "RESTOCK"}},
        "quantity": 25
      }
    ]
  }'

# Get stock adjustment items
curl -X GET -u ":banana" "localhost:5000/api/v1/stock-adjustments/com.myapp.adjustmentId=ADJ-001~with(items)"
```

> **Stock Adjustment Fields:**
> - `timestamp` (required): The time of the adjustment (ISO 8601)
> - `items[]` (required): Array of adjustment items, each containing:
>   - `product` (required): Product reference
>   - `place` (required): Stock place reference
>   - `reason` (required): Stock adjustment reason reference
>   - `quantity` (optional, defaults to 1): Positive for increase, negative for decrease
>   - `instance` (optional): Instance data for serialized products
> - `owner` (optional): Auto-inferred from the first item's place if omitted

---

## Stock Resets

Stock reset is a method endpoint (`POST /v1/stock-reset`) that creates a StockReset adjustment across all stock places owned by the specified agent (optionally limited to a single product). It offsets any specified excess/deficit on each stock place account to reconcile balances.

```bash
# Reset all stock for an owner (offsets specified excess/deficit across all stock places)
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-reset" \
  -H "Content-Type: application/json" \
  -d '{
    "owner": {"identifiers": {"com.heads.seedID": "ourcompany"}}
  }'

# Reset stock for a specific product only
curl -X POST -u ":banana" "localhost:5000/api/v1/stock-reset" \
  -H "Content-Type: application/json" \
  -d '{
    "owner": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "product": {"identifiers": {"com.myapp.sku": "SKU-001"}}
  }'
```

> **Stock Reset Fields:**
> - `owner` (required): The agent whose stock is to be reset
> - `product` (optional): Limit reset to a specific product; if omitted, all products are reset
>
> Returns `{ "success": true }` on completion.
>
> **Note:** Stock reset does NOT accept `stockPlace` or `quantity` parameters. It operates on all stock places under the owner's stock roots (optionally limited to `product`) and creates adjustment items to offset specified excess/deficit, rather than deleting transactions or aligning "physical" to "available".
>
> **Error responses:**
> - `400 Bad Request` with `"Missing argument: owner"` if owner is not provided
> - `400 Bad Request` with `"Stock owner not found"` if the specified owner does not exist
> - `400 Bad Request` with `"Product not found"` if the specified product does not exist
