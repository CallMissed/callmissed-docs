---
title: "Facebook & Instagram API"
description: "Publish posts, moderate comments and reply to DMs on a connected Facebook Page and Instagram professional account — one API, both channels, no account id in the path."
slug: "social-api"
breadcrumb: "Facebook & Instagram"
---

# Facebook & Instagram API

Publish posts, moderate comments and reply to DMs on a connected Facebook Page and Instagram professional account — one API, both channels, no account id in the path.

The Social API is the programmatic surface for a connected **Facebook Page** and
**Instagram professional account**. One API key publishes posts, reads and
moderates comments, and replies to direct messages across both channels.

You never put an account id in the path. Connect a Page or an Instagram account
once, then call `POST /api/v1/facebook/posts` — the account is resolved from your
workspace server-side. This page covers the base path, authentication, that
resolution model, and the error shapes every other Social page depends on.

**Base URL:** `https://api.callmissed.com`
**Base path:** `/api/v1/facebook` and `/api/v1/instagram`

| Area | Page |
|---|---|
| Publish posts, photos, reels, carousels, stories | [Publishing](/docs/social-publish) |
| Read, reply to, hide and delete comments | [Comments](/docs/social-comments) |
| Read inbox threads, reply to DMs, messaging analytics | [Messaging](/docs/social-messaging) |
| Generate a caption, hashtags and an image from a topic | [Post Studio](/docs/social-posts) |

## Connecting an account

Connecting a Facebook Page or an Instagram professional account is a one-time
setup done from your dashboard — you sign in with Facebook or Instagram and grant
CallMissed permission to act on the asset. The **first** account you connect on a
channel becomes that channel's **default**, so from the moment it is connected
every endpoint below works without naming it.

An Instagram account must be a **Business or Creator** account with content
publishing granted. A personal account cannot publish through the API.

### Completing the connection yourself

If you run Meta's login dialog in your own front end instead of the dashboard,
post the authorization code it returns to CallMissed. The code must be issued for
**CallMissed's** Meta app — a code from your own Meta app is rejected with `400` —
and it is single-use and short-lived, so post it straight away. Needs
`whatsapp:write`.

`POST /api/v1/facebook/onboarding/exchange`

| Field | Type | Required | Notes |
|---|---|---|---|
| `code` | string, 1 to 2048 chars | Yes | The authorization code from Facebook Login for Business |
| `page_id` | string, 1 to 64 chars | No | Connect only this Page. Omit to connect **every** Page the login granted |
| `bot_id` | UUID | No | The agent that answers these Pages' DMs |

The response lists one entry per connected Page, plus any granted Page that could
not be connected and why:

```json
{
  "pages": [
    {
      "page": {
        "id": "3f9a2c71-5d8e-4b16-9f03-7c1a2e5b84d0",
        "page_id": "102938475610293",
        "name": "Kalyani Motors",
        "bot_id": null,
        "connected_user_id": "7788990011223344",
        "is_active": true,
        "is_default": true,
        "created_at": "2026-08-20T11:04:18.912004+00:00"
      },
      "fully_provisioned": true,
      "onboarding_error": null,
      "instagram_business_account_id": "17841400000000000"
    }
  ],
  "skipped": []
}
```

`POST /api/v1/instagram/onboarding/exchange`

| Field | Type | Required | Notes |
|---|---|---|---|
| `code` | string, 1 to 2048 chars | Yes | The authorization code from Business Login for Instagram (strip any trailing `#_`) |
| `ig_user_id` | string, 1 to 64 chars | No | Assert which account you expect; the login's own account is always the one connected |
| `redirect_uri` | string, max 2048 | No | Must match the `redirect_uri` used when the login was started, if one was used |
| `bot_id` | UUID | No | The agent that answers this account's DMs |

The response is `{ "account": { … }, "fully_provisioned": true, "onboarding_error": null }`,
with `account` in the [Instagram account shape](#list-connected-accounts).

`fully_provisioned: false` means the account is saved but inbound DMs will not
arrive yet; `onboarding_error` gives a short reason. Re-run the connection to
retry — it is idempotent for an account already in your workspace.

| HTTP | Meaning |
|---|---|
| `400` | Meta rejected the code (expired, already used, or issued for another app) |
| `403` | The login does not grant access to the Page or account you named, or grants none |
| `404` | The `bot_id` is not one of your agents |
| `409` | The Page or account is already connected to another workspace |

## Authentication

Every endpoint accepts either a `cm_` API key or a dashboard session:

```
Authorization: Bearer cm_your_api_key
```

API keys are checked against three scopes. These are the **same scope names** used
across the messaging channels — a Facebook or Instagram call is authorized by the
`whatsapp:*` scope family, not a separate `facebook:*` or `instagram:*` scope.

| Scope | Grants |
|---|---|
| `whatsapp:read` | List connected accounts, list comments and their replies, read inbox threads and messages, read messaging analytics, draft an agent system prompt |
| `whatsapp:write` | Complete a connection (`/onboarding/exchange`), change which account is the default, attach or clear an account's agent |
| `whatsapp:send` | Publish posts, reply to and hide/delete comments, send DMs, check and publish Instagram containers, read the Instagram publishing quota |

A key without the scope gets `403`. Add scopes under the key's **Permissions**
section in your dashboard.

## How an account is chosen

Every call is scoped to your workspace, and to exactly one connected account. You
do not have to say which one:

1. **One connected account on that channel** — it is used. Nothing to pass.
2. **Several, one marked default** — the default is used.
3. **Several, no default** — the call is refused with `409` and a message asking
   you to name one. Nothing is published or modified.

To override the choice on a single call, pass the optional **`?account=`** query
parameter. It accepts **either** identifier, so the id you already have works
without translation:

- the account's CallMissed id (a UUID), or
- Meta's own id — `page_id` for a Page, `ig_user_id` for an Instagram account.

```bash
# Uses the workspace's only (or default) Page
curl -X POST https://api.callmissed.com/api/v1/facebook/posts \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{ "message": "Open till 9 tonight." }'

# Publishes as one specific Page — Meta's page_id
curl -X POST "https://api.callmissed.com/api/v1/facebook/posts?account=102938475610293" \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{ "message": "Open till 9 tonight." }'
```

`?account=` is accepted on every publishing and comment endpoint. Messaging is
the exception: a DM send names its account in the **body** instead, because the
value you need is already on the thread you are replying to — see
[Messaging](/docs/social-messaging).

### List connected accounts

`GET /api/v1/facebook/pages` · `GET /api/v1/instagram/accounts` — needs
`whatsapp:read`.

Every account your workspace has connected on that channel, newest first. Access
tokens are never returned. `is_default` tells you which account a call with no
`?account=` will act as.

```json
[
  {
    "id": "3f9a2c71-5d8e-4b16-9f03-7c1a2e5b84d0",
    "page_id": "102938475610293",
    "name": "Kalyani Motors",
    "bot_id": null,
    "connected_user_id": "7788990011223344",
    "is_active": true,
    "is_default": true,
    "created_at": "2026-08-20T11:04:18.912004+00:00"
  }
]
```

The Instagram shape carries `ig_user_id`, `username` and `token_expires_at`
instead of `page_id`, `name` and `connected_user_id`. An Instagram connection
carries an expiry — `token_expires_at` is that deadline. Watch it: once it passes,
calls on that account start returning `401` and the account has to be reconnected
from your dashboard. Reconnecting is the same one-time flow and keeps the
account's id, so nothing in your integration changes.

### Set the default account

`PATCH /api/v1/facebook/pages/{page_ref}/default`
`PATCH /api/v1/instagram/accounts/{account_ref}/default` — needs `whatsapp:write`.

Nominates the account that answers every call that names none. `page_ref` /
`account_ref` is the CallMissed id **or** Meta's id, the same as `?account=`.
Setting a default clears it from the previous holder, so a workspace has at most
one per channel. Returns the updated account in the shape above.

This is what turns a multi-account workspace from "every call needs `?account=`"
back into "most calls need nothing".

### Turn AI replies on or off for an account

`PATCH /api/v1/facebook/pages/{page_ref}/bot`
`PATCH /api/v1/instagram/accounts/{account_ref}/bot` — needs `whatsapp:write`.

Attaches the agent that auto-answers that account's DMs, or clears it. `page_ref`
/ `account_ref` is the CallMissed id **or** Meta's id, the same as `?account=`.
The body is the agent to attach; send `null` to turn AI off:

```json
{ "bot_id": "b7c1e2a4-9d3f-4a80-8e21-5f6c0a1b2d3e" }
```

Once an agent is attached, incoming DMs on that account are answered from the
agent's configured behaviour. Sending `{ "bot_id": null }` stops that — the
account keeps receiving DMs, they just aren't answered automatically. Returns the
updated account in the shape above, with `bot_id` reflecting the change.

The account must be connected: attaching an agent to a disconnected account
returns `409` and asks you to reconnect it first.

### Draft an agent system prompt

`POST /api/v1/facebook/ai/build_system_prompt`
`POST /api/v1/instagram/ai/build_system_prompt` — needs `whatsapp:read`.

Turns a plain-language description of your business into a system prompt shaped
for DMs and comment replies (short, plain text). It writes nothing: review the
returned markdown and save it onto your agent yourself. Not metered.

| Field | Type | Required | Notes |
|---|---|---|---|
| `intent` | string, 10 to 8000 chars | Yes | What the business does and what the agent should handle, or an existing prompt to improve |
| `language` | string, 2 to 8 chars | No | Default `en` |
| `tone` | string, max 40 | No | e.g. `friendly`, `playful` |
| `existing_prompt` | string, max 8000 | No | When set, this prompt is improved instead of starting fresh |

```json
{ "system_prompt": "## Role\nYou answer DMs for Kalyani Motors..." }
```

If the model cannot produce a usable prompt the call returns `422`; rephrase the
description and retry.

### Legacy: the account id in the path

The older form, with the account's **CallMissed id** (a UUID — not Meta's id) in
the path, still works and is not being removed. Same bodies, same responses, same
scopes; these paths do not take `?account=`.

| Facebook | Instagram |
|---|---|
| `POST /api/v1/facebook/{page_uuid}/posts` | `POST /api/v1/instagram/{ig_account_uuid}/posts` |
| `POST /api/v1/facebook/{page_uuid}/photos` | `GET /api/v1/instagram/{ig_account_uuid}/posts/{container_id}` |
| `POST /api/v1/facebook/{page_uuid}/reels` | `POST /api/v1/instagram/{ig_account_uuid}/posts/{container_id}/publish` |
| | `GET /api/v1/instagram/{ig_account_uuid}/publishing_limit` |
| `GET /api/v1/facebook/{page_uuid}/posts/{post_id}/comments` | `GET /api/v1/instagram/{ig_account_uuid}/media/{media_id}/comments` |
| `GET /api/v1/facebook/{page_uuid}/comments/{comment_id}/replies` | `GET /api/v1/instagram/{ig_account_uuid}/comments/{comment_id}/replies` |
| `POST /api/v1/facebook/{page_uuid}/comments/{comment_id}/reply` | `POST /api/v1/instagram/{ig_account_uuid}/comments/{comment_id}/reply` |
| `POST /api/v1/facebook/{page_uuid}/comments/{comment_id}/hide` | `POST /api/v1/instagram/{ig_account_uuid}/comments/{comment_id}/hide` |
| `DELETE /api/v1/facebook/{page_uuid}/comments/{comment_id}` | `DELETE /api/v1/instagram/{ig_account_uuid}/comments/{comment_id}` |

New integrations should use the shorter paths on this site: they need no id at
all in the common case, and one query parameter in the multi-account case.

## Errors

Errors are returned as a JSON body with a `detail` string. Upstream Meta error
text and numeric codes are **never** echoed back — a failure is mapped to a clean
HTTP status with a message safe to show an operator.

```json
{ "detail": "This Page's connection has expired. Reconnect it to keep posting." }
```

These are the statuses you will actually hit across the Social API:

| HTTP | Meaning |
|---|---|
| `400` | The request was rejected before any Meta call — a closed messaging window, a missing access token, or a body that failed local validation. |
| `401` | The connected account's access token has expired or been revoked. Reconnect the Page or account. |
| `403` | The account lacks the permission for this action (publishing, engagement/moderation), or the key lacks the scope. |
| `404` | No connected account answers this call — your workspace has none on that channel, or the `?account=` value does not exist or is not yours (identical response for both). |
| `409` | Either the account is **ambiguous** (several connected, none marked default — name one with `?account=` or set a default), or the resolved account is **disconnected** / its token is unavailable, or the upstream state is ambiguous (e.g. a possible duplicate). |
| `422` | The body failed validation — a bad post type, an unreachable media URL, a caption over the limit, or an out-of-range schedule. |
| `429` | Meta is rate-limiting this account. Wait a few minutes and retry. |
| `503` | The upstream call failed, or the channel is temporarily unavailable. When the reason is an ambiguous upstream timeout, the action **may or may not** have completed: check the account before retrying rather than assuming it failed. (An ambiguous Instagram **publish** reports `409` instead.) |

Two details worth designing around:

- **A missing account and someone else's account return the same `404`**, with the
  same message. The API is not an id oracle, so you cannot use it to probe for
  accounts you do not own.
- **`409` is the one to handle in code**, because it is the only status that a
  correct, well-formed request can hit purely because a second account got
  connected. Read `detail`: an ambiguity asks you to name an account, a
  disconnection asks you to reconnect.

## Billing

Organic publishing, comment moderation and messaging on Facebook and Instagram
are **not metered** — Meta does not charge for them, and neither do we. No credits
are reserved or deducted on these endpoints. (The [Post Studio](/docs/social-posts),
which generates an image and a caption with AI, is metered for that generation —
see its page.)
