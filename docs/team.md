---
title: "Team & Tenants"
description: "Manage your organization (tenant): invite members, update and remove them, and transfer ownership."
slug: "team"
breadcrumb: "API Reference"
---

# Team & Tenants

Manage your organization (tenant): invite members, update and remove them, and transfer ownership.

## Overview

A **tenant** is your organization. Every user, bot, key, and conversation belongs to exactly one tenant, and all data is isolated per tenant. Roles are **owner**, **admin**, and **agent**:

| Role | Can do |
| --- | --- |
| `owner` | Everything, plus transfer ownership. One per tenant |
| `admin` | Invite, update, and remove members (not the owner); manage bots, keys, billing |
| `agent` | Operational access: handle conversations. No team or billing management |

## Credential class: dashboard JWT only

Every endpoint here resolves a **dashboard access token**. A `cm_` API key is **rejected with `401`**; team management is not part of the API key surface, whatever scopes the key carries.

```
Authorization: Bearer <jwt_access_token>
```

| Endpoint | Role required |
| --- | --- |
| `GET /api/v1/users` | Any member |
| `POST /api/v1/users` | Owner or admin. Inviting **as** `owner` is owner-only |
| `PUT /api/v1/users/{user_id}` | Owner or admin. Granting the `owner` role is owner-only |
| `DELETE /api/v1/users/{user_id}` | Owner or admin |
| `PATCH /api/v1/users/{user_id}/role` | **Owner only** |

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`; validation failures are `422`.

## Guard rails

These rules are enforced on every write and are the most common source of a `400` or `403`:

- The owner's role cannot be changed through `PUT`, and the owner cannot be deactivated or deleted.
- You cannot change your **own** role through `PUT`, or delete yourself.
- Only the existing owner can create or grant the `owner` role, on any endpoint.
- Ownership transfer goes through `PATCH /{user_id}/role` and demotes the previous owner to `admin` in the same transaction.
- A tenant may create at most **30 users in any rolling 24 hours**.

---

## GET /api/v1/users

Lists every member of your tenant, oldest first. No parameters.

```bash
curl https://api.callmissed.com/api/v1/users \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
[
  {
    "id": "u1234567-89ab-cdef-0123-456789abcdef",
    "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
    "email": "priya@acme.com",
    "full_name": "Priya Nair",
    "phone_number": "+919876543210",
    "role": "owner",
    "is_active": true,
    "created_at": "2026-05-02T09:00:00Z",
    "updated_at": "2026-07-14T08:31:02Z"
  }
]
```

## POST /api/v1/users

Invites a member and emails them a set-your-password link. Returns `201`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `email` | `string` | Yes | Valid email. Lowercased and trimmed server-side |
| `full_name` | `string` | Yes | 1-255 chars |
| `password` | `string \| null` | No | 8-72 chars. **Omit it** and the invitee sets their own via the reset-password OTP flow |
| `role` | `string` | No | One of `owner`, `admin`, `agent`. Default `agent` |

Invited members skip email verification: the inviter vouches for them.

```bash
curl -X POST https://api.callmissed.com/api/v1/users \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"email": "agent@acme.com", "full_name": "Sam Rivera", "role": "agent"}'
```

```json
{
  "id": "u7654321-ba98-fedc-3210-fedcba987654",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "email": "agent@acme.com",
  "full_name": "Sam Rivera",
  "phone_number": null,
  "role": "agent",
  "is_active": true,
  "created_at": "2026-08-04T12:10:00Z",
  "updated_at": "2026-08-04T12:10:00Z"
}
```

Writes a `user.invite` [audit event](/docs/audit-logs).

| Status | `detail` |
| --- | --- |
| `400` | `Email is required`, or `You are already a member of this organisation` |
| `403` | `Only owners/admins can invite users` |
| `403` | `Only the organisation owner can invite a new owner` |
| `409` | `This email is already a member of your team` |
| `409` | `This email is already registered to another organisation` |
| `429` | `Too many invites in the last 24 hours. Try again later.` (the 30-per-24h tenant cap) |
| `422` | Invalid email, name outside 1-255 chars, password under 8 chars, or an unknown role |

## PUT `/api/v1/users/{user_id}`

Partial update of a member. Omitted fields are unchanged. Owner or admin.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `full_name` | `string \| null` | No | 1-255 chars |
| `role` | `string \| null` | No | One of `owner`, `admin`, `agent`. Subject to the guard rails above |
| `is_active` | `boolean \| null` | No | `false` deactivates the member |

```bash
curl -X PUT https://api.callmissed.com/api/v1/users/u7654321-ba98-fedc-3210-fedcba987654 \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"role": "admin", "is_active": true}'
```

Returns the updated user object.

| Status | `detail` |
| --- | --- |
| `400` | `Cannot change the owner's role` |
| `400` | `You cannot change your own role. Ask another admin or the owner.` |
| `400` | `Cannot deactivate the organization owner` |
| `403` | `Only owners/admins can update users` |
| `403` | `Only the organisation owner can grant the owner role` |
| `404` | `User not found` in your tenant |
| `422` | Unknown role, or a name outside 1-255 chars |

## DELETE `/api/v1/users/{user_id}`

Returns `204` with an empty body. Owner or admin. Writes a `user.delete` audit event.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/users/u7654321-ba98-fedc-3210-fedcba987654 \
  -H "Authorization: Bearer <jwt_access_token>"
```

| Status | `detail` |
| --- | --- |
| `400` | `Cannot delete yourself` |
| `400` | `Cannot delete the organization owner` |
| `403` | `Only owners/admins can delete users` |
| `404` | `User not found` |

## PATCH `/api/v1/users/{user_id}/role`

Dedicated role change, **owner only**, always audited. Promoting another member to `owner` is an ownership transfer: the calling owner is demoted to `admin` in the same transaction, and both parties are emailed.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `role` | `string` | Yes | Exactly one of `owner`, `admin`, `agent` |

```bash
curl -X PATCH https://api.callmissed.com/api/v1/users/u7654321-ba98-fedc-3210-fedcba987654/role \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"role": "owner"}'
```

```json
{
  "id": "u7654321-ba98-fedc-3210-fedcba987654",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "email": "agent@acme.com",
  "full_name": "Sam Rivera",
  "phone_number": null,
  "role": "owner",
  "is_active": true,
  "created_at": "2026-08-04T12:10:00Z",
  "updated_at": "2026-08-04T12:22:41Z"
}
```

Sending the role the user already has is a no-op that returns the user unchanged. Writes a `role.change` audit event with `from`, `to`, and `ownership_transfer`.

| Status | `detail` |
| --- | --- |
| `403` | `Requires role: owner` |
| `404` | `User not found` |
| `422` | `role` is not `owner`, `admin`, or `agent` |
