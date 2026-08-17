---
title: "Macros, Tags & Routing"
description: "Canned replies with placeholders and side-effect actions, a tag vocabulary, and a first-match-wins routing engine with a dry-run evaluator."
slug: "support-ops"
breadcrumb: "API Reference"
---

# Macros, Tags & Routing

Canned replies with placeholders and side-effect actions, a tag vocabulary, and a first-match-wins routing engine with a dry-run evaluator.

## Overview

Three operational primitives sit behind the support desk:

- **Macros** — canned replies. A macro carries a body with `{{placeholder}}` variables and, optionally, `actions` that are applied *besides* sending the text: set a status, add tags, assign a user.
- **Tags** — your tenant's tag vocabulary for tickets: a name, a display colour and a description.
- **Routing rules** — an ordered, first-match-wins triage list. Each rule ANDs a set of conditions over a fixed six-field allowlist and, when it matches, applies actions.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| Every list/read, plus `POST /routing-rules/evaluate` | `support_ops:read` |
| Every create, update, delete, plus `POST /macros/{id}/use` | `support_ops:write` |

`POST /routing-rules/evaluate` is a **dry run** — it writes nothing, so it needs only the read scope. `POST /macros/{id}/use` increments a counter, so it needs write.

---

# Macros

## The macro object

```json
{
  "id": "9b8a…",
  "tenant_id": "a0b1…",
  "name": "Refund acknowledged",
  "body": "Hi {{customer_name}}, your refund for order {{order_id}} is on its way.",
  "actions": { "set_status": "pending", "add_tags": ["refund"], "assign_to": null },
  "category": "billing",
  "is_active": true,
  "usage_count": 41,
  "created_at": "2026-08-01T10:00:00Z",
  "updated_at": "2026-08-16T14:00:00Z"
}
```

### `actions`

| Field | Type | Constraints |
| --- | --- | --- |
| `set_status` | `string \| null` | At most 32 characters, lowercase identifier |
| `add_tags` | `string[] \| null` | At most 20, each at most 64 characters, non-blank |
| `assign_to` | `UUID \| null` | Must be a user in your tenant |

Unknown keys inside `actions` are **rejected** with `422` rather than ignored, so a typo cannot silently do nothing.

## GET `/api/v1/support/ops/macros`

Ordered by `usage_count` descending, then name — so the picker shows what your team actually uses.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `category` | `string` | At most 64 characters, exact match |
| `is_active` | `boolean` | |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## POST `/api/v1/support/ops/macros`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, not blank, unique per tenant |
| `body` | `string` | Yes | 1–20,000 characters, not blank |
| `actions` | `object` | No | See above |
| `category` | `string` | No | At most 64 characters |
| `is_active` | `boolean` | No | Default `true` |

`409 A macro named '…' already exists` on a duplicate name.

## PATCH / DELETE `/api/v1/support/ops/macros/{macro_id}`

`PATCH` takes the same fields, all optional; sending `actions: null` clears the actions. `DELETE` returns `204`.

## POST `/api/v1/support/ops/macros/{macro_id}/use`

Renders the macro against a context and bumps `usage_count`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `context` | `object` | No | Default `{}`. At most 50 keys; at most 16,000 characters serialised |

```bash
curl -X POST https://api.callmissed.com/api/v1/support/ops/macros/9b8a…/use \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "context": { "customer_name": "Asha", "order_id": "A-4419" } }'
```

```json
{
  "id": "9b8a…",
  "name": "Refund acknowledged",
  "body": "Hi Asha, your refund for order A-4419 is on its way.",
  "actions": { "set_status": "pending", "add_tags": ["refund"] },
  "usage_count": 42
}
```

Placeholder syntax is `{{name}}`, where the name starts with a letter or underscore. Values are truncated to 500 characters.

> **Unknown or null-valued placeholders are left verbatim** — `{{order_id}}` stays in the text rather than becoming an empty gap. Check the rendered body before sending it to a customer, or supply every variable the macro declares.

Applying the returned `actions` is your job: this endpoint renders and counts, it does not mutate a ticket.

---

# Tags

```json
{
  "id": "7f6e…",
  "tenant_id": "a0b1…",
  "name": "refund",
  "color": "#c2410c",
  "description": "Money going back to the customer",
  "created_at": "2026-08-01T10:00:00Z",
  "updated_at": "2026-08-01T10:00:00Z"
}
```

| Method | Path | Scope |
| --- | --- | --- |
| `GET` | `/api/v1/support/ops/tags` | `support_ops:read` |
| `POST` | `/api/v1/support/ops/tags` | `support_ops:write` |
| `PATCH` | `/api/v1/support/ops/tags/{tag_id}` | `support_ops:write` |
| `DELETE` | `/api/v1/support/ops/tags/{tag_id}` | `support_ops:write` |

`GET` is ordered by name and takes `limit` (`1..500`, default `100`) and `offset` (`0..100000`). There are no filters.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–64 characters, not blank, unique per tenant |
| `color` | `string` | No | A hex triplet such as `#c2410c` or `#c40`. Stored lowercase |
| `description` | `string` | No | At most 255 characters |

Anything other than a hex triplet returns `422 color must be a hex triplet like '#aabbcc'`.

---

# Routing rules

## Vocabulary

| Concept | Allowed values |
| --- | --- |
| Condition `field` | `channel`, `priority`, `status`, `subject_contains`, `contact_email_domain`, `tag` |
| Condition `operator` | `eq`, `neq`, `contains`, `in`, `is_set`, `is_not_set` |
| `assign_strategy` | `direct`, `round_robin` |

The field list is a **closed allowlist** — an unknown field is `422`, not a silently-false condition.

### Condition value rules

| Operator | `value` |
| --- | --- |
| `in` | A non-empty array, at most 100 entries |
| `is_set` / `is_not_set` | Omitted |
| Everything else | A non-blank scalar (string at most 512 characters, number or boolean) |

## The rule object

```json
{
  "id": "2c3d…",
  "tenant_id": "a0b1…",
  "name": "Enterprise urgent → Ravi",
  "position": 0,
  "conditions": [
    { "field": "priority", "operator": "eq", "value": "urgent" },
    { "field": "contact_email_domain", "operator": "in", "value": ["acme.com", "globex.com"] }
  ],
  "assign_to_user_id": "b1f2…",
  "assign_strategy": "direct",
  "set_priority": "urgent",
  "add_tags": ["enterprise"],
  "is_active": true,
  "created_at": "2026-08-05T09:00:00Z",
  "updated_at": "2026-08-05T09:00:00Z"
}
```

All conditions on a rule are **ANDed**. Rules are evaluated in `position` order and the **first match wins** — `position` is the policy, so the list is never re-sorted for you.

## GET `/api/v1/support/ops/routing-rules`

Ordered by `position`. Takes `is_active`, `limit` (`1..500`, default `100`) and `offset` (`0..100000`).

## POST `/api/v1/support/ops/routing-rules`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, unique per tenant |
| `conditions` | `object[]` | No | Default `[]`. **At most 20 per rule** |
| `position` | `integer` | No | `0 <= position <= 100000`, default `0` |
| `assign_to_user_id` | `UUID` | No | Must be a user in your tenant |
| `assign_strategy` | `string` | No | `direct` (default) or `round_robin` |
| `set_priority` | `string` | No | At most 16 characters, lowercase identifier |
| `add_tags` | `string[]` | No | At most 20, each at most 64 characters. De-duplicated |
| `is_active` | `boolean` | No | Default `true` |

A rule with an empty `conditions` array matches everything — use it deliberately as a final catch-all at the highest `position`.

## PATCH `/api/v1/support/ops/routing-rules/reorder`

Rewrites the whole evaluation order.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `rule_ids` | `UUID[]` | Yes | 1–500 entries. Must be **every** routing rule in your tenant, exactly once |

New `position` is the array index. A partial list returns `422 rule_ids must list every routing rule in this tenant exactly once` — reordering is all-or-nothing, so two concurrent partial reorders cannot interleave into a nonsense order.

## POST `/api/v1/support/ops/routing-rules/evaluate`

A **dry run**. Nothing is written; you get back what would happen.

| Field | Type | Constraints |
| --- | --- | --- |
| `channel` | `string` | At most 64 characters |
| `priority` | `string` | At most 32 characters |
| `status` | `string` | At most 32 characters |
| `subject` | `string` | At most 1,000 characters |
| `contact_email` | `string` | At most 320 characters |
| `tags` | `string[]` | At most 100 entries |

Every field is optional. **An absent fact fails any condition that asks about it** — so send the whole picture when testing.

```bash
curl -X POST https://api.callmissed.com/api/v1/support/ops/routing-rules/evaluate \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "channel": "whatsapp",
    "priority": "urgent",
    "subject": "Cannot check out",
    "contact_email": "asha@acme.com",
    "tags": ["vip"]
  }'
```

```json
{
  "matched": true,
  "rule_id": "2c3d…",
  "rule_name": "Enterprise urgent → Ravi",
  "assign_to_user_id": "b1f2…",
  "assign_strategy": "direct",
  "set_priority": "urgent",
  "add_tags": ["enterprise"]
}
```

Only `is_active: true` rules are considered. When nothing matches you get `{"matched": false, …, "add_tags": []}` — a miss, not an error.

## PATCH / DELETE `/api/v1/support/ops/routing-rules/{rule_id}`

Same fields as create, all optional. `DELETE` returns `204`.

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `support_ops:read` / `support_ops:write` |
| `404` | Macro, tag, rule or assignee not in your tenant |
| `409` | Duplicate macro, tag or rule name |
| `422` | Over 20 conditions, an unknown condition field or operator, a bad colour, an unknown key in `actions`, an incomplete `reorder` list |

Nothing on this page consumes credits.
