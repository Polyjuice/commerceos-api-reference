# Trade Orders & Fulfillment Examples

Curl examples for trade orders, trade relationships, shipments, and payments.

**Base URL:** `http://localhost:5000/api/v1`
**API Key:** `banana` (passed via Basic Auth with empty username: `-u ":banana"`)

> **See also:** [Examples Index](../examples.md) | [Discount Rules](./discount-rules.md) | [Reference Documentation](../../reference/)

---

## Trade Orders

> **Important Notes:**
> - `sellers` is **required** on create (determines stock source)
> - Item amounts like `unitAmountInclVat` are **read-only** (calculated from prices)
> - Item mutations (POST/PATCH/DELETE to `/items`) are **not supported** — items must be set at creation
> - Use `tryApprove`/`tryCancel` actions (not `confirm`/`cancel`)

```bash
# List all trade orders
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders"

# Get trade order by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001"

# Get trade order with items
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001~with(items)"

# Get trade order with customer and supplier
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001~with(customer,supplier)"

# Get trade order items
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/items"

# Get trade order payments
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/payments"

# Get trade order shipments
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/shipments"

# Create a sales order (sellers REQUIRED, amounts are read-only)
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-orders" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.orderId": "ORD-2024-001"},
    "supplier": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "customer": {"identifiers": {"com.myapp.customerId": "CUST-001"}},
    "sellers": [{"identifiers": {"com.heads.seedID": "store1"}}],
    "currency": {"identifiers": {"currencyCode": "SEK"}},
    "items": [
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
        "quantity": 2
      }
    ]
  }'

# Create purchase order (with sellers)
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-orders" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.orderId": "PO-2024-001"},
    "supplier": {"identifiers": {"com.myapp.supplierId": "SUPP-001"}},
    "customer": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "sellers": [{"identifiers": {"com.myapp.supplierId": "SUPP-001"}}],
    "currency": {"identifiers": {"currencyCode": "SEK"}},
    "items": [
      {
        "product": {"identifiers": {"com.myapp.sku": "SKU-001"}},
        "quantity": 100
      }
    ]
  }'

# Order actions - approve order (use tryApprove, not confirm)
curl -X PATCH -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/actions" \
  -H "Content-Type: application/json" \
  -d '{"tryApprove": true}'

# Order actions - cancel order (use tryCancel, not cancel)
curl -X PATCH -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/actions" \
  -H "Content-Type: application/json" \
  -d '{"tryCancel": true}'

# Order actions - create shipment
curl -X PATCH -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/actions" \
  -H "Content-Type: application/json" \
  -d '{"createShipment": true}'

# Order actions - create payment (requires namespaced methodId from /v1/payment-methods)
curl -X PATCH -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/actions" \
  -H "Content-Type: application/json" \
  -d '{
    "createPayment": {
      "timestamp": "2024-01-15T10:30:00Z",
      "transactionId": "TXN-001",
      "method": {"identifiers": {"com.heads.methodId": "com.heads.cash"}}
    }
  }'

# Delete order
curl -X DELETE -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001"
```

---

## Trade Relationships

```bash
# List all trade relationships
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-relationships"

# Get trade relationship by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-relationships/com.heads.seedID=rel-001"

# Get relationship with agents
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-relationships/com.heads.seedID=rel-001~with(supplierAgent,customerAgent)"

# Create a trade relationship (supplier-customer)
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-relationships" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.relId": "REL-001"},
    "supplierAgent": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "customerAgent": {"identifiers": {"com.myapp.customerId": "CUST-001"}},
    "defaultCurrency": {"identifiers": {"currencyCode": "SEK"}}
  }'

# Create relationship with payment and delivery terms
curl -X POST -u ":banana" "localhost:5000/api/v1/trade-relationships" \
  -H "Content-Type: application/json" \
  -d '{
    "identifiers": {"com.myapp.relId": "REL-002"},
    "supplierAgent": {"identifiers": {"com.heads.seedID": "ourcompany"}},
    "customerAgent": {"identifiers": {"com.myapp.companyId": "COMP-001"}},
    "defaultCurrency": {"identifiers": {"currencyCode": "SEK"}},
    "creditAllowed": true,
    "allowsBackOrder": true
  }'

# Update relationship
curl -X PATCH -u ":banana" "localhost:5000/api/v1/trade-relationships/com.myapp.relId=REL-001" \
  -H "Content-Type: application/json" \
  -d '{"creditAllowed": true}'
```

---

## Shipment Orders

> **Note:** Shipment orders are **read-only** — they are created via trade order `createShipment` action.

```bash
# List all shipment orders
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders"

# Get shipment order by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001"

# Get shipment with items
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001~with(items)"

# Get shipment's related trade orders
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001/orders"

# Get shipment records
curl -X GET -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001/records"

# Shipment finder (only modifiedTag filter is supported)
curl -X POST -u ":banana" "localhost:5000/api/v1/shipment-orders/@find" \
  -H "Content-Type: application/json" \
  -d '{"modifiedTag": "abc123"}'

# Release shipment order (only available action)
curl -X PATCH -u ":banana" "localhost:5000/api/v1/shipment-orders/com.myapp.shipmentId=SHIP-001/actions" \
  -H "Content-Type: application/json" \
  -d '{"release": true}'
```

---

## Picking Orders

> **Note:** Picking orders are **read-only** — they are created automatically as part of shipment processing.

```bash
# List all picking orders
curl -X GET -u ":banana" "localhost:5000/api/v1/picking-orders"

# Get picking order by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/picking-orders/pickingOrderId=PICK-001"
```

---

## Payment Orders

> **Note:** Payment orders are **read-only** — they are created via trade order `createPayment` action.

```bash
# List all payment orders
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-orders"

# Get payment order by ID
curl -X GET -u ":banana" "localhost:5000/api/v1/payment-orders/com.myapp.paymentOrderId=PAY-001"

# Get payment orders for a trade order
curl -X GET -u ":banana" "localhost:5000/api/v1/trade-orders/com.myapp.orderId=ORD-2024-001/payments"
```

---

## Notes

- **sellers required**: Every trade order must have `sellers` — this determines where stock is picked from
- **Read-only amounts**: Item amounts (`unitAmountInclVat`, `totalAmount`, etc.) are calculated from prices
- **No item mutations**: Cannot POST/PATCH/DELETE items after order creation — set all items at creation
- **Action names**: Use `tryApprove`/`tryCancel`, not `confirm`/`cancel`
- **Shipments/Payments**: Created via trade order actions, not POST to their respective endpoints
- **Payment methods**: Use namespaced IDs from `/v1/payment-methods` (e.g., `com.heads.cash`, `com.heads.card`)
