---
title: "Support Tickets"
description: "Create, filter, assign and move support tickets through their lifecycle, with server-managed response and resolution stamps."
slug: "support-tickets"
breadcrumb: "API Reference"
---

# Support Tickets

Create, filter, assign and move support tickets through their lifecycle, with server-managed response and resolution stamps.

## Overview

A **ticket** is one unit of support work. It can stand alone, or hang off a conversation and a contact so an agent sees the thread that produced it.

The lifecycle timestamps — `first_responded_at`, `resolved_at`, `closed_at`, `reopened_count` — are **server-managed**. You never send them; you change `status` and the API stamps the rest. That is what makes the [SLA endpoints](/docs/support-sla) and CSAT reporting trustworthy.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get | `support_tickets:read` |
| Create, update, assign, status, delete | `support_tickets:write` |

## Enumerations

| Field | Values |
| --- | --- |
| `status` | `open`, `pending`, `waiting_on_customer`, `resolved`, `closed` |
| `priority` | `low`, `normal`, `high`, `urgent` |

`resolved` and `closed` are the **terminal** statuses; the other three are **active**.

## The ticket object

```json
{
  "id": "e5d4…",
  "tenant_id": "a0b1…",
  "conversation_id": "c0ff…",
  "contact_id": "4411…",
  "subject": "Refund not received",
  "description": "Customer says the refund has not landed after 7 days.",
  "status": "open",
  "priority": "high",
  "assignee_user_id": null,
  "tags": ["billing", "refund"],
  "first_responded_at": null,
  "resolved_at": null,
  "closed_at": null,
  "reopened_count": 0,
  "created_at": "2026-08-17T06:10:00Z",
  "updated_at": "2026-08-17T06:10:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `assignee_user_id` | `UUID \| null` | `null` means the ticket is in the unassigned queue |
| `tags` | `string[] \| null` | Trimmed, blanks dropped, de-duplicated, order preserved |
| `first_responded_at` | `datetime \| null` | Stamped **once**, the first time the ticket leaves `open`. Never re-stamped |
| `resolved_at` | `datetime \| null` | Stamped on the move to `resolved` |
| `closed_at` | `datetime \| null` | Stamped on the move to `closed` |
| `reopened_count` | `integer` | Incremented each time a terminal ticket returns to an active status |

## GET `/api/v1/support/tickets`

Newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `status` | `string` | No | One of the five statuses |
| `priority` | `string` | No | One of the four priorities |
| `assignee_user_id` | `UUID` | No | One agent's queue |
| `contact_id` | `UUID` | No | |
| `conversation_id` | `UUID` | No | |
| `unassigned` | `boolean` | No | `true` = no assignee, `false` = has one. Cannot be combined with `assignee_user_id` |
| `q` | `string` | No | At most 255 characters. Case-insensitive substring on `subject` |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/support/tickets?status=open&unassigned=true&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

Sending both `unassigned=true` and `assignee_user_id` returns `422 unassigned=true cannot be combined with assignee_user_id` — the two contradict each other, so the API refuses rather than silently picking one.

## POST `/api/v1/support/tickets`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `subject` | `string` | Yes | 1–255 characters, not blank |
| `description` | `string` | No | At most 20,000 characters |
| `conversation_id` | `UUID` | No | Must exist in your tenant |
| `contact_id` | `UUID` | No | Must exist in your tenant |
| `assignee_user_id` | `UUID` | No | Must be a user in your tenant |
| `status` | `string` | No | Default `open` |
| `priority` | `string` | No | Default `normal` |
| `tags` | `string[]` | No | At most 20 tags, each at most 64 characters |

```bash
curl -X POST https://api.callmissed.com/api/v1/support/tickets \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Refund not received",
    "description": "Customer says the refund has not landed after 7 days.",
    "conversation_id": "c0ffee00-1111-2222-3333-444455556666",
    "priority": "high",
    "tags": ["billing", "refund"]
  }'
```

Returns `201`. Creating a ticket directly as `resolved` or `closed` stamps the matching timestamp immediately.

`404 Conversation not found` / `Contact not found` / `Assignee not found` when a linked id is not in your tenant.

## GET `/api/v1/support/tickets/{ticket_id}`

One ticket. `404 Ticket not found`.

## PATCH `/api/v1/support/tickets/{ticket_id}`

Accepts the same editable fields as create. The lifecycle timestamps and `reopened_count` are **not** accepted — a status change here runs the same transition rules as the dedicated status endpoint.

`subject` sent as blank or `null` returns `422 subject must not be blank`. An explicit `null` for `status` or `priority` is ignored rather than written.

## POST `/api/v1/support/tickets/{ticket_id}/assign`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `assignee_user_id` | `UUID \| null` | No | `null` unassigns and returns the ticket to the queue |

## POST `/api/v1/support/tickets/{ticket_id}/status`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `status` | `string` | Yes | One of the five statuses |

**Idempotent.** Sending the status the ticket already has returns it untouched — no re-stamp, no `reopened_count` increment. Safe to retry.

### Transition rules

| Move | Effect |
| --- | --- |
| Same status | No-op |
| → `resolved` | Stamps `resolved_at` if unset, clears `closed_at` |
| → `closed` | Stamps `closed_at`, leaves `resolved_at` alone |
| Terminal → active | Clears both stamps, `reopened_count += 1` |
| Leaving `open` for the first time | Stamps `first_responded_at` once |

## DELETE `/api/v1/support/tickets/{ticket_id}`

Returns `204`. `404 Ticket not found`.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `support_tickets:read` / `support_tickets:write` |
| `404` | Ticket, conversation, contact or assignee is not in your tenant |
| `422` | Unknown status/priority, blank subject, or `unassigned` combined with `assignee_user_id` |

Nothing on this page consumes credits.
