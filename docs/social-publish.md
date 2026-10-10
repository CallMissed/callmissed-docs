---
title: "Publishing"
description: "Publish text, links, photos, reels, carousels and stories to a Facebook Page and Instagram professional account."
slug: "social-publish"
breadcrumb: "Facebook & Instagram"
---

# Publishing

Publish text, links, photos, reels, carousels and stories to a Facebook Page and Instagram professional account.

Publish organic content to a connected Facebook Page or Instagram professional
account. All publishing endpoints need the `whatsapp:send` scope, and none is
metered — Meta does not charge for organic publishing.

No account id goes in the path. The Page or Instagram account is resolved from
your workspace: its only connected account, or the one marked default. Pass
`?account=` (a CallMissed id or Meta's `page_id` / `ig_user_id`) to publish as a
specific account — see [how an account is chosen](/docs/social-api#how-an-account-is-chosen).

Media you publish must be at a **publicly reachable URL**: Meta fetches the file
from its own servers, so a signed or expiring link that Meta cannot read fails as
a `422`. The public image URL returned by the [Post Studio](/docs/social-posts) is
exactly this shape.

## Facebook: publish a post

`POST /api/v1/facebook/posts`

A text post, a link post, or a scheduled version of either. Supply `message`,
`link`, or both — a body with neither is a `422`.

| Field | Type | Description |
|---|---|---|
| `message` | string | Post body text, up to 63,206 characters. Optional if `link` is set. |
| `link` | string | A URL to attach as a link post, up to 2,048 characters. Optional if `message` is set. |
| `scheduled_publish_time` | integer | A UNIX timestamp. When present, the post is created **unpublished and scheduled** instead of going live now. Must be **10 minutes to 75 days** out, or `422`. |

```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/facebook/posts \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{ "message": "Winter service special is on now.", "link": "https://example.com/book" }'
```

```json
{ "id": "102938475610293_890123456789012", "post_id": null, "video_id": null }
```

## Facebook: publish a photo or multi-photo post

`POST /api/v1/facebook/photos`

Supply `url` for a single photo, or `urls` for a **multi-photo** post (2–30). A
single-URL `urls` list falls through to the single-photo path.

| Field | Type | Description |
|---|---|---|
| `url` | string | One publicly reachable photo URL. |
| `urls` | string[] | 2–30 photo URLs for a single multi-photo post. |
| `caption` | string | Caption for a single photo. Used as the feed text of a multi-photo post when `message` is absent. |
| `message` | string | Feed text for a multi-photo post. Used as the caption of a single photo when `caption` is absent. |
| `published` | boolean | Single photo only. `false` uploads the photo without showing it in the feed. Default `true`. |

Supply `url` or `urls` — a body with neither is a `422`.

The 30-URL cap is CallMissed's own bound (each URL is an upload call), not Meta's.
The response carries both the photo `id` and the resulting feed `post_id`.

```json
{ "id": "10160000000000000", "post_id": "102938475610293_890123456789012" }
```

## Facebook: publish a reel

`POST /api/v1/facebook/reels`

Publish a video reel from a publicly reachable `video_url`.

| Field | Type | Description |
|---|---|---|
| `video_url` | string | Publicly reachable video URL, up to 2,048 characters. Required. |
| `description` | string | Reel caption. |
| `title` | string | Reel title, up to 255 characters. |
| `scheduled_publish_time` | integer | A UNIX timestamp to schedule the reel. Reels have their **own** schedule window — **10 minutes to 29 days** out, or `422`. |

The response carries the reel's `video_id`. Meta caps a Page at **30 reels
published through the API per rolling 24 hours**; past that the call returns
`429`.

```json
{ "id": null, "post_id": null, "video_id": "1029384756102938" }
```

## Instagram: publish a post

`POST /api/v1/instagram/posts`

One endpoint for every Instagram post shape. The `type` field picks it:

| `type` | What it makes | Media field(s) |
|---|---|---|
| `image` | A single feed image | `image_url` |
| `video` | A single video, published as a reel (Instagram no longer accepts standalone feed videos) | `video_url` |
| `reel` | A reel | `video_url` (+ reel-only options) |
| `story` | An image or video story | exactly one of `image_url` / `video_url` |
| `carousel` | A 2–10 item carousel | `items[]` |

| Field | Type | Description |
|---|---|---|
| `type` | string | `image`, `video`, `reel`, `story` or `carousel`. Required. |
| `image_url` | string | Image URL. Required for `image`; one of the two for `story`. |
| `video_url` | string | Video URL. Required for `video`/`reel`; one of the two for `story`. |
| `caption` | string | Caption, up to 2,200 characters. Not used for stories or carousel children. |
| `alt_text` | string | Accessibility alt text; images only. |
| `location_id` | string | A location tag id for the post. |
| `items` | array | Carousel children (2–10), each with one of `image_url`/`video_url` and optional `alt_text`. Required for `carousel`. |
| `cover_url` | string | `video`/`reel` only: cover image URL. |
| `share_to_feed` | boolean | `video`/`reel` only: also show the reel in the main feed. |
| `thumb_offset` | integer | `video`/`reel` only: the frame offset to use as the thumbnail (≥ 0). If `cover_url` is also set, `cover_url` wins. |
| `audio_name` | string | `video`/`reel` only: a name for the reel's audio. |

Caption length (≤ 2,200) and carousel size (2–10) are validated **before** any
upstream call, so a bad body fails fast without leaving orphaned containers. A
reel cannot be a carousel child (use a `video` child instead), and each child
carries **exactly one** of `image_url` / `video_url`. A story takes no caption.

```bash [cURL]
# A 3-image carousel
curl -X POST https://api.callmissed.com/api/v1/instagram/posts \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "carousel",
    "caption": "Three from this week'"'"'s firing.",
    "items": [
      { "image_url": "https://media.callmissed.com/a.jpg", "alt_text": "A speckled mug" },
      { "image_url": "https://media.callmissed.com/b.jpg" },
      { "image_url": "https://media.callmissed.com/c.jpg" }
    ]
  }'
```

### The two-step finish for a slow Instagram post

Instagram publishing is a multi-step upstream flow: a container is created, Meta
processes the media, then the container is published. This endpoint waits a short
budget (about 27 seconds) for that to finish. If the media is still processing
when the budget runs out, it returns the `container_id` with status
`IN_PROGRESS` instead of hanging — **do not re-post** (that risks a duplicate).
Finish it with the two routes below.

```json
{ "media_id": "17895695668004550", "container_id": "17889455560051444", "status": "PUBLISHED" }
```

**Check a container's status** — `GET /api/v1/instagram/posts/{container_id}`

Poll this until `status_code` is `FINISHED`, then publish. Poll no more than once
a minute. A container **expires 24 hours** after creation. `status_code` is one of
`IN_PROGRESS`, `FINISHED`, `PUBLISHED`, `ERROR` or `EXPIRED`.

```json
{ "container_id": "17889455560051444", "status_code": "FINISHED" }
```

**Publish a finished container** — `POST /api/v1/instagram/posts/{container_id}/publish`

Publishes a container whose media has finished processing, returning the live
`media_id`.

Both need `whatsapp:send` and take the same optional `?account=` hint. Use the **same** account you created
the container with — a container belongs to the account that made it, so resolving
to a different one fails upstream.

## Instagram: check the publishing quota

`GET /api/v1/instagram/publishing_limit`

Instagram caps how many posts an account can publish in a rolling 24-hour window.
Read the **live** quota rather than assuming a number (Meta's own docs quote
different limits on different pages). A carousel counts as **one** post against
the quota. Needs `whatsapp:send`, like the publishing calls it protects.

```json
{ "quota_usage": 12, "quota_total": 100, "quota_duration": 86400 }
```

`quota_total` and `quota_duration` (seconds) are forwarded from Meta as
reported; any of the three can be `null` when Meta omits it.

## Scheduling windows at a glance

| Surface | Schedule window |
|---|---|
| Facebook feed post (`/posts`) | 10 minutes – 75 days |
| Facebook reel (`/reels`) | 10 minutes – 29 days |
| Instagram | No native scheduling — publish at the time you want to post |

A Facebook `503` timeout on publish is **ambiguous** — the post may or may not have gone
live. Check the Page before retrying; the API never auto-retries a publish. The
Instagram equivalent is a `409` saying the post may or may not have been created —
handle it the same way. A Facebook `409` on publish can also mean Meta rejected a
**duplicate** of the post you just published; change the content before retrying.

The legacy `POST /api/v1/facebook/{page_uuid}/posts` form of every endpoint on
this page still works — see [legacy paths](/docs/social-api#legacy-the-account-id-in-the-path).
