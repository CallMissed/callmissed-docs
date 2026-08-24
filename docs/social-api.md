---
title: "Facebook & Instagram API"
description: "Publish posts, moderate comments and reply to DMs on a connected Facebook Page and Instagram professional account — one API, both channels."
slug: "social-api"
breadcrumb: "Facebook & Instagram"
---

# Facebook & Instagram API

Publish posts, moderate comments and reply to DMs on a connected Facebook Page and Instagram professional account — one API, both channels.

The Social API is the programmatic surface for a connected **Facebook Page** and
**Instagram professional account**. One API key publishes posts, reads and
moderates comments, and replies to direct messages across both channels. This
page covers the base path, authentication, the connected-asset model, and the
error shapes every other Social page depends on.

**Base URL:** `https://api.callmissed.com`
**Base path:** `/api/v1/facebook` and `/api/v1/instagram`

| Area | Page |
|---|---|
| Publish posts, photos, reels, carousels, stories | [Publishing](/docs/social-publish) |
| Read, reply to, hide and delete comments | [Comments](/docs/social-comments) |
| Read inbox threads and reply to DMs | [Messaging](/docs/social-messaging) |
| Generate a caption, hashtags and an image from a topic | [Post Studio](/docs/social-posts) |

## Connecting an account

Connecting a Facebook Page or an Instagram professional account is a one-time
setup done from your dashboard — you sign in with Facebook or Instagram and grant
CallMissed permission to act on the asset. The API then addresses each connected
asset by its CallMissed **UUID**, which you get back when you connect it and which
appears in the dashboard. Every endpoint below takes that UUID in the path
(`{page_uuid}` for a Page, `{ig_account_uuid}` for an Instagram account).

An Instagram account must be a **Business or Creator** account with content
publishing granted. A personal account cannot publish through the API.

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
| `whatsapp:read` | List comments and their replies, read inbox threads and messages |
| `whatsapp:write` | (reserved for account management done from the dashboard) |
| `whatsapp:send` | Publish posts, reply to and hide/delete comments, send DMs |

A key without the scope gets `403`. Add scopes under the key's **Permissions**
section in your dashboard.

## The connected-asset model

Every call is scoped to your workspace and to one connected asset:

- The **UUID in the path** identifies which connected Page or Instagram account
  the call acts as. It must belong to your workspace.
- A UUID that does not exist **and** one that belongs to another workspace return
  the **same `404`** — the API is not an id oracle, so you cannot use it to probe
  for assets you do not own.
- A connected asset that has been **disconnected**, or whose stored access token
  can no longer be used, returns `409`. Reconnect it from the dashboard.

## Error shape

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
| `401` | The connected asset's access token has expired or been revoked. Reconnect the Page or account. |
| `403` | The asset lacks the permission for this action (publishing, engagement/moderation), or the key lacks the scope. |
| `404` | The asset UUID does not exist or is not in your workspace (identical response for both). |
| `409` | The asset is disconnected, its token is unavailable, or an ambiguous upstream state (e.g. a possible duplicate) — check before retrying. |
| `422` | The body failed validation — a bad post type, an unreachable media URL, a caption over the limit, or an out-of-range schedule. |
| `429` | Meta is rate-limiting this asset. Wait a few minutes and retry. |
| `502` | The upstream call failed. |
| `503` | The channel is temporarily unavailable. |
| `504` | An ambiguous upstream timeout — the action **may or may not** have completed. Check the asset before retrying rather than assuming it failed. |

## Billing

Organic publishing, comment moderation and messaging on Facebook and Instagram
are **not metered** — Meta does not charge for them, and neither do we. No credits
are reserved or deducted on these endpoints. (The [Post Studio](/docs/social-posts),
which generates an image and a caption with AI, is metered for that generation —
see its page.)
