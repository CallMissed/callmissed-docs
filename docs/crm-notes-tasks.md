---
title: "Notes & Tasks"
description: "Attach freeform notes to a contact, company or deal, and track follow-up work with due dates, assignees and an overdue flag."
slug: "crm-notes-tasks"
breadcrumb: "API Reference"
---

# Notes & Tasks

Attach freeform notes to a contact, company or deal, and track follow-up work with due dates, assignees and an overdue flag.

## Overview

**Notes** are freeform text attached to a contact, company or deal. **Tasks** are the work someone still has to do — with a title, an optional due date, an assignee and an optional link to a record.

Both show up in the [timeline](/docs/crm-lead-scores#timeline) for the record they attach to.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| Read notes | `crm_notes:read` |
| Write notes | `crm_notes:write` |
| Read tasks | `crm_tasks:read` |
| Write tasks | `crm_tasks:write` |

---

# Notes

## The note object

```json
{
  "id": "aa22…",
  "tenant_id": "a0b1…",
  "entity_type": "company",
  "entity_id": "5c6d…",
  "body": "Renewal call went well — wants a Hindi voice agent.",
  "author_user_id": "b1f2…",
  "created_at": "2026-08-16T12:00:00Z",
  "updated_at": "2026-08-16T12:00:00Z"
}
```

`author_user_id` is the dashboard user who wrote it, and is **`null` when the note was created with an API key** — a key is not a person.

## GET `/api/v1/crm/notes`

Newest first. Both entity parameters are **required** — notes are always read in the context of one record.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact`, `company` or `deal` |
| `entity_id` | `UUID` | Yes | Must exist in your tenant |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/crm/notes?entity_type=company&entity_id=5c6d…" \
  -H "Authorization: Bearer cm_your_api_key"
```

## POST `/api/v1/crm/notes`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact`, `company` or `deal` |
| `entity_id` | `UUID` | Yes | Must exist in your tenant |
| `body` | `string` | Yes | 1–10,000 characters, not blank |

Returns `201`. `404 Company not found` (or Contact / Deal) when the target is not yours.

## PATCH / DELETE `/api/v1/crm/notes/{note_id}`

`PATCH` edits **`body` only** — a note cannot be re-pointed at a different record, so the audit trail stays honest. `DELETE` returns `204`.

---

# Tasks

## The task object

```json
{
  "id": "bb33…",
  "tenant_id": "a0b1…",
  "title": "Send the renewal quote",
  "description": "Include the Hindi voice add-on.",
  "status": "open",
  "due_at": "2026-08-19T10:00:00Z",
  "completed_at": null,
  "assignee_user_id": "b1f2…",
  "entity_type": "company",
  "entity_id": "5c6d…",
  "created_by_user_id": "b1f2…",
  "created_at": "2026-08-16T12:05:00Z",
  "updated_at": "2026-08-16T12:05:00Z",
  "is_overdue": false
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `status` | `string` | `open` or `done` |
| `completed_at` | `datetime \| null` | **Server-managed** — not accepted on create or update |
| `is_overdue` | `boolean` | Computed: `due_at` is in the past **and** `status` is `open`. A done task is never overdue |

## GET `/api/v1/crm/tasks`

Ordered by `due_at` ascending with undated tasks last, then newest first — so the list reads as a work queue.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `status` | `string` | `open` or `done` |
| `assignee_user_id` | `UUID` | One person's queue |
| `entity_type` | `string` | `contact`, `company` or `deal` |
| `entity_id` | `UUID` | |
| `overdue` | `boolean` | |
| `due_before` | `datetime` | |
| `due_after` | `datetime` | |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/crm/tasks?status=open&overdue=true" \
  -H "Authorization: Bearer cm_your_api_key"
```

## POST `/api/v1/crm/tasks`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `title` | `string` | Yes | 1–255 characters, not blank |
| `description` | `string` | No | At most 10,000 characters |
| `status` | `string` | No | `open` (default) or `done` |
| `due_at` | `datetime` | No | |
| `assignee_user_id` | `UUID` | No | Must be a user in your tenant |
| `entity_type` | `string` | No | `contact`, `company` or `deal` |
| `entity_id` | `UUID` | No | Must exist in your tenant |

`entity_type` and `entity_id` must be **sent together** — one without the other returns `422 entity_type and entity_id must be sent together`. A task with neither is a standalone to-do.

## PATCH `/api/v1/crm/tasks/{task_id}`

Same fields, all optional. The entity link is validated against the **merged** result, so you can move a task from a contact to a company in one call without tripping the pairing rule.

## POST `/api/v1/crm/tasks/{task_id}/complete`

No body. Marks the task done and stamps `completed_at`.

**Idempotent** — completing an already-done task returns it unchanged. Safe to retry.

## GET / DELETE `/api/v1/crm/tasks/{task_id}`

`DELETE` returns `204`. `404 Task not found`.

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing the matching `crm_notes:*` / `crm_tasks:*` scope |
| `404` | Note, task, or the linked record is not in your tenant |
| `422` | Blank body/title, unknown `entity_type` or `status`, an entity pair sent half-filled, or an assignee outside your tenant |

Nothing on this page consumes credits.
