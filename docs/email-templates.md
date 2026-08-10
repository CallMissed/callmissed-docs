---
title: "Email Templates"
description: "Save a reusable subject and body once, then send it with per-recipient substitution values."
slug: "email-templates"
breadcrumb: "Email"
---

# Email Templates

Save a reusable subject and body once, then send it with per-recipient substitution values.

## Templates

Save a reusable subject + body once, then send it with per-recipient values. Templates are tenant-scoped and managed with the same `cm_` key (email permission).

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/email/templates` | Create a template (`201`). Duplicate `name` for the same account → `409` |
| `GET /api/v1/email/templates` | List your templates, newest first. `limit` (1–200, default 50) and `offset` (≥0, default 0) |
| `GET /api/v1/email/templates/{id}` | Fetch one (`404` if not yours) |
| `PUT /api/v1/email/templates/{id}` | Partial update, only supplied fields change |
| `DELETE /api/v1/email/templates/{id}` | Delete a template (`204`) |

## Create a template

**`POST /api/v1/email/templates`**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | Yes | 1–255 chars; unique per account |
| `subject` | string | Yes | Base subject (overridable per send) |
| `html` | string | Yes | Base HTML body |
| `text` | string | No | Base plain-text body |
| `default_sender` | string | No | Used when the send omits `from` / `sender` |
| `default_reply_to` | string | No | Used when the send omits `reply_to` |
| `tags` | array | No | String tags for your own categorisation |
| `is_active` | boolean | No | Defaults to `true`; an inactive template can't be sent |

The template object returns `id`, `name`, `subject`, `html`, `text`, `default_sender`, `default_reply_to`, `tags`, `is_active`, `created_at`, `updated_at`. The `id` is a UUID.

`PUT /templates/{id}` takes the same fields, all optional, and applies only the ones you actually send.

Two write-time rejections apply to both create and update, and both come back as a `422` with a plain string `detail`:

- A control character in `subject`, `default_sender` or `default_reply_to`. Those values are rendered into raw headers at send time.
- A body or subject that references `{{ contact.anything }}`. Nothing can populate that namespace, so the reference would render as an empty string and ship a broken message. Pass the value in `params` instead. The same reference on a send is rejected as `422 unresolvable_template_vars`.

## Using a template on send

Add `templateId` and `params` to `POST /api/v1/email/send`:

:::tabs
```bash [cURL]
# Create a template
curl -X POST https://api.callmissed.com/api/v1/email/templates \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "receipt",
    "subject": "Your receipt, {{ params.name }}",
    "html": "<p>Hi {{ params.name }}, your order {{ params.order_id }} is confirmed.</p>",
    "default_sender": "Acme Ops <donotreply@acme.com>"
  }'

# Send from it - send-call fields override the template
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "templateId": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
    "to": ["Ada <customer@example.com>"],
    "params": { "name": "Ada", "order_id": "1043" }
  }'
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}

tpl = httpx.post(f"{BASE}/templates", headers=h, json={
    "name": "receipt",
    "subject": "Your receipt, {{ params.name }}",
    "html": "<p>Hi {{ params.name }}, your order {{ params.order_id }} is confirmed.</p>",
    "default_sender": "Acme Ops <donotreply@acme.com>",
}).json()

httpx.post(f"{BASE}/send", headers=h, json={
    "templateId": tpl["id"],
    "to": ["Ada <customer@example.com>"],
    "params": {"name": "Ada", "order_id": "1043"},
})
```
```javascript [JavaScript]
const BASE = "https://api.callmissed.com/api/v1/email";
const headers = {
  Authorization: "Bearer cm_your_key",
  "Content-Type": "application/json",
};

const tpl = await fetch(`${BASE}/templates`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    name: "receipt",
    subject: "Your receipt, {{ params.name }}",
    html: "<p>Hi {{ params.name }}, your order {{ params.order_id }} is confirmed.</p>",
    default_sender: "Acme Ops <donotreply@acme.com>",
  }),
}).then((r) => r.json());

await fetch(`${BASE}/send`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    templateId: tpl.id,
    to: ["Ada <customer@example.com>"],
    params: { name: "Ada", order_id: "1043" },
  }),
});
```
:::

The send returns the same `202 Accepted` body as any other send. See [Send Email](/docs/email-send#response).

- **Override rule.** The template's `subject` / `html` / `text` are the base; an explicit `subject` / `html` / `text` / `from` / `reply_to` on the send **wins**. `default_sender` / `default_reply_to` fill in only when the send omits them.
- **Substitution.** `{{ params.KEY }}` (and nested `{{ params.a.b }}`) are replaced from `params`; a missing key renders empty. Values placed into the HTML body are HTML-escaped. This is plain variable substitution and **not** a programming language: no logic, loops, or expressions, and it can only read the `params` you pass. A substituted value is inserted once and never re-scanned, so a param whose value itself contains `{{ ... }}` is not expanded again.
- **Inline substitution.** `params` also renders placeholders in a `subject` / `text` / `html` you pass directly on the send, so you can use `{{ params.KEY }}` with no `templateId` at all. Whichever value is actually used, yours or the template's, is rendered exactly once.

> **Note:** unlike Brevo's integer template id, a CallMissed `templateId` is a UUID.

## Manage templates

```bash
# List (limit / offset supported)
curl "https://api.callmissed.com/api/v1/email/templates?limit=50&offset=0" \
  -H "Authorization: Bearer cm_your_key"

# Fetch one
curl https://api.callmissed.com/api/v1/email/templates/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key"

# Partial update - only the fields you send change
curl -X PUT https://api.callmissed.com/api/v1/email/templates/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"subject": "Your updated receipt, {{ params.name }}", "is_active": true}'

# Delete (204)
curl -X DELETE https://api.callmissed.com/api/v1/email/templates/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key"
```

## Common template failures

| Status | Reason | Meaning |
|--------|--------|---------|
| 404 | `template_not_found` | The `templateId` doesn't exist or isn't yours |
| 422 | `template_inactive` | The template exists but is not active (`is_active: false`) |
| 422 | `invalid_headers` | A rendered header value contains invalid characters, usually a template `param` with a newline in it |
| 422 | `unresolvable_template_vars` | The rendered subject or body references `{{ contact.something }}` |
| 409 | *(string `detail`)* | Duplicate template name for the same account, on create **or** on a rename |

`409` and `404` on the template routes use a plain string `detail` with no `reason`; `template_not_found` and `template_inactive` on a send are nested under `detail`. See [Limits, Quotas & Errors](/docs/email-limits#response-shapes).
