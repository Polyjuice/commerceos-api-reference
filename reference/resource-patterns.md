# CommerceOS API Resource Patterns

This document highlights repeatable resource patterns observed in the CommerceOS API.

---

## Agent Members

Agents (person, company, store) share common members:

| Member | Description | Write Semantics |
|--------|-------------|-----------------|
| `identifiers` | External IDs + key | Via common identifiers |
| `name` | Mapped from fullName property | Direct set |
| `nationality` | Agent nationality | Direct set |
| `languages` | Spoken languages | Direct set |
| `vatId` | VAT identification number | Direct set |
| `addresses` | Object with `main`, `home`, `invoice`, `delivery`, `visiting` | Each address settable |
| `contactMethods` | Object with `landlinePhone`, `mobilePhone`, `workPhone`, `email` | Each method settable |
| `confirmationAttempts` | Confirmation attempts for agent | Read-only |
| `customerGroups` | Groups owned by agent | Add/remove |
| `customerRelations` | Trade relationships where agent is supplier | Read-only |
| `supplierRelations` | Trade relationships where agent is customer | Read-only |
| `manufacturerRelations` | Manufacturer relationships | Read-only |
| `labels` | Assigned labels | Add/remove |
| `timeline` | Receipts for buyer | Read-only |
| `stockRoots` | Stock places attached to agent | Add/remove |
| `assortmentOwner` | Owner agent for assortment inheritance | Writable |
| `assortmentRoots` | Root product nodes in assortment | Add/remove |
| `assortment` | All product nodes in assortment | Add/remove |
| `preferredCurrency` | Preferred currency | Set by key |

Notes:
- `timeline` fetches receipts where agent is the buyer.
- `stockRoots` provides stock owner interface.
- `assortmentOwner` writes config at the agent node.
- Phone numbers are converted on set.
- Email addresses are converted on set.

---

## Product Node Members

Product nodes share common members:

| Member | Description | Write Semantics |
|--------|-------------|-----------------|
| `identifiers` | External IDs + key | Via common identifiers |
| `name` | Product node name | Direct set |
| `gtin` | Global Trade Item Numbers | Direct set |
| `plu` | Price Look-Up codes | Direct set |
| `hidden` | Hidden flag | Direct set |
| `createdAt` | Creation timestamp | Read-only |
| `createdBy` | Creating agent | Read-only |
| `promotionTitle` | Localized promotion title | Localized set |
| `promotionBanner` | Localized promotion banner | Localized set |
| `promotionDescription` | Localized promotion description | Localized set |
| `notesForPicking` | Picker instructions | Direct set |
| `images` | Image files | **Full replace on set** |
| `assortmentOwners` | Agents owning this node | Add/remove |
| `assortmentContexts` | Assortment relation contexts | **Nested fields writable** (see below) |
| `categories` | Category assignments with weight | Add/remove; weight settable per assignment |
| `labels` | Assigned labels | Add/remove |
| `prices` | Price rules for this product | **Replace on set** |
| `xrefs.compatibles` | Compatible products | Add/remove |
| `xrefs.alternatives` | Alternative products | Add/remove |
| `application.webshop` | Webshop settings | Nested fields writable |

### assortmentContexts

The collection itself is read-only, but **each context element has writable nested fields**:

| Nested Field | Description | Write Semantics |
|--------------|-------------|-----------------|
| `owner` | Owning agent | Read-only |
| `articleNumber` | Custom article number for this owner | Settable |
| `minimumOrderQuantity` | Minimum order quantity | Settable |
| `primarySupplier` | Primary supplier company | Settable |

When setting nested fields, the relation is created if it doesn't exist.

### categories

Uses a wrapper object pattern:
- `category`: The assigned category
- `weight`: Child weight in the category

Weights stored on the category relation.

### prices

Gets prices specific to this node.
- Replace on set means setting the array replaces all existing prices
- On add, price patterns are updated
- On remove, if no other products remain, price is purged entirely
