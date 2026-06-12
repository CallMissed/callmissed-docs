---
title: "Team & Tenants"
description: "Manage your organization (tenant): team members and roles, preferences, data retention, usage, and onboarding state."
slug: "team"
breadcrumb: "API Reference"
---

# Team & Tenants

Manage your organization (tenant): team members and roles, preferences, data retention, usage, and onboarding state.

## Overview

A **tenant** is your organization. Every user, bot, key, and conversation belongs to exactly one tenant, and all data is isolated per tenant. Roles are **owner**, **admin**, and **agent**:

| Role | Can do |
| --- | --- |
| `owner` | Everything, plus transfer ownership and deactivate the org. One per tenant. |
| `admin` | Invite/update/remove members (not the owner), manage bots, keys, billing. |
| `agent` | Operational access — handle conversations; no team or billing management. |

> Team and tenant endpoints authenticate with your **dashboard session (JWT)**, not a `cm_` API key. Inviting members, changing roles, and deactivating the org are not exposed to API keys.

## Invite a member

Omit `password` and the invitee sets their own via the reset-password OTP flow:

```bash
curl https://api.callmissed.com/api/v1/users \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"email": "agent@acme.com", "full_name": "Sam Rivera", "role": "agent"}'
```

```json
{
  "id": "u1234567-89ab-cdef-0123-456789abcdef",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "email": "agent@acme.com",
  "full_name": "Sam Rivera",
  "phone_number": null,
  "role": "agent",
  "is_active": true,
  "created_at": "2026-06-06T12:10:00Z",
  "updated_at": "2026-06-06T12:10:00Z"
}
```

Only the owner can invite or promote someone **to** `owner`; admins cannot. Ownership transfer goes through the owner-only `PATCH /api/v1/users/:id/role`.