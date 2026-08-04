---
title: "Integration Settings API"
description: "Manage per-tenant channel credentials — WhatsApp (Meta), Twilio, and bring-your-own Sarvam key — and verify connectivity."
slug: "settings-api"
breadcrumb: "API Reference"
---

# Integration Settings API

Manage per-tenant channel credentials — WhatsApp (Meta), Twilio, and bring-your-own Sarvam key — and verify connectivity.

> These endpoints store the third-party credentials a tenant uses for channels and BYOK providers. Each provider has a `verify` action that does a live connectivity check. Credentials are write-only: reads return masked values.

## Credential class: dashboard JWT

Every endpoint here resolves a **dashboard access token**. A `cm_` API key is **rejected with `401`**.

```
Authorization: Bearer <jwt_access_token>
```

| Method | Role required |
| --- | --- |
| `GET` | Any signed-in member of the tenant |
| `PUT`, `POST .../verify` | **Owner or admin.** Others get `403 Only owners/admins can update settings` |

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`; validation failures are `422`.

## How masking works

A `GET` never returns a stored secret in full. Values are masked as `first4 + "••••••••" + last4`, or `"••••••••"` when the value is 8 characters or shorter, or `null` when nothing is stored. A `PUT` replaces the whole credential set for that provider: send **every** required field, not a partial patch.

## How verify works

The `verify` endpoints call the provider with the credentials **already stored** for your tenant. They take no request body, so save with `PUT` first. They return `200` in every outcome, with the result in the body rather than the status code:

```json
{ "status": "connected", "error": null }
```

| `status` | Meaning |
| --- | --- |
| `connected` | The provider accepted the stored credentials |
| `not_configured` | Nothing is stored yet for this provider |
| `error` | The provider rejected the call, timed out, or the check failed. `error` carries a short reason |

---

# WhatsApp (Meta Cloud API)

## GET /api/v1/settings/whatsapp

```bash
curl https://api.callmissed.com/api/v1/settings/whatsapp \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "phone_number_id": "109876543210987",
  "access_token": "EAAG••••••••ZDZD",
  "verify_token": "my-v••••••••oken",
  "app_secret": "3f9a••••••••7c21",
  "configured": true,
  "webhook_url": "https://api.callmissed.com/api/v1/webhooks/whatsapp"
}
```

`phone_number_id` is returned in full (it is an identifier, not a secret) and is `""` when unset. `configured` is `true` only when both `phone_number_id` and `access_token` are stored. Point your Meta app's webhook at `webhook_url`.

## PUT /api/v1/settings/whatsapp

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `phone_number_id` | `string` | Yes | 1-100 chars |
| `access_token` | `string` | Yes | 1-1024 chars |
| `verify_token` | `string` | Yes | 1-255 chars. The token you also enter in the Meta webhook config |
| `app_secret` | `string \| null` | No | Max 255 chars. Stored as `""` when omitted |

```bash
curl -X PUT https://api.callmissed.com/api/v1/settings/whatsapp \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "109876543210987",
    "access_token": "EAAGm0PX4ZCpsBO...",
    "verify_token": "my-verify-token",
    "app_secret": "3f9a1c4e6b8d0a137c21"
  }'
```

```json
{ "detail": "WhatsApp settings saved" }
```

`403` for non-admins; `422` when a field is missing or exceeds its length bound.

## POST /api/v1/settings/whatsapp/verify

No request body. Performs a live read of the stored `phone_number_id` against the Meta Graph API using the stored access token.

```bash
curl -X POST https://api.callmissed.com/api/v1/settings/whatsapp/verify \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{ "status": "connected", "error": null }
```

Other outcomes: `{"status": "not_configured", "error": "WhatsApp credentials not configured"}`, `{"status": "error", "error": "WhatsApp API returned 401"}`, `{"status": "error", "error": "WhatsApp API timeout — try again"}`. `403` for non-admins.

---

# Twilio

## GET /api/v1/settings/twilio

```bash
curl https://api.callmissed.com/api/v1/settings/twilio \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "account_sid": "ACb1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6",
  "auth_token": "9a4b••••••••d8e9",
  "phone_number": "+14155550123",
  "configured": true,
  "voice_webhook_url": "https://api.callmissed.com/api/v1/webhooks/twilio/voice"
}
```

`account_sid` and `phone_number` are returned in full (`""` when unset); only `auth_token` is masked. Set the Twilio number's Voice webhook to `voice_webhook_url`.

## PUT /api/v1/settings/twilio

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `account_sid` | `string` | Yes | 1-100 chars |
| `auth_token` | `string` | Yes | 1-255 chars |
| `phone_number` | `string` | Yes | 1-20 chars, E.164 |

```bash
curl -X PUT https://api.callmissed.com/api/v1/settings/twilio \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "account_sid": "ACb1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6",
    "auth_token": "9a4b1c2d3e4f5061728394a5b6c7d8e9",
    "phone_number": "+14155550123"
  }'
```

```json
{ "detail": "Twilio settings saved" }
```

`403` for non-admins; `422` on validation failure.

## POST /api/v1/settings/twilio/verify

No request body. Fetches the stored account from the Twilio REST API using the stored SID and auth token.

```bash
curl -X POST https://api.callmissed.com/api/v1/settings/twilio/verify \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{ "status": "connected", "error": null }
```

Other outcomes: `{"status": "not_configured", "error": "Twilio credentials not configured"}`, `{"status": "error", "error": "Twilio API returned 401"}`, `{"status": "error", "error": "Twilio API timeout — try again"}`. `403` for non-admins.

---

# Sarvam AI (bring your own key)

## GET /api/v1/settings/sarvam

```bash
curl https://api.callmissed.com/api/v1/settings/sarvam \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "api_key": "sk_a••••••••9f21",
  "configured": true
}
```

`api_key` is `null` when nothing is stored.

## PUT /api/v1/settings/sarvam

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `api_key` | `string` | Yes | 1-255 chars |

```bash
curl -X PUT https://api.callmissed.com/api/v1/settings/sarvam \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"api_key": "sk_a1b2c3d4e5f60718293a4b5c6d7e8f9f21"}'
```

```json
{ "detail": "Sarvam AI settings saved" }
```

`403` for non-admins; `422` on validation failure.

## POST /api/v1/settings/sarvam/verify

No request body. Probes Sarvam with the stored key. A `400`, `405`, or `422` from Sarvam still counts as `connected` (the key authenticated; only the request shape was wrong), while `401` or `403` reports `Invalid API key`.

```bash
curl -X POST https://api.callmissed.com/api/v1/settings/sarvam/verify \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{ "status": "connected", "error": null }
```

Other outcomes: `{"status": "not_configured", "error": "Sarvam AI key not configured"}`, `{"status": "error", "error": "Invalid API key"}`, `{"status": "error", "error": "Sarvam API timeout — try again"}`. `403` for non-admins.
