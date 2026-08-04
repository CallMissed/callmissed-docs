---
title: "Authentication API"
description: "Full REST reference for account auth — register, login, OAuth, OTP, session management, two-factor (TOTP), and passkeys."
slug: "auth-api"
breadcrumb: "API Reference"
---

# Authentication API

Full REST reference for account auth — register, login, OAuth, OTP, session management, two-factor (TOTP), and passkeys.

> These endpoints power the dashboard's auth flows. They use **JWT** (access + refresh tokens), not `cm_` API keys. Access tokens last 24h; refresh tokens last 7d and rotate on use. Auth endpoints are rate-limited per IP (5-10 req/min).

## Credential class

| Credential | Works here? |
| --- | --- |
| No auth (public) | Yes, for `/register`, `/register/verify`, `/login`, `/google`, `/refresh`, `/logout`, `/otp/send`, `/otp/login`, `/reset-password`, `/2fa/verify`, `/2fa/recovery/*`, `/passkey/login/verify` |
| Dashboard JWT (`Authorization: Bearer <access_token>`) | Yes, for `/me`, `/sessions*`, `/2fa/status`, `/2fa/setup`, `/2fa/enable`, `/2fa/disable`, `/2fa/backup-codes/regenerate`, `/passkey/list`, `/passkey/register/*`, `/passkey/{id}` |
| API key (`Authorization: Bearer cm_...`) | **No.** Every authenticated route here resolves a JWT access token. A `cm_` key returns `401`. |

Base URL for every example: `https://api.callmissed.com`.

All errors share one shape:

```json
{ "detail": "Human-readable message" }
```

Validation failures (`422`) use FastAPI's shape:

```json
{ "detail": [ { "type": "string_too_short", "loc": ["body", "password"], "msg": "String should have at least 8 characters" } ] }
```

## Per-IP rate limits

Every path below is capped per client IP in a 60-second sliding window. Exceeding it returns `429` with a `Retry-After` header.

| Path | Limit / 60s |
| --- | --- |
| `POST /api/v1/auth/register` | 5 |
| `POST /api/v1/auth/register/verify` | 10 |
| `POST /api/v1/auth/login` | 10 |
| `POST /api/v1/auth/google` | 10 |
| `POST /api/v1/auth/refresh` | 10 |
| `POST /api/v1/auth/logout` | 10 |
| `POST /api/v1/auth/otp/send` | 5 |
| `POST /api/v1/auth/otp/login` | 10 |
| `POST /api/v1/auth/reset-password` | 5 |
| `POST /api/v1/auth/2fa/verify` | 10 |
| `POST /api/v1/auth/2fa/recovery/request` | 5 |
| `POST /api/v1/auth/2fa/recovery/confirm` | 5 |
| `POST /api/v1/auth/passkey/login/verify` | 10 |
| Everything else | 200 |

## The `LoginResponse` contract

`/login`, `/google`, `/otp/login`, `/register/verify`, `/reset-password`, `/2fa/verify`, and `/passkey/login/verify` all return the same object. It is a union: when the account has a second factor enabled you get a **challenge** instead of tokens.

| Field | Type | Notes |
| --- | --- | --- |
| `access_token` | `string \| null` | Null when `two_factor_required` is true |
| `refresh_token` | `string \| null` | Null when `two_factor_required` is true |
| `token_type` | `string` | Always `"bearer"` |
| `two_factor_required` | `boolean` | Default `false` |
| `challenge_token` | `string \| null` | Feed to `/2fa/verify` or `/passkey/login/verify` |
| `email` | `string \| null` | Echoed for the 2FA screen |
| `methods` | `string[]` | Subset of `["totp", "passkey"]` |
| `passkey_options` | `object \| null` | `PublicKeyCredentialRequestOptions` for `navigator.credentials.get()`; present only when `"passkey"` is in `methods` |
| `is_new_user` | `boolean` | True only when a Google sign-in just created the account |

Branch on `two_factor_required` before you read `access_token`.

---

# Registration

## POST /api/v1/auth/register

Creates the account and emails a 6-digit verification code. **No tokens are returned here** - call `/register/verify` next.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `full_name` | `string` | Yes | 1-255 chars |
| `email` | `string` | Yes | Valid email |
| `password` | `string` | Yes | 8-72 chars |
| `phone_number` | `string` | No | Max 20 chars, default `""` |
| `accept_terms` | `boolean` | Yes (must be `true`) | Default `false`; `false` returns `422` |
| `cf_turnstile_token` | `string \| null` | Yes in production | Max 2048 chars; Cloudflare Turnstile token from the signup widget |

```bash
curl https://api.callmissed.com/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "full_name": "Priya Nair",
    "email": "priya@acme.com",
    "password": "correct-horse-battery",
    "phone_number": "+919876543210",
    "accept_terms": true,
    "cf_turnstile_token": "0.AbCdEf..."
  }'
```

```json
{
  "detail": "Verification code sent to your email.",
  "email": "priya@acme.com"
}
```

| Status | Cause |
| --- | --- |
| `401` | Captcha verification failed |
| `422` | `accept_terms` not true, or a field failed validation |
| `429` | More than 5 registrations per minute from this IP |

## POST /api/v1/auth/register/verify

Consumes the emailed code and returns real tokens. A brand-new account cannot have 2FA yet, so `two_factor_required` is always `false` here.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `email` | `string` | Yes | Valid email |
| `code` | `string` | Yes | Exactly 6 digits (`^[0-9]{6}$`) |

```bash
curl https://api.callmissed.com/api/v1/auth/register/verify \
  -H "Content-Type: application/json" \
  -d '{"email": "priya@acme.com", "code": "418302"}'
```

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "two_factor_required": false,
  "challenge_token": null,
  "email": null,
  "methods": [],
  "passkey_options": null,
  "is_new_user": false
}
```

---

# Sign-in

## POST /api/v1/auth/login

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `email` | `string` | Yes | Valid email |
| `password` | `string` | Yes | 1-72 chars |

```bash
curl https://api.callmissed.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "priya@acme.com", "password": "correct-horse-battery"}'
```

No second factor:

```json
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "eyJhbGciOi...",
  "token_type": "bearer",
  "two_factor_required": false,
  "methods": [],
  "is_new_user": false
}
```

Second factor required:

```json
{
  "access_token": null,
  "refresh_token": null,
  "token_type": "bearer",
  "two_factor_required": true,
  "challenge_token": "4f1c8a9e2b7d6053a1c4e8f7b2d9a06e",
  "email": "priya@acme.com",
  "methods": ["totp", "passkey"],
  "passkey_options": { "challenge": "...", "rpId": "callmissed.com", "allowCredentials": [] },
  "is_new_user": false
}
```

| Status | Cause |
| --- | --- |
| `401` | Wrong email or password |
| `403` | Account not verified or deactivated |
| `429` | More than 10 attempts per minute from this IP |

## POST /api/v1/auth/google

Exchanges a Google ID token for CallMissed tokens. Creates the account on first use.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `credential` | `string` | Yes | 10-4096 chars; the Google ID token |
| `accept_terms` | `boolean` | Only for a NEW account | Default `false` |
| `cf_turnstile_token` | `string \| null` | Yes for web clients | Max 2048 chars. Skipped when the token's audience is a registered native-mobile client id |

```bash
curl https://api.callmissed.com/api/v1/auth/google \
  -H "Content-Type: application/json" \
  -d '{"credential": "eyJhbGciOiJSUzI1NiIs...", "accept_terms": true, "cf_turnstile_token": "0.AbCdEf..."}'
```

Returns a `LoginResponse`. On a first-time sign-in `is_new_user` is `true`.

| Status | Cause |
| --- | --- |
| `401` | Invalid Google credential, or captcha verification failed |
| `429` | More than 10 attempts per minute from this IP |

## POST /api/v1/auth/otp/send

Emails a 6-digit one-time code. **Always returns 200 with the same body**, whether or not the account exists - this is deliberate anti-enumeration. Do not treat the response as proof that an account exists.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `email` | `string` | Yes | Valid email |
| `purpose` | `string` | No | One of `login`, `reset_password`, `register`. Default `login` |
| `cf_turnstile_token` | `string \| null` | Usually | Max 2048 chars. Skipped when this is a resend within 30 minutes of a prior verified request for the same email+purpose |

```bash
curl https://api.callmissed.com/api/v1/auth/otp/send \
  -H "Content-Type: application/json" \
  -d '{"email": "priya@acme.com", "purpose": "login", "cf_turnstile_token": "0.AbCdEf..."}'
```

```json
{ "detail": "If an account exists, a code has been sent." }
```

| Status | Cause |
| --- | --- |
| `401` | Captcha verification failed (only on the non-resend path) |
| `429` | More than 5 sends per minute from this IP |

## POST /api/v1/auth/otp/login

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `email` | `string` | Yes | Valid email |
| `code` | `string` | Yes | Exactly 6 digits |
| `purpose` | `string` | No | One of `login`, `reset_password`, `register`. Default `login` |

```bash
curl https://api.callmissed.com/api/v1/auth/otp/login \
  -H "Content-Type: application/json" \
  -d '{"email": "priya@acme.com", "code": "418302", "purpose": "login"}'
```

Returns a `LoginResponse`. `400`/`401` on an invalid or expired code; `429` above 10/min per IP.

## POST /api/v1/auth/reset-password

Sets a new password using an OTP obtained from `/otp/send` with `purpose: "reset_password"`, then signs the user in.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `email` | `string` | Yes | Valid email |
| `code` | `string` | Yes | Exactly 6 digits |
| `new_password` | `string` | Yes | 8-72 chars |
| `cf_turnstile_token` | `string \| null` | No | Max 2048 chars. Verified but **not** enforced on this route: a failed captcha is logged, not rejected, because the OTP is itself a strong factor |

```bash
curl https://api.callmissed.com/api/v1/auth/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "email": "priya@acme.com",
    "code": "418302",
    "new_password": "a-much-better-passphrase",
    "cf_turnstile_token": "0.AbCdEf..."
  }'
```

Returns a `LoginResponse`. `400`/`401` on a bad or expired code; `429` above 5/min per IP.

---

# Tokens and sessions

## POST /api/v1/auth/refresh

Rotates the refresh token. The presented token is blacklisted, so a refresh token is single-use.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `refresh_token` | `string` | Yes | 10-2048 chars |

```bash
curl https://api.callmissed.com/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}'
```

```json
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "eyJhbGciOi...",
  "token_type": "bearer"
}
```

| Status | `detail` | Cause |
| --- | --- | --- |
| `401` | `Invalid refresh token` | Token was not a refresh-typed token |
| `401` | `User not found` | The subject no longer exists or is inactive |
| `401` | `Session not found` | The `sid` claim points at a session row that is gone |
| `401` | `Session has been revoked` | Session revoked, **or** an already-rotated token was replayed. Replay is treated as a compromise: every session for that user is revoked |
| `429` | rate limit | More than 10/min per IP |

## POST /api/v1/auth/logout

Blacklists the refresh token and revokes its session row. Idempotent: a malformed or already-blacklisted token still returns `200`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `refresh_token` | `string` | Yes | 10-2048 chars |

```bash
curl https://api.callmissed.com/api/v1/auth/logout \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "eyJhbGciOi..."}'
```

```json
{ "detail": "Logged out" }
```

## GET /api/v1/auth/sessions

Lists the caller's own live (non-revoked) sessions, newest first.

```bash
curl https://api.callmissed.com/api/v1/auth/sessions \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
[
  {
    "id": "9a4b1c2d-3e4f-5061-7283-94a5b6c7d8e9",
    "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36",
    "ip": "203.0.113.42",
    "created_at": "2026-08-01T09:12:44Z",
    "is_current": true
  }
]
```

`is_current` is computed from the `sid` claim on the access token you presented.

## DELETE `/api/v1/auth/sessions/{session_id}`

Revokes one session. Scoped to the calling user, so another user's session id returns `404`.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/auth/sessions/9a4b1c2d-3e4f-5061-7283-94a5b6c7d8e9 \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{ "detail": "Session revoked" }
```

`404` when the session does not exist or is not yours.

## DELETE /api/v1/auth/sessions

Signs out every other device. The caller's own session stays alive.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/auth/sessions \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{ "detail": "Other sessions revoked", "revoked": 3 }
```

---

# Profile

## GET /api/v1/auth/me

```bash
curl https://api.callmissed.com/api/v1/auth/me \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "user": {
    "id": "u1234567-89ab-cdef-0123-456789abcdef",
    "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
    "email": "priya@acme.com",
    "full_name": "Priya Nair",
    "phone_number": "+919876543210",
    "role": "owner",
    "is_active": true
  },
  "tenant": {
    "id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
    "name": "Acme",
    "slug": "acme",
    "plan": "pro",
    "credit_balance": 4820.5,
    "is_active": true
  }
}
```

## PATCH /api/v1/auth/me

Self-service correction of your own name and phone number. Role, email, and active state are **not** editable here.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `full_name` | `string \| null` | No | 1-255 chars when present |
| `phone_number` | `string \| null` | No | Max 20 chars; an empty string clears it |

```bash
curl -X PATCH https://api.callmissed.com/api/v1/auth/me \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"full_name": "Priya R. Nair", "phone_number": "+919812345678"}'
```

Returns the same `MeResponse` shape as `GET /me`. Writes a `profile.update` audit event.

---

# Two-factor (TOTP)

Enrolment is three calls: `/2fa/setup` (stash a pending secret and render the QR), `/2fa/enable` (prove possession, receive backup codes), then `/2fa/verify` at every future login.

## GET /api/v1/auth/2fa/status

```bash
curl https://api.callmissed.com/api/v1/auth/2fa/status \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "enabled": true,
  "enabled_at": "2026-07-14T08:31:02Z",
  "backup_codes_remaining": 8,
  "has_password": true
}
```

`has_password` is `false` for passwordless (Google-only) accounts, which changes what `/2fa/disable` requires.

## POST /api/v1/auth/2fa/setup

No request body. Idempotent for about 15 minutes, so reloading the setup page does not invalidate a QR the user already scanned.

```bash
curl -X POST https://api.callmissed.com/api/v1/auth/2fa/setup \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "secret": "JBSWY3DPEHPK3PXP",
  "provisioning_uri": "otpauth://totp/CallMissed:priya@acme.com?secret=JBSWY3DPEHPK3PXP&issuer=CallMissed",
  "account_name": "priya@acme.com",
  "issuer": "CallMissed"
}
```

`400` with `2FA already enabled. Disable it first to re-enroll.` when a factor is already active.

## POST /api/v1/auth/2fa/enable

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `code` | `string` | Yes | 6-7 chars matching `^[0-9 ]{6,7}$` (a space is tolerated) |

```bash
curl -X POST https://api.callmissed.com/api/v1/auth/2fa/enable \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"code": "492013"}'
```

```json
{
  "backup_codes": [
    "3f9a-21c7", "8b0d-4e15", "c27f-9a63", "51ea-70bd", "aa39-16f2",
    "6d84-cb07", "17c5-e93a", "b402-5f8e", "9e61-30da", "24bf-a7c9"
  ],
  "enabled_at": "2026-08-04T10:02:11Z"
}
```

Backup codes are returned **exactly once**. No later endpoint exposes them in plaintext.

| Status | `detail` |
| --- | --- |
| `400` | `2FA already enabled.` |
| `400` | `Call /2fa/setup first.` |
| `400` | `Setup expired. Start over from /2fa/setup.` |
| `400` | `Invalid code. Check your authenticator app's clock.` |

## POST /api/v1/auth/2fa/disable

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `password` | `string \| null` | Required if the account has a password | 1-72 chars |
| `code` | `string` | Yes | 6-15 chars. A live TOTP code **or** a backup code |

```bash
curl -X POST https://api.callmissed.com/api/v1/auth/2fa/disable \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"password": "correct-horse-battery", "code": "492013"}'
```

```json
{ "detail": "2FA disabled" }
```

Returns `{"detail": "2FA was not enabled"}` (still `200`) when there was nothing to disable. `400` on `Incorrect password.` or `Invalid 2FA code.`.

## POST /api/v1/auth/2fa/backup-codes/regenerate

Burns the old set and issues ten fresh codes. Requires a live TOTP code so a stolen session cannot silently rotate them.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `code` | `string` | Yes | 6-7 chars matching `^[0-9 ]{6,7}$` |

```bash
curl -X POST https://api.callmissed.com/api/v1/auth/2fa/backup-codes/regenerate \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"code": "770125"}'
```

Same body shape as `/2fa/enable`. `400` on `2FA is not enabled.` or `Invalid code.`.

## POST /api/v1/auth/2fa/verify

Second leg of login. Send the `challenge_token` from the `LoginResponse`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `challenge_token` | `string` | Yes | 10-256 chars |
| `code` | `string` | Yes | 6-15 chars |
| `is_backup_code` | `boolean` | No | Default `false`. **Advisory only** - the server dispatches on the code's shape, not this flag |

```bash
curl -X POST https://api.callmissed.com/api/v1/auth/2fa/verify \
  -H "Content-Type: application/json" \
  -d '{"challenge_token": "4f1c8a9e2b7d6053a1c4e8f7b2d9a06e", "code": "492013"}'
```

```json
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "eyJhbGciOi...",
  "token_type": "bearer",
  "two_factor_required": false,
  "methods": [],
  "is_new_user": false
}
```

| Status | Cause |
| --- | --- |
| `400` | Wrong code. The message reports how many attempts remain |
| `401` | Challenge invalid, expired, already used, or the account is unavailable |
| `429` | Per-challenge attempt cap exhausted, or more than 10 requests/min from this IP. Start a new sign-in |

## POST /api/v1/auth/2fa/recovery/request

Last resort for a user who lost both the authenticator and the backup codes. Passwordless (Google-only) accounts cannot recover this way. **Always returns 200 with an identical body**, regardless of outcome.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `email` | `string` | Yes | Valid email |
| `password` | `string \| null` | Yes in practice | 1-72 chars. A wrong or missing password still returns the generic 200 |
| `cf_turnstile_token` | `string \| null` | Yes in production | Max 2048 chars |

```bash
curl -X POST https://api.callmissed.com/api/v1/auth/2fa/recovery/request \
  -H "Content-Type: application/json" \
  -d '{"email": "priya@acme.com", "password": "correct-horse-battery", "cf_turnstile_token": "0.AbCdEf..."}'
```

```json
{ "detail": "If the account exists and has a password set, a recovery code was emailed." }
```

`401` only on captcha failure; `429` above 5/min per IP.

## POST /api/v1/auth/2fa/recovery/confirm

Disables 2FA and wipes the backup codes. Every failure path returns one identical `400`, so this cannot be used as a brute-force oracle.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `email` | `string` | Yes | Valid email |
| `code` | `string` | Yes | Exactly 6 digits |
| `password` | `string \| null` | Yes in practice | 1-72 chars |

```bash
curl -X POST https://api.callmissed.com/api/v1/auth/2fa/recovery/confirm \
  -H "Content-Type: application/json" \
  -d '{"email": "priya@acme.com", "code": "551907", "password": "correct-horse-battery"}'
```

```json
{ "detail": "2FA disabled. Sign in again to continue." }
```

`400` with `Unable to complete recovery.` on every negative branch; `429` above 5/min per IP.

---

# Passkeys (WebAuthn / FIDO2)

## GET /api/v1/auth/passkey/list

```bash
curl https://api.callmissed.com/api/v1/auth/passkey/list \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
[
  {
    "id": "7d3e9f10-2ab4-4c5d-8e6f-0a1b2c3d4e5f",
    "name": "MacBook Touch ID",
    "transports": "internal,hybrid",
    "created_at": "2026-07-20T14:05:00Z",
    "last_used_at": "2026-08-03T08:41:12Z"
  }
]
```

## POST /api/v1/auth/passkey/register/options

No request body. Returns the `PublicKeyCredentialCreationOptions` for `navigator.credentials.create()` plus a one-shot `registration_token`. Calling this consumes any in-flight challenge for the user.

```bash
curl -X POST https://api.callmissed.com/api/v1/auth/passkey/register/options \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "options": {
    "rp": { "id": "callmissed.com", "name": "CallMissed" },
    "user": { "id": "...", "name": "priya@acme.com", "displayName": "Priya Nair" },
    "challenge": "...",
    "pubKeyCredParams": [{ "type": "public-key", "alg": -7 }],
    "excludeCredentials": []
  },
  "registration_token": "b19d4f77e0a2c85316fa..."
}
```

## POST /api/v1/auth/passkey/register/verify

`registration_token` is a **query parameter**, not a body field. Sending it in the body returns `422`.

| Parameter | In | Type | Required | Constraints |
| --- | --- | --- | --- | --- |
| `registration_token` | query | `string` | Yes | 10-256 chars, from `/register/options` |
| `credential` | body | `object` | Yes | The `PublicKeyCredential` JSON from `navigator.credentials.create()` |
| `name` | body | `string` | No | 1-120 chars, default `"Passkey"` |

```bash
curl -X POST "https://api.callmissed.com/api/v1/auth/passkey/register/verify?registration_token=b19d4f77e0a2c85316fa..." \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "credential": { "id": "AaBb...", "rawId": "AaBb...", "type": "public-key", "response": { "clientDataJSON": "...", "attestationObject": "..." } },
    "name": "MacBook Touch ID"
  }'
```

```json
{
  "id": "7d3e9f10-2ab4-4c5d-8e6f-0a1b2c3d4e5f",
  "name": "MacBook Touch ID",
  "transports": "internal,hybrid",
  "created_at": "2026-08-04T10:15:00Z",
  "last_used_at": null
}
```

| Status | `detail` |
| --- | --- |
| `400` | `Registration session expired. Start over.` |
| `400` | `Invalid registration session.` |
| `400` | `Passkey registration failed. Try again.` |
| `400` | `This passkey is already registered.` |

## PATCH `/api/v1/auth/passkey/{passkey_id}`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1-120 chars |

```bash
curl -X PATCH https://api.callmissed.com/api/v1/auth/passkey/7d3e9f10-2ab4-4c5d-8e6f-0a1b2c3d4e5f \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Work laptop"}'
```

Returns the updated passkey summary. `404` when the passkey is not yours.

## DELETE `/api/v1/auth/passkey/{passkey_id}`

```bash
curl -X DELETE https://api.callmissed.com/api/v1/auth/passkey/7d3e9f10-2ab4-4c5d-8e6f-0a1b2c3d4e5f \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{ "detail": "Passkey removed" }
```

`404` when the passkey is not yours.

## POST /api/v1/auth/passkey/login/verify

Passkey equivalent of `/2fa/verify`. Sign the challenge from the `LoginResponse` with `navigator.credentials.get()` and post the assertion.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `challenge_token` | `string` | Yes | 10-256 chars |
| `credential` | `object` | Yes | The `PublicKeyCredential` JSON from `navigator.credentials.get()` |

```bash
curl -X POST https://api.callmissed.com/api/v1/auth/passkey/login/verify \
  -H "Content-Type: application/json" \
  -d '{
    "challenge_token": "4f1c8a9e2b7d6053a1c4e8f7b2d9a06e",
    "credential": { "id": "AaBb...", "rawId": "AaBb...", "type": "public-key", "response": { "clientDataJSON": "...", "authenticatorData": "...", "signature": "...", "userHandle": "..." } }
  }'
```

Returns a `LoginResponse` carrying real tokens.

| Status | Cause |
| --- | --- |
| `400` | No passkey challenge on this session |
| `401` | Challenge invalid, expired, or already used |
| `429` | Attempt cap exhausted, or more than 10 requests/min from this IP |
