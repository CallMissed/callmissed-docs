---
title: "Comments"
description: "Read, reply to, hide and delete comments on Facebook Page posts and Instagram media."
slug: "social-comments"
breadcrumb: "Facebook & Instagram"
---

# Comments

Read, reply to, hide and delete comments on Facebook Page posts and Instagram media.

Read and moderate comments on a connected Facebook Page's posts and an Instagram
account's media. Reading needs `whatsapp:read`; replying, hiding and deleting need
`whatsapp:send`. None of it is metered.

Comment lists are **cursor-paginated**: a response carries an `after` cursor; pass
it back as the `after` query parameter to get the next page. There is no offset —
an offset would skip comments as new ones arrive. `limit` is 1–100 (CallMissed's
own bound; default 25).

## Facebook: list comments on a post

`GET /api/v1/facebook/{page_uuid}/posts/{post_id}/comments`

| Query | Type | Description |
|---|---|---|
| `filter` | string | `toplevel` (top-level comments) or `stream` (the flattened thread). |
| `summary` | boolean | Include Meta's `total_count` (default `true`). |
| `limit` | integer | Page size, 1–100 (default 25). |
| `after` | string | The cursor from a previous response. |

```json
{
  "data": [
    {
      "id": "1122334455667788_9988776655",
      "message": "Do you deliver to Pune?",
      "author": { "id": "7788990011223344", "name": "Ritu Sharma" },
      "created_time": "2026-08-24T09:14:02+0000",
      "like_count": 3,
      "comment_count": 1,
      "parent_id": null,
      "can_comment": true
    }
  ],
  "after": "QVFIUmxr...",
  "total_count": 42
}
```

`author` is omitted when Meta withholds the commenter's identity (a permissions
state, not an error) — treat it as optional. `total_count` is `null` when `summary`
is false or Meta omits it.

## Facebook: list replies to a comment

`GET /api/v1/facebook/{page_uuid}/comments/{comment_id}/replies`

On Facebook a reply is a comment one level down. `parent_id` carries the id of the
comment being replied to. Same `limit`/`after` paging as above; no `filter` (it is
meaningless one level down).

## Instagram: list comments on media

`GET /api/v1/instagram/{ig_account_uuid}/media/{media_id}/comments`

| Query | Type | Description |
|---|---|---|
| `limit` | integer | Page size, 1–100 (default 25). |
| `after` | string | The cursor from a previous response. |

```json
{
  "data": [
    {
      "id": "17924118234567890",
      "text": "Do you ship to Bengaluru?",
      "timestamp": "2026-08-24T09:14:03+0000",
      "username": "anita.makes",
      "like_count": 3,
      "hidden": false,
      "parent_id": null,
      "reply_count": 2
    }
  ],
  "after": "QVFIUmp0WFZ6..."
}
```

There is **no `total_count`** on Instagram — its comments edge documents no total,
so none is invented. Every field except `id` is nullable: reading another
commenter's `username` needs the comment-management permission, and Meta omits
what a token may not see. `reply_count` is a count; fetch the replies themselves
from the replies endpoint.

## Instagram: list replies to a comment

`GET /api/v1/instagram/{ig_account_uuid}/comments/{comment_id}/replies`

On Instagram a reply lives on a **distinct** edge (unlike Facebook), so replies
are read here, not from the media-comments endpoint. Same response shape and
paging as the media comments.

## Reply to a comment

Facebook — `POST /api/v1/facebook/{page_uuid}/comments/{comment_id}/reply`
Instagram — `POST /api/v1/instagram/{ig_account_uuid}/comments/{comment_id}/reply`

| Field | Type | Description |
|---|---|---|
| `message` | string | The reply text. Required. |

```json
{ "id": "17924118301122334" }
```

## Hide or unhide a comment

Facebook — `POST /api/v1/facebook/{page_uuid}/comments/{comment_id}/hide`
Instagram — `POST /api/v1/instagram/{ig_account_uuid}/comments/{comment_id}/hide`

Hiding removes a comment from public view without deleting it. Send `hidden`:

| Field | Type | Description |
|---|---|---|
| `hidden` | boolean | `true` to hide, `false` to unhide. Required. |

## Delete a comment

Facebook — `DELETE /api/v1/facebook/{page_uuid}/comments/{comment_id}`
Instagram — `DELETE /api/v1/instagram/{ig_account_uuid}/comments/{comment_id}`

Permanently deletes a comment you have permission to remove. You can only delete
comments on your own posts/media, or your own comments — Meta enforces the rest.
