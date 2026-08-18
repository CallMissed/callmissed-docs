---
title: "SLA Policies"
description: "Define response and resolution targets with business hours, read a ticket's live SLA clock, and list what is breaching."
slug: "support-sla"
breadcrumb: "API Reference"
---

# SLA Policies

Define response and resolution targets with business hours, read a ticket's live SLA clock, and list what is breaching.

## Overview

An **SLA policy** promises two things about a ticket: how fast someone will respond, and how fast it will be resolved. A policy may be scoped to one priority, or left un-scoped as the catch-all.

Deadlines are computed against the ticket's server-managed [lifecycle stamps](/docs/support-tickets), so a policy cannot be gamed by editing a timestamp.

Breach is **computed on read, never stored**. That means the numbers are always current, and it has one consequence for pagination — see the note on `/breaches`.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List policies, ticket status, breaches | `sla:read` |
| Create, update, delete a policy | `sla:write` |

## Business hours

A policy with `business_hours` only burns its clock during those hours. Omit it and the clock runs continuously.

| Field | Type | Default | Constraints |
| --- | --- | --- | --- |
| `tz` | `string \| null` | Your tenant's timezone | At most 64 characters |
| `days` | `integer[]` | `[1,2,3,4,5]` | 1–7 entries, values `1`–`7` where Monday is `1` and Sunday is `7`. Stored sorted and de-duplicated |
| `start` | `string` | `"09:00"` | `HH:MM`, 24-hour |
| `end` | `string` | `"18:00"` | `HH:MM`, must be later than `start` |

```json
{ "tz": "Asia/Kolkata", "days": [1,2,3,4,5,6], "start": "10:00", "end": "19:00" }
```

## The policy object

```json
{
  "id": "d1c2…",
  "tenant_id": "a0b1…",
  "name": "Urgent — 15 min first response",
  "priority": "urgent",
  "first_response_minutes": 15,
  "resolution_minutes": 240,
  "business_hours": null,
  "is_active": true,
  "created_at": "2026-08-10T08:00:00Z",
  "updated_at": "2026-08-10T08:00:00Z"
}
```

## GET `/api/v1/support/sla/policies`

Newest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `priority` | `string` | At most 16 characters, matched lowercase |
| `is_active` | `boolean` | |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## POST `/api/v1/support/sla/policies`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, not blank, unique per tenant |
| `priority` | `string` | No | At most 16 characters, must start with a letter. Omit for a catch-all policy |
| `first_response_minutes` | `integer` | No | `1 <= n <= 100000` |
| `resolution_minutes` | `integer` | No | `1 <= n <= 100000` |
| `business_hours` | `object` | No | See above |
| `is_active` | `boolean` | No | Default `true` |

**At least one of `first_response_minutes` / `resolution_minutes` must be set** — a policy that promises nothing is rejected with `422 set at least one of first_response_minutes / resolution_minutes`.

```bash
curl -X POST https://api.callmissed.com/api/v1/support/sla/policies \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Urgent — 15 min first response",
    "priority": "urgent",
    "first_response_minutes": 15,
    "resolution_minutes": 240
  }'
```

A duplicate name returns `409 An SLA policy named '…' already exists`.

## PATCH `/api/v1/support/sla/policies/{policy_id}`

All fields optional. The "must promise something" rule is re-checked against the **merged** result, so you cannot clear both minute fields in two steps.

## DELETE `/api/v1/support/sla/policies/{policy_id}`

Returns `204`. `404 SLA policy not found`.

## GET `/api/v1/support/sla/status/{ticket_id}`

The live clock for one ticket.

```json
{
  "ticket_id": "e5d4…",
  "policy_id": "d1c2…",
  "policy_name": "Urgent — 15 min first response",
  "first_response_due_at": "2026-08-17T06:25:00Z",
  "first_response_breached": true,
  "resolution_due_at": "2026-08-17T10:10:00Z",
  "resolution_breached": false,
  "minutes_remaining": 84
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `policy_id` | `UUID \| null` | `null` when no policy matched this ticket |
| `minutes_remaining` | `integer \| null` | Wall-clock minutes to the nearest still-running deadline. **Negative when overdue.** `null` when both clocks have stopped or no policy matched |

Policy selection: a policy scoped to the ticket's priority wins; otherwise the catch-all applies; otherwise nothing does.

`404 Ticket not found`.

## GET `/api/v1/support/sla/breaches`

Tickets currently past a deadline, oldest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `priority` | `string` | At most 16 characters |
| `include_resolved` | `boolean` | Default `false` |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

```json
[
  {
    "ticket_id": "e5d4…",
    "subject": "Refund not received",
    "status": "open",
    "priority": "urgent",
    "created_at": "2026-08-17T06:10:00Z",
    "sla": {
      "policy_id": "d1c2…",
      "policy_name": "Urgent — 15 min first response",
      "first_response_breached": true,
      "resolution_breached": false,
      "minutes_remaining": -32
    }
  }
]
```

> **Pagination behaves differently here.** Because breach is computed rather than stored, `limit` and `offset` page over the **tickets scanned**, not the breaches returned. A page can come back shorter than `limit`, or empty, and still have more behind it. Keep advancing `offset` until a page returns zero scanned rows rather than stopping at the first short page.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `sla:read` / `sla:write` |
| `404` | Policy or ticket not in your tenant |
| `409` | Duplicate policy name |
| `422` | Policy promises nothing, blank name, or `end` not later than `start` |

Nothing on this page consumes credits.
