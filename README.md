# CommerceOS API Reference

Comprehensive documentation for integrating with the **CommerceOS REST API**. This reference covers HTTP usage, authentication, pagination, query operators, content negotiation, and common workflows.

> **Audience:** Developers and AI agents building integrations with the CommerceOS API.

---

## What This Repository Is

This is a **high-level reference, example database and cookbook-style guide** for AI agents and developers who need broad contextual understanding of the CommerceOS API. It covers:

- Common workflows and task recipes
- Query patterns and operator usage
- Patterns for common integrations with CommerceOS
- Pitfalls and troubleshooting
- Practical curl examples

**Why no endpoint listings?** The API serves its own documentation at each deployed instance. Endpoint schemas and model definitions are always up-to-date there, so this repository focuses on the higher-level patterns that don't change between versions.

---

## Where to Find Documentation

| Location | Description |
|----------|-------------|
| `<base-uri>/api-docs` | **Tenant-specific docs** - human-friendly, always check here first |
| `<base-uri>/openapi/spec.json` | **OpenAPI spec** - machine-readable endpoints and model definitions |
| [dev.heads.com/api-docs](https://dev.heads.com/api-docs) | Global documentation (when no tenant available) |
| [dev.heads.com/openapi/spec.json](https://dev.heads.com/openapi/spec.json) | Global OpenAPI spec |

`<base-uri>` here is the URL to the running CommerceOS instance, for example `mycompany.app.heads.com`. `dev.heads.com` hosts a public such instance for public documentation.

**Best practice:** Use the tenant's documentation when available - it reflects the specific API version and configuration for that deployment.

---

## Repository Structure

```
commerceos-api-reference/
├── getting-started/              # Onboarding and learning resources
│   └── learning-path.md          # Structured learning progression
├── guide/                        # Practical guides and examples
│   ├── agent-guide.md            # Contextual usage guide, task recipes, troubleshooting
│   ├── examples.md               # Comprehensive curl examples
│   ├── advanced-queries.md       # 300 advanced query examples
│   └── creating-skills.md        # How to create Claude skills
├── reference/                    # API reference documentation
│   ├── overview.md               # API basics, auth, endpoints
│   ├── operators.md              # Query operators (~where, ~with, etc.)
│   ├── mapped-types.md           # Data transformation
│   ├── sync-webhooks.md          # Automated data synchronization
│   ├── resource-patterns.md      # Common resource patterns
│   ├── type-members.md           # Type member reference
│   ├── common-gotchas.md         # Known pitfalls and solutions
│   ├── platform-takeaways.md     # Cross-cutting behaviors
│   └── email-integrations.md     # Email provider configuration
└── features/                     # Feature-specific documentation
    ├── sql-export.md             # SQL export specification
    └── config-import-export.md   # Config utility spec
```

---

## Quick Start

1. **New to the API?** Start with [`getting-started/learning-path.md`](getting-started/learning-path.md)
2. **How do I...?** See [`guide/agent-guide.md`](guide/agent-guide.md) for task recipes and troubleshooting
3. **Need examples?** See [`guide/examples.md`](guide/examples.md) for curl examples
4. **Advanced queries?** Check [`guide/advanced-queries.md`](guide/advanced-queries.md) for 300 examples
5. **Looking for operators?** See [`reference/operators.md`](reference/operators.md)
6. **Avoiding pitfalls?** Read [`reference/common-gotchas.md`](reference/common-gotchas.md)

---

## Key Topics

| Topic | Documentation |
|-------|---------------|
| API Overview | [`reference/overview.md`](reference/overview.md) |
| Query Operators | [`reference/operators.md`](reference/operators.md) |
| Mapped Types | [`reference/mapped-types.md`](reference/mapped-types.md) |
| Sync Webhooks | [`reference/sync-webhooks.md`](reference/sync-webhooks.md) |
| Type Members | [`reference/type-members.md`](reference/type-members.md) |
| Common Gotchas | [`reference/common-gotchas.md`](reference/common-gotchas.md) |
| Resource Patterns | [`reference/resource-patterns.md`](reference/resource-patterns.md) |
| Task Recipes | [`guide/agent-guide.md`](guide/agent-guide.md) |
| Troubleshooting | [`guide/agent-guide.md`](guide/agent-guide.md) |
| Curl Examples | [`guide/examples.md`](guide/examples.md) |
| Advanced Queries | [`guide/advanced-queries.md`](guide/advanced-queries.md) |
| Creating Skills | [`guide/creating-skills.md`](guide/creating-skills.md) |

---

## Common Patterns

### Query Operators

```bash
# Filter and paginate
GET /v1/products~where(status=Active)~take(10)

# Expand fields
GET /v1/products/com.example.sku=ABC~with(prices,categories)

# Project specific fields only
GET /v1/orders~just(identifiers,total,status)

# Apply mapped type transformation
GET /v1/receipts~map(com.heads.receipt-csv)
```

### External Identifiers

Resources support user-defined external identifiers using reverse-domain notation:

```bash
# Lookup by external ID
GET /v1/products/com.myapp.sku=WIDGET-001

# Create with external ID
PUT /v1/products
[{ "identifiers": { "com.myapp.sku": "WIDGET-001" }, "name": "Widget" }]
```

---

## Pitfalls

- Sync webhook `in` requests only use `url`; `method`, `headers`, and `body` are ignored, and `out.body` is ignored because sync webhooks always send the mapped item JSON.

---

## License

MIT License - see [LICENSE](LICENSE) for details.
