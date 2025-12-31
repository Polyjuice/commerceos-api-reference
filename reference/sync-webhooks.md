# Sync Webhooks

Sync webhooks are automated data synchronization tasks that pull data from the API, optionally transform it, and push it to an external endpoint. They support scheduling, retries, and post-sync mutations.

---

## Overview

Use sync webhooks when you need to:
- **Push data to external systems** on a schedule (e.g., sync products to an external catalog)
- **Transform data** before sending using mapped types
- **Apply mutations** to synced objects after successful delivery
- **Retry failed deliveries** with configurable attempts

**Execution flow:**
1. **Pull** - Fetch data from the API using the `in` request configuration
2. **Map** - Optionally transform the response using a mapped type
3. **Push** - Send each item to the external endpoint defined in `out`
4. **Mutate** - Apply `then` actions to update the source objects
5. **Reschedule** - If `repeat` is enabled, schedule the next run

---

## Resource & Endpoint

Sync webhooks are managed via the `/v1/sync-webhooks` collection.

| Endpoint | Description |
|----------|-------------|
| `GET /v1/sync-webhooks` | List all sync webhooks |
| `GET /v1/sync-webhooks/{key}` | Get a specific sync webhook |
| `POST /v1/sync-webhooks` | Create a new sync webhook |
| `PUT /v1/sync-webhooks/{key}` | Create or update a sync webhook |
| `PATCH /v1/sync-webhooks/{key}` | Partially update a sync webhook |
| `DELETE /v1/sync-webhooks/{key}` | Delete a sync webhook |

**Lookup by external identifier:**
```bash
GET /v1/sync-webhooks/com.myapp.hookId=product-sync
```

---

## Configuration Fields

### Core Fields

| Field | Type | Description |
|-------|------|-------------|
| `identifiers` | object | Common identifiers for the webhook (includes `key` and external IDs) |
| `name` | string | Human-readable name for the webhook |
| `description` | string | Description of what the webhook does |

### Input/Output Configuration

| Field | Type | Description |
|-------|------|-------------|
| `in` | request | The API request to fetch source data |
| `out` | request | The external request to send transformed data |
| `map` | mapped type | Reference to a mapped type for data transformation |
| `then` | object | Actions to perform on source objects after successful sync |

### Scheduling

| Field | Type | Description |
|-------|------|-------------|
| `when` | string | API request URL that returns a date-time for the next run |
| `repeat` | boolean | Whether to reschedule after each successful run |
| `next` | date-time (read-only) | The next scheduled execution time |
| `last` | date-time (read-only) | The last execution time |

### Retry Configuration

| Field | Type | Description |
|-------|------|-------------|
| `attempts` | number | Current retry count since last successful run |
| `maxAttempts` | number | Maximum retry attempts before stopping |
| `error` | string (read-only) | Error message from the last failed attempt |

### Security & Logging

| Field | Type | Description |
|-------|------|-------------|
| `authorizedScopes` | string[] | API scopes the webhook is authorized to use |
| `verboseLogging` | boolean | Enable detailed logging for debugging |

---

## Request Configuration

Both `in` and `out` use a request object with the following structure:

```json
{
  "method": "POST",
  "url": "https://api.example.com/receive",
  "auth": { ... },
  "headers": { ... },
  "body": { ... }
}
```

### Request Fields

| Field | Type | Description |
|-------|------|-------------|
| `method` | string | HTTP method (used for `out` only; defaults to `POST` when omitted) |
| `url` | string | Target URL (API path for `in`, external URL for `out`) |
| `auth` | object | Authentication configuration (used for `out` only) |
| `headers` | object | Additional HTTP headers (used for `out` only; `Content-Type` is always JSON) |
| `body` | object | Request body (currently ignored; sync webhooks always send the mapped item) |

**Notes:**
- `in` only uses `url`; other fields are ignored.
- `out` always sends the mapped item JSON as the request body.

### Authentication Options

**Basic Authentication:**
```json
{
  "auth": {
    "basic": {
      "username": "user",
      "password": "secret"
    }
  }
}
```

**OAuth 2.0 Client Credentials:**
```json
{
  "auth": {
    "clientCredentials": {
      "tokenUrl": "https://auth.example.com/oauth/token",
      "client_id": "your-client-id",
      "client_secret": "your-secret",
      "scope": "write:data"
    }
  }
}
```

**Authorization Header:**
```json
{
  "auth": {
    "authorizationHeader": "Bearer static-token"
  }
}
```

**Custom Header:**
```json
{
  "auth": {
    "customHeader": {
      "X-API-Key": "your-api-key"
    }
  }
}
```

---

## Execution Flow

### 1. Fetch Source Data (`in`)

The webhook fetches data from the API using the `in` configuration:
```json
{
  "in": {
    "url": "api/v1/products~where(status=Active)"
  }
}
```

The URL is relative to the API root. Standard query operators (`~where`, `~take`, etc.) are supported.

### 2. Transform Data (`map`)

If a mapped type is specified, each item is transformed before sending:
```json
{
  "map": {
    "identifiers": { "com.myapp.id": "product-transform" },
    "body": {
      "sku": "identifiers/com.myapp.sku",
      "title": "name",
      "price": "prices/default/amount"
    }
  }
}
```

See [Mapped Types](mapped-types.md) for the full selector language.

### 3. Send to External Endpoint (`out`)

Each transformed item is sent to the external endpoint:
```json
{
  "out": {
    "method": "POST",
    "url": "https://external-api.example.com/products",
    "auth": {
      "basic": { "username": "api", "password": "secret" }
    }
  }
}
```

The webhook sends each item as a separate request with `Content-Type: application/json`.

### 4. Apply Post-Sync Actions (`then`)

After successful delivery, the `then.set` object is applied to each source object:
```json
{
  "then": {
    "set": {
      "identifiers": {
        "com.myapp.synced": "true"
      }
    }
  }
}
```

This allows you to mark objects as synced or update tracking fields.

### 5. Reschedule

If `repeat` is `true`, the webhook reschedules based on the `when` expression. If `repeat` is `false`, the webhook sets `when` to `"never"` after completion.

---

## Scheduling with `when`

The `when` field contains an API request URL that returns a date-time value. The webhook uses this to determine when to run next.

**Common patterns:**

```bash
# Run 5 seconds from now (useful for testing)
api/v1/now/+=0:0:5

# Run at the next midnight
api/v1/now/+=1:0:0:0/floor(day)

# Run at a specific time each day
api/v1/config/com.myapp.sync-time/value
```

The date-time result determines the next execution. If the webhook fails to resolve the schedule, it sets `error` and stops (`when` becomes `"never"`).

---

## Retries & Failure Semantics

When a webhook execution fails:

1. **Error recorded** - The `error` field is set with the failure message
2. **Attempts incremented** - The `attempts` counter increases by 1
3. **Retry scheduled** - If `attempts < maxAttempts`, a retry is scheduled for 1 minute later
4. **Stopped** - If `attempts >= maxAttempts`, the webhook stops (`when` becomes `"never"`)

**Successful execution:**
- Resets `attempts` to 0
- Clears `error`
- Reschedules if `repeat` is `true`

**Common failure causes:**
- External endpoint unreachable or returns non-2xx status
- Authentication failure (expired credentials, wrong scope)
- Invalid `in` query (unauthorized scopes, malformed URL)
- Invalid `when` expression (doesn't return a date-time)

---

## Security & Safety

### Authorized Scopes

Sync webhooks run with restricted permissions. The `authorizedScopes` field defines which API scopes the webhook can use. These are intersected with the creating client's scopes.

```json
{
  "authorizedScopes": ["read:products", "write:products"]
}
```

If no scopes are authorized, the webhook cannot execute.

### Credential Security

- **Never log credentials** - Passwords, secrets, and tokens are masked in logs (`********`)
- **Use OAuth 2.0** when possible - Tokens are cached and refreshed automatically
- **Avoid secrets in `in` URLs** - Use authorized scopes instead of embedded credentials
- **Scope hygiene** - Only authorize the minimum scopes needed

### Payload Safety

- Ensure external endpoints use HTTPS
- Validate that `out` URLs point to trusted destinations
- Review mapped type transformations to avoid leaking sensitive data

---

## Examples

### Minimal Sync: Push Inventory to External API

Push latest inventory items to an external system without any transformation:

```json
{
  "identifiers": { "com.example.syncId": "inventory-push" },
  "name": "Inventory Push",
  "description": "Push latest inventory items to external system",
  "when": "api/v1/now/+=0:5:0",
  "repeat": true,
  "in": {
    "url": "api/v1/products~with(stockLevels)"
  },
  "out": {
    "method": "POST",
    "url": "https://example.com/webhooks/inventory",
    "auth": {
      "authorizationHeader": "Bearer REPLACE_WITH_TOKEN"
    }
  },
  "authorizedScopes": ["read:products"],
  "verboseLogging": false
}
```

### Full Example: Map, Sync, and Mark as Synced

A complete webhook with mapping, retries, and post-sync mutation:

```json
{
  "identifiers": { "com.example.syncId": "people-sync" },
  "name": "People Sync",
  "description": "Map people into external schema and mark synced flag",
  "when": "api/v1/now/+=0:10:0",
  "repeat": true,
  "in": {
    "url": "api/v1/people~with(identifiers,addresses,contactMethods)"
  },
  "map": {
    "identifiers": { "mappedTypeName": "com.example.people-export" }
  },
  "out": {
    "method": "POST",
    "url": "https://example.com/webhooks/people",
    "auth": {
      "basic": { "username": "api", "password": "REPLACE_WITH_PASSWORD" }
    }
  },
  "then": {
    "set": {
      "properties": {
        "dynamic": {
          "com.example.synced": true
        }
      }
    }
  },
  "authorizedScopes": ["read:people", "write:people"],
  "verboseLogging": true
}
```

### Testing with a Sandbox Endpoint

For development and testing, use a request logging service:

```json
{
  "identifiers": { "com.myapp.id": "test-webhook" },
  "name": "Test Webhook",
  "when": "api/v1/now/+=0:0:10",
  "repeat": false,
  "in": {
    "url": "api/v1/products~take(5)"
  },
  "out": {
    "method": "POST",
    "url": "https://webhook.site/your-unique-id"
  },
  "authorizedScopes": ["read:products"],
  "verboseLogging": true
}
```

**Tips for testing:**
- Use `verboseLogging: true` to see detailed execution logs
- Set `repeat: false` for one-time test runs
- Use short `when` intervals (e.g., `+=0:0:10` for 10 seconds)
- Check the `error` field if the webhook stops unexpectedly

### OAuth 2.0 Client Credentials Authentication

For outgoing requests that require OAuth 2.0 client credentials:

```json
{
  "method": "POST",
  "url": "https://example.com/webhooks/sync",
  "auth": {
    "clientCredentials": {
      "tokenUrl": "https://example.com/oauth2/token",
      "client_id": "REPLACE_WITH_CLIENT_ID",
      "client_secret": "REPLACE_WITH_CLIENT_SECRET",
      "scope": "webhook.write"
    }
  }
}
```

The webhook automatically fetches and caches access tokens, refreshing them when they expire.

### Pause and Stop Behavior

To pause or stop a webhook:

- **Pause/stop:** Set `"when": "never"` to prevent further executions
- **One-time run:** If `repeat` is `false`, the webhook automatically sets `when` to `"never"` after a successful run
- **Resume:** Set `when` to a valid schedule expression (e.g., `"api/v1/now/+=0:5:0"`) to restart

```bash
# Pause a webhook
PATCH /v1/sync-webhooks/com.example.syncId=my-webhook
{ "when": "never" }

# Resume a webhook
PATCH /v1/sync-webhooks/com.example.syncId=my-webhook
{ "when": "api/v1/now/+=0:5:0" }
```

---

## Troubleshooting

### Webhook Not Running

| Symptom | Cause | Solution |
|---------|-------|----------|
| `when` is `"never"` | Webhook stopped due to error or exhausted attempts | Check `error` field, fix the issue, reset `when` |
| `next` is in the past | Task queue delay or system restart | The webhook will run on next queue processing |
| No `authorizedScopes` | Missing or empty scopes | Add required scopes to `authorizedScopes` |

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| "No authorized scopes" | `authorizedScopes` is empty | Add required scopes |
| "Request to {url} failed with: 401" | Authentication failure | Check `out.auth` credentials |
| "Request to {url} failed with: 403" | Insufficient permissions | Verify external API permissions |
| "The response was not a date" | Invalid `when` expression | Ensure `when` URL returns a date-time |
| "Invalid configuration for client credentials" | Missing OAuth fields | Provide all required fields: `tokenUrl`, `client_id`, `client_secret`, `scope` |

### Checking Status

```bash
# View webhook configuration and status
GET /v1/sync-webhooks/com.myapp.id=my-webhook

# Check specific fields
GET /v1/sync-webhooks/com.myapp.id=my-webhook~just(error,attempts,last,next,when)
```

### Resetting a Failed Webhook

To restart a webhook that stopped due to errors:

```bash
# Reset attempts and set a new schedule
PATCH /v1/sync-webhooks/com.myapp.id=my-webhook
{
  "attempts": 0,
  "when": "api/v1/now/+=0:0:30"
}
```

---

## Related Documentation

- [Mapped Types](mapped-types.md) - Data transformation for sync webhooks
- [Query Operators](operators.md) - Filter and transform data in `in` requests
- [Overview](overview.md) - API authentication and basics
