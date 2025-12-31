# Email Integrations (SMTP + Microsoft Graph)

## Overview

CommerceOS provides unified email delivery supporting both SMTP and Microsoft Graph (MS Graph) providers. A single **Email** settings page lets administrators enable providers using per-provider **active** toggles.

Email functionality is split across three packages:
- `@commerceos/connector-smtp` – SMTP send logic (exports helpers only)
- `@commerceos/connector-msgraph` – MS Graph send logic (exports helpers only)
- `@commerceos/email` – Orchestrator that registers the unified SendEmailTask handler

---

## Settings Page

- **Route:** `/cos/integrations/email`
- **Menu location:** Integrations → Email

The page has:

1. **Provider Status Summary** – Shows which providers are enabled and which is active
2. **Tabbed Configuration** – SMTP and Microsoft Graph settings in separate tabs, each with an **Enable** toggle

### Provider Active Flags

Each provider has an **active** flag in its configuration:

| Provider | Toggle Label | Config Class |
|----------|--------------|--------------|
| SMTP | "Enable SMTP" | `SMTPConfig.active` |
| MS Graph | "Enable Microsoft Graph" | `MSGraphMailConfig.active` |

When a provider is enabled (active=true), its configuration **must** be valid or email sending will fail hard.

### SMTP Tab Fields

- Server URL, Username
- Password (basic auth, shown when OAuth2 disabled)
- OAuth2/XOAUTH2 toggle
- OAuth2 Client ID, Client Secret, Tenant ID (shown when OAuth2 enabled)
- Default sender email and name
- STARTTLS requirement toggle

### Microsoft Graph Tab Fields

- Tenant ID, Client ID, Client Secret
- Sender email and name
- Save to Sent Items toggle

---

## Configuration

Provider selection is controlled by the **active** flag on each provider's config:

| Config Class | Active Flag | Required Fields |
|--------------|-------------|-----------------|
| `SMTPConfig` | `active` | `url`, `username`, credentials |
| `MSGraphMailConfig` | `active` | `tenantId`, `clientId`, `clientSecret` |

- If a provider is **active** (flag is true), its configuration must be valid or sending fails hard.
- If a provider is **inactive**, it is ignored even if partially configured.
- The legacy `EmailConfig.provider` field is no longer used; active flags determine provider selection.

---

## Runtime Behavior

### Connector Roles

- `@commerceos/connector-smtp` contains SMTP send logic and configuration.
- `@commerceos/connector-msgraph` contains MS Graph send logic and configuration.
- `@commerceos/email` registers the **only** SendEmailTask handler and orchestrates provider selection.

### Orchestrator Selection Rules

1. If MS Graph is **active**, validate required fields and send via MS Graph; if invalid, fail hard.
2. Else if SMTP is **active**, validate required fields and send via SMTP; if invalid, fail hard.
3. If neither provider is active, fail hard with a clear configuration error.

This design ensures only one handler is active and avoids load-order conflicts.

---

## Error Messages

Configuration errors direct users to the unified settings page:

| Scenario | Error |
|----------|-------|
| SMTP active but invalid | "SMTP is active but not configured. Configure it in Settings → Integrations → Email, or disable this provider." |
| MS Graph active but invalid | "Microsoft Graph is active but not configured. Configure it in Settings → Integrations → Email, or disable this provider." |
| Neither provider active | "No email provider is active. Enable and configure at least one provider (SMTP or Microsoft Graph) in Settings → Integrations → Email." |

---

## Legacy Routes

The original settings pages remain for backward compatibility (bookmarks, links):

| Legacy Route | Behavior |
|--------------|----------|
| `/cos/connector-smtp/settings` | Shows redirect message with link to Email settings |
| `/cos/connector-msgraph/settings` | Shows redirect message with link to Email settings |

These routes are no longer in the admin menu. They display a warning message directing users to the new unified Email settings page.

---

## Security Notes

- Masked credential fields (password, client secrets) must not be re-saved as literal `*****` strings
- Secrets, URLs, headers, and tokens never appear in UI labels or metrics

---
