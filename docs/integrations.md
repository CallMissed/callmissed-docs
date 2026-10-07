---
title: "Integrations"
description: "Connect external services your agents use (stores, calendars, CRMs, spreadsheets, payment gateways), and run Google Sheets order automations."
slug: "integrations"
breadcrumb: "API Reference"
---

# Integrations

Connect external services your agents use (stores, calendars, CRMs, spreadsheets, payment gateways), and run Google Sheets order automations.

## Overview

An **integration** is one connected account at an external service: a Shopify store, a Cal.com account, a Google Sheets login, your own Razorpay account. Once connected, its tools become available to your agents (see [Voice agent tools](/docs/voice-agent-tools)).

Credentials you send are validated against the provider **before** they are stored, are stored encrypted, and are **never returned** by any endpoint. Responses only tell you whether a credential is stored (`has_credentials`).

A **sheet automation** watches one tab of a spreadsheet on a Google Sheets connection and acts on each new order row: it sends an approved WhatsApp template and/or has an agent call the customer, then writes the result back into the sheet. The agent-side behaviour (call variables, write-back columns) is described in [Voice agent tools → Order automation from Google Sheets](/docs/voice-agent-tools).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get, catalog, Shopify OAuth status, Shopify pixel snippet, list sheet automations, automation orders, sheet columns | `integrations:read` |
| Create, update, delete, set spreadsheets, create/update/delete sheet automations | `integrations:write` |

Connecting a **sign-in (OAuth) provider** (Google Calendar & Gmail, Google Sheets, Shopify OAuth) is done in the console (**Developer → Integrations**), because it needs a person to approve it. Once connected, everything on this page works with a key.

## Providers and credentials

`POST /api/v1/integrations` takes a `provider` and a `credentials` object whose shape depends on the provider. Send only the keys listed for the provider: `calcom`, `hubspot`, `woocommerce`, `razorpay` and `manual` reject any other key with `422`.

| `provider` | How to connect | `credentials` | `external_account_id` set to |
| --- | --- | --- | --- |
| `calcom` | API key | `{ "api_key" }` | Your Cal.com username |
| `hubspot` | API key | `{ "api_key" }`: a HubSpot private-app token | (none) |
| `woocommerce` | API key | `{ "site_url", "consumer_key", "consumer_secret" }`: a read-only WooCommerce REST API key | Normalised store URL |
| `razorpay` | API key | `{ "key_id", "key_secret", "webhook_secret" }`. `key_id` must look like `rzp_live_…` or `rzp_test_…`; `webhook_secret` at least 8 characters. See [Payment requests](/docs/payment-requests) | The `key_id` |
| `shopify` | Access token, or OAuth in the console | `{ "shop_domain", "access_token" }` (custom-app Admin API token); optional `api_secret_key` | The shop's `myshopify.com` domain |
| `manual` | API key | Any of `{ "bearer_token", "api_key", "headers" }`, at least one. Used by the `http_request` tool's `connection_id` | (none) |
| `google_calendar` | OAuth in the console only | — | The Google account email |
| `google_sheets` | OAuth in the console only | — | The Google account email |

The provider makes a real authenticated call to check the credentials. If it refuses, you get `422` with the reason (e.g. `api_key is required`, `key id should look like rzp_live_… or rzp_test_…`).

Use [`GET /api/v1/integrations/catalog`](#get-apiv1integrationscatalog) for the live list of what can be connected.

## The integration object

```json
{
  "id": "6a1e…",
  "tenant_id": "a0b1…",
  "provider": "shopify",
  "name": "Main store",
  "status": "connected",
  "scopes": null,
  "external_account_id": "acme.myshopify.com",
  "has_credentials": true,
  "metadata": null,
  "created_at": "2026-09-20T06:10:00Z",
  "updated_at": "2026-09-20T06:10:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `provider` | `string` | One of the providers above |
| `name` | `string` | Your label, 1–255 characters |
| `status` | `string` | `connected`, `disconnected` or `error` |
| `scopes` | `object \| null` | Free-form object you may store with the connection |
| `external_account_id` | `string \| null` | The connected account (see table above). One connection per provider + account: connecting the same account again returns `409` |
| `has_credentials` | `boolean` | `true` when a credential is stored. The credential itself is never returned |
| `metadata` | `object \| null` | Provider-specific settings. Google Sheets: `spreadsheets` (the allowlist, see below). Razorpay: `max_amount_inr` (see [Payment requests](/docs/payment-requests)). Treat other keys as opaque |

## GET `/api/v1/integrations`

Returns an array of integration objects, newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/integrations?limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

## POST `/api/v1/integrations`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `provider` | `string` | Yes | 1–64 characters, a provider from the table above |
| `name` | `string` | Yes | 1–255 characters |
| `credentials` | `object` | Yes | Provider-specific, see above |
| `external_account_id` | `string` | No | At most 255 characters. Ignored when the provider derives its own account id |
| `scopes` | `object` | No | Stored as given |

```bash
curl -X POST https://api.callmissed.com/api/v1/integrations \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "calcom",
    "name": "Clinic bookings",
    "credentials": { "api_key": "cal_live_xxxxxxxx" }
  }'
```

Returns `201` with the integration object, `status: "connected"`.

| Status | When |
| --- | --- |
| `400 Unknown integration provider: <provider>` | `provider` is not one this API can connect |
| `409 This account is already connected for this provider` | Same provider and account already connected. Update the existing integration instead |
| `422` | The credentials are missing fields, have unknown fields, or the provider rejected them |

## GET `/api/v1/integrations/catalog`

The gallery of connectable services, annotated with what this account already has connected. No credentials.

```bash
curl https://api.callmissed.com/api/v1/integrations/catalog \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "items": [
    {
      "provider": "calcom",
      "label": "Cal.com",
      "category": "scheduling",
      "auth_kind": "apikey",
      "logo_key": "calcom",
      "tool_names": ["calcom_list_slots", "calcom_book"],
      "enabled": true,
      "aggregator": false,
      "description": "Check availability and book appointments on the call.",
      "connected": true,
      "account": "dr-mehta"
    }
  ]
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `category` | `string` | `scheduling`, `crm`, `messaging`, `productivity`, `commerce`, `payments` or `mcp` |
| `auth_kind` | `string` | `apikey` (one key), `apikey_multi` (several credential fields), `oauth` (console sign-in) or `mcp` |
| `tool_names` | `string[]` | Agent tools the connection unlocks |
| `enabled` | `boolean` | `false` means "coming soon": it cannot be connected yet |
| `connected` | `boolean` | This account has a `connected` integration for the provider |
| `account` | `string \| null` | Its `external_account_id`, when connected |

## GET `/api/v1/integrations/{integration_id}`

One integration. `404 Integration not found`.

```bash
curl https://api.callmissed.com/api/v1/integrations/{integration_id} \
  -H "Authorization: Bearer cm_your_api_key"
```

## PATCH `/api/v1/integrations/{integration_id}`

Every field is optional; only the fields you send are changed.

| Field | Type | Constraints |
| --- | --- | --- |
| `name` | `string` | 1–255 characters |
| `status` | `string` | `connected`, `disconnected` or `error` |
| `credentials` | `object` | Rotates the credential. Validated with the provider exactly as on create; `external_account_id` is re-derived |
| `scopes` | `object` | Replaces the stored object |

```bash
curl -X PATCH https://api.callmissed.com/api/v1/integrations/{integration_id} \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "credentials": { "api_key": "cal_live_newkey" } }'
```

Returns the updated integration. `404 Integration not found`; `422` if the new credentials are rejected or `status` is not one of the three values.

## PUT `/api/v1/integrations/{integration_id}/spreadsheets`

Google Sheets connections only. Sets which spreadsheets agents may use through the connection. The list **replaces** the existing one.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `spreadsheets` | `string[]` | No | At most 20. Each is a Google Sheets URL (`docs.google.com/spreadsheets/d/…`) or a bare spreadsheet id. Duplicates are dropped. `[]` clears the list |

```bash
curl -X PUT https://api.callmissed.com/api/v1/integrations/{integration_id}/spreadsheets \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "spreadsheets": ["https://docs.google.com/spreadsheets/d/1AbCdEfGhIjKlMnOpQrStUvWxYz0123456789/edit"] }'
```

Each spreadsheet is opened with the connection's own Google access before it is accepted, so it must be one the connected Google account can reach **and** has granted to CallMissed. The connection only reaches files picked in the Google file picker (console → Tools → Connected apps → Google Sheets → Choose spreadsheets). Returns the updated integration with `metadata.spreadsheets`:

```json
{
  "metadata": {
    "spreadsheets": [
      { "id": "1AbCdEfGhIjKlMnOpQrStUvWxYz0123456789", "title": "Orders 2026", "sheets": ["Orders", "Returns"] }
    ]
  }
}
```

| Status | When |
| --- | --- |
| `400 Only a Google Sheets connection has a spreadsheet list` | The integration is another provider |
| `404 Integration not found` | |
| `409 Reconnect Google Sheets and try again` | The stored Google credential can no longer be used |
| `422 Paste a Google Sheets link (docs.google.com/spreadsheets/d/…)` | An entry is neither a Sheets URL nor a spreadsheet id |
| `422 Couldn't open spreadsheet <id>. …` | The connected Google account cannot open it, or it was not picked in the file picker |

## DELETE `/api/v1/integrations/{integration_id}`

Disconnects and deletes the integration and its stored credential. Its sheet automations are deleted with it. Returns `204`. `404 Integration not found`.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/integrations/{integration_id} \
  -H "Authorization: Bearer cm_your_api_key"
```

## Shopify

A Shopify store connects either with an Admin API access token (`POST /api/v1/integrations`, `provider: "shopify"`) or with Shopify sign-in from the console.

### GET `/api/v1/integrations/shopify/oauth/status`

Whether Shopify sign-in is available, so a UI can choose between the sign-in button and the access-token form. Scope `integrations:read`.

```bash
curl https://api.callmissed.com/api/v1/integrations/shopify/oauth/status \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "configured": true }
```

### GET `/api/v1/integrations/shopify/pixel-snippet`

Returns the JavaScript to paste into Shopify admin → Settings → Customer events → Add custom pixel, so storefront events (product viewed, added to cart, checkout started and completed) reach CallMissed for that store. Scope `integrations:read`.

| Parameter | Type | Required | Notes |
| --- | --- | --- | --- |
| `integration_id` | `UUID` | Yes | A Shopify integration in your account |

```bash
curl "https://api.callmissed.com/api/v1/integrations/shopify/pixel-snippet?integration_id={integration_id}" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "snippet": "// CallMissed storefront tracking — paste into Shopify admin →\n// Settings → Customer events → Add custom pixel.\nvar CM_URL=…" }
```

The first call creates the store's pixel token and saves it on the integration; later calls return a snippet with the same token. Paste the `snippet` value as returned. `404 Shopify integration not found`; `400 Integration has no shop domain`.

## Sheet automations

One automation covers one tab of one spreadsheet. The spreadsheet must already be on the Google Sheets connection's list ([`PUT …/spreadsheets`](#put-apiv1integrationsintegration_idspreadsheets)). Each tab can have one automation.

How it behaves:

- **Columns are matched by header name**, case-insensitively and ignoring extra spaces. The header row is the first of the top 5 rows that has at least two filled cells, so a title row above the header is skipped.
- **Orders already in the tab when the automation starts are left alone.** Only rows that appear afterwards are acted on, each order key at most once.
- **COD vs prepaid.** A row is `cod` when its `payment_column` cell contains one of `cod_values` (case-insensitive); otherwise, or with no `payment_column`, it is `prepaid`. Each segment has its own WhatsApp and call switches.
- **Calls** are placed only inside `call_window_start`–`call_window_end` in `timezone_name`, and unanswered calls are retried up to `max_attempts` times.
- **Write-back.** The result is written to `status_column` and `note_column` (default `CallMissed Status` / `CallMissed Note`).

### The automation object

```json
{
  "id": "9b2c…",
  "integration_id": "6a1e…",
  "spreadsheet_id": "1AbCdEfGhIjKlMnOpQrStUvWxYz0123456789",
  "sheet": "Orders",
  "enabled": true,
  "key_column": "Order ID",
  "phone_column": "Phone",
  "name_column": "Customer",
  "payment_column": "Payment",
  "cod_values": ["cod", "cash on delivery"],
  "status_column": "CallMissed Status",
  "note_column": "CallMissed Note",
  "cod_whatsapp": true,
  "cod_call": true,
  "prepaid_whatsapp": true,
  "prepaid_call": false,
  "whatsapp_template_id": "77aa…",
  "whatsapp_params": { "1": "Customer", "2": "Order ID" },
  "bot_id": "c3d4…",
  "from_number_id": "e5f6…",
  "timezone_name": "Asia/Kolkata",
  "call_window_start": 9,
  "call_window_end": 21,
  "max_attempts": 3,
  "default_country_code": "91",
  "baseline_done": true,
  "last_polled_at": "2026-09-25T08:01:00Z",
  "last_error": null,
  "created_at": "2026-09-25T07:55:00Z",
  "updated_at": "2026-09-25T08:01:00Z",
  "agent_prompt_ready": true
}
```

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `key_column` | `string` | required | Header of the column holding the order id (the de-duplication key) |
| `phone_column` | `string` | required | Header of the customer phone column |
| `name_column` | `string \| null` | `null` | Customer name column |
| `payment_column` | `string \| null` | `null` | Payment method column. Without it every order is `prepaid` |
| `cod_values` | `string[]` | `["cod", "cash on delivery"]` | 1–10 values, each trimmed to 40 characters |
| `status_column` / `note_column` | `string` | `CallMissed Status` / `CallMissed Note` | Write-back columns |
| `cod_whatsapp` / `cod_call` | `boolean` | `true` / `true` | What to do for COD orders |
| `prepaid_whatsapp` / `prepaid_call` | `boolean` | `true` / `false` | What to do for prepaid orders |
| `whatsapp_template_id` | `UUID \| null` | `null` | An **approved** WhatsApp template in your account |
| `whatsapp_params` | `object` | `{}` | Template placeholder → column header. At most 20. Every placeholder in the template must be mapped |
| `bot_id` | `UUID \| null` | `null` | The agent that places confirmation calls. Required when any `*_call` is on |
| `from_number_id` | `UUID \| null` | `null` | An **active** phone number in your account to call from. Required when any `*_call` is on |
| `timezone_name` | `string` | `Asia/Kolkata` | IANA zone, at most 64 characters |
| `call_window_start` | `integer` | `9` | `0`–`23`, hour of day. Must be before `call_window_end` |
| `call_window_end` | `integer` | `21` | `1`–`24` |
| `max_attempts` | `integer` | `3` | `1`–`5` call attempts per order |
| `default_country_code` | `string` | `91` | 1–4 digits, prepended to a 10-digit local number |
| `enabled` | `boolean` | `true` | |
| `baseline_done` | `boolean` | read-only | `true` once the rows present at start have been recorded |
| `last_polled_at` | `datetime \| null` | read-only | Last time the tab was read |
| `last_error` | `string \| null` | read-only | The most recent problem reading or writing the tab |
| `agent_prompt_ready` | `boolean` | read-only | `false` when calls are on but the agent's prompt does not use `{{order_details}}` or `{{order_id}}`, so the agent would not know which order the call is about |

### GET `/api/v1/sheet-automations`

Returns an array of automation objects, newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl https://api.callmissed.com/api/v1/sheet-automations \
  -H "Authorization: Bearer cm_your_api_key"
```

### GET `/api/v1/sheet-automations/columns`

The detected header row of a tab, to pick column names before creating an automation.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `integration_id` | `UUID` | Yes | A Google Sheets integration in your account |
| `spreadsheet_id` | `string` | Yes | 10–128 characters. Must be on the connection's spreadsheet list |
| `sheet` | `string` | Yes | Tab name, 1–255 characters |

```bash
curl "https://api.callmissed.com/api/v1/sheet-automations/columns?integration_id={integration_id}&spreadsheet_id=1AbCdEfGhIjKlMnOpQrStUvWxYz0123456789&sheet=Orders" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "header_row": 2, "columns": ["Order ID", "Customer", "Phone", "Payment", "Total"] }
```

`404 Google Sheets connection not found`; `422 Choose this spreadsheet in the Google Sheets connection first`; `422 Couldn't read the tab '<sheet>'`; `409 Reconnect Google Sheets and try again`.

### POST `/api/v1/sheet-automations`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `integration_id` | `UUID` | Yes | A Google Sheets integration in your account |
| `spreadsheet_id` | `string` | Yes | 10–128 characters, on the connection's spreadsheet list |
| `sheet` | `string` | Yes | Tab name, 1–255 characters |
| `key_column` | `string` | Yes | 1–255 characters, must be in the header |
| `phone_column` | `string` | Yes | 1–255 characters, must be in the header |
| Any other field from the object table | | No | Defaults as listed |

```bash
curl -X POST https://api.callmissed.com/api/v1/sheet-automations \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "integration_id": "6a1e0000-1111-2222-3333-444455556666",
    "spreadsheet_id": "1AbCdEfGhIjKlMnOpQrStUvWxYz0123456789",
    "sheet": "Orders",
    "key_column": "Order ID",
    "phone_column": "Phone",
    "name_column": "Customer",
    "payment_column": "Payment",
    "whatsapp_template_id": "77aa0000-1111-2222-3333-444455556666",
    "whatsapp_params": { "1": "Customer", "2": "Order ID" },
    "bot_id": "c3d40000-1111-2222-3333-444455556666",
    "from_number_id": "e5f60000-1111-2222-3333-444455556666"
  }'
```

Returns `201`. The whole automation is checked against the **live** sheet and your account before it is saved:

| Status | When |
| --- | --- |
| `409 This tab already has an automation` | Update the existing one instead |
| `409 Reconnect Google Sheets and try again` | The stored Google credential can no longer be used |
| `422 Google Sheets connection not found` | `integration_id` is not a Google Sheets integration in your account |
| `422 Choose this spreadsheet in the Google Sheets connection first` | `spreadsheet_id` is not on the connection's list |
| `422 Couldn't read the tab '<sheet>'. Check the tab name.` | |
| `422 <Order ID / Phone / Name / Payment> column '<name>' is not in the header (row N): …` | A mapped column does not exist. The message lists the header |
| `422 WhatsApp template not found` / `That WhatsApp template is not approved by Meta yet` | |
| `422 Choose a column for template variable(s): …` | A template placeholder has no column in `whatsapp_params` |
| `422 Template variable <p>: column '<name>' not in the header` | |
| `422 Choose the agent and the phone number to call from` | A `*_call` switch is on without `bot_id` and `from_number_id` |
| `422 Agent not found` / `Phone number not found or not active` | |
| `422 The calling window must start before it ends` | |
| `422 timezone_name must be an IANA zone like Asia/Kolkata` | |

### PATCH `/api/v1/sheet-automations/{automation_id}`

Accepts any field from the object table except `integration_id`, `spreadsheet_id` and `sheet` (the tab an automation watches cannot be changed; delete it and create a new one). Send only the fields you want to change. An empty string clears an optional column such as `payment_column`.

When the automation is (or stays) `enabled`, the result is validated exactly as on create and the same `422`/`409` errors apply. Sending `{"enabled": false}` pauses it without re-checking the sheet.

```bash
curl -X PATCH https://api.callmissed.com/api/v1/sheet-automations/{automation_id} \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "prepaid_call": true, "call_window_end": 20 }'
```

Returns the updated automation. `404 Automation not found`.

### GET `/api/v1/sheet-automations/{automation_id}/orders`

The orders this automation has acted on, newest first. Rows that were already in the tab when it started are not listed.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/sheet-automations/{automation_id}/orders?limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "total": 1,
  "items": [
    {
      "id": "0d1e…",
      "order_key": "#1042",
      "segment": "cod",
      "phone_e164": "+919812345678",
      "whatsapp_status": "sent",
      "whatsapp_error": null,
      "call_status": "done",
      "call_attempts": 1,
      "call_error": null,
      "outcome": "confirmed",
      "note": "Customer confirmed delivery address.",
      "sheet_status": "Confirmed",
      "sheet_error": null,
      "created_at": "2026-09-25T08:03:00Z",
      "updated_at": "2026-09-25T08:09:00Z"
    }
  ]
}
```

| Field | Values |
| --- | --- |
| `segment` | `cod` or `prepaid` |
| `whatsapp_status` | `null` (not wanted), `pending`, `sending`, `sent`, `failed`, `skipped` |
| `call_status` | `null` (not wanted), `pending`, `calling`, `answered`, `done`, `failed` |
| `outcome` | `confirmed`, `cancelled`, `reschedule`, `unclear`, `no_answer` or `null` |
| `sheet_status` | The text last written to the status column, e.g. `Confirmed`, `No answer`, `Call scheduled` |
| `sheet_error` | Why the last write-back failed, if it did |

`404 Automation not found`.

### DELETE `/api/v1/sheet-automations/{automation_id}`

Stops the automation and deletes it with its order history. Returns `204`. `404 Automation not found`.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/sheet-automations/{automation_id} \
  -H "Authorization: Bearer cm_your_api_key"
```

## Errors

| Status | When |
| --- | --- |
| `400` | Unknown provider, or a Google Sheets-only endpoint called on another provider |
| `403` | Key is missing `integrations:read` / `integrations:write` |
| `404` | Integration, automation or Shopify integration is not in your account |
| `409` | Account already connected, tab already automated, or the stored Google credential needs reconnecting |
| `422` | Invalid or rejected credentials, an unreadable spreadsheet or tab, or an automation setting that does not match the sheet or your account |

Managing integrations and automations consumes no credits. The WhatsApp messages and calls an automation makes are billed like any other message or call.
