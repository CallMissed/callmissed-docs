---
title: "Response Cache"
description: "Inspect and purge the gateway's cached completions — hit counts, cached token totals, and targeted or full invalidation."
slug: "gateway-cache"
breadcrumb: "API Reference"
---

# Response Cache

Inspect and purge the gateway's cached completions — hit counts, cached token totals, and targeted or full invalidation.

## Overview

The gateway can serve a repeated completion from cache instead of re-running the model. This endpoint group lets you see what your tenant currently has cached and throw it away when the underlying facts change.

Every entry is tenant-scoped. You never see, count or purge another tenant's cache.

A typical use: your knowledge base changed, so the cached answers about it are now wrong. Purge the cache and let the next request repopulate it.

## What gets cached

- **Only `POST /v1/chat/completions`, non-streaming.** Streaming requests are never cached.
- **Only deterministic requests:** `temperature` of `0` (or omitted), or any `temperature` together with a `seed`. A request with `temperature > 0` and no `seed` asks for variety, so it is never served from cache.
- **Exact match only.** The whole request body must be identical to an earlier one from your account.
- **Never under [zero data retention](/docs/gateway-controls)**, never an error response, and never a response over 1 MB.
- Entries expire **one hour** after they were stored.

Every non-streaming response carries an `X-CM-Cache` header: `hit` (served from cache), `miss` (looked up, not found, now stored), or `bypass` (the request is not cacheable).

A cache hit is billed **zero credits** — no model ran. It still appears in your [usage logs](/docs/usage-api) and counts toward your plan's monthly call cap.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| `GET /stats` | `cache:read` |
| `DELETE` (all or one key) | `cache:write` |

## GET `/api/v1/gateway/cache/stats`

Counts only live, unexpired entries.

```bash
curl https://api.callmissed.com/api/v1/gateway/cache/stats \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "entries": 1284,
  "total_hits": 9317,
  "bytes": 4218904,
  "cached_prompt_tokens": 2841002,
  "cached_completion_tokens": 512884
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `entries` | `integer` | Live cached responses |
| `total_hits` | `integer` | Times an entry has been served from cache |
| `bytes` | `integer` | Approximate stored size |
| `cached_prompt_tokens` | `integer` | Input tokens a cache hit avoided re-sending |
| `cached_completion_tokens` | `integer` | Output tokens a cache hit avoided re-generating |

The two token totals are the value the cache has returned so far — multiply them by the model's rate to see what you saved.

## DELETE `/api/v1/gateway/cache`

Purges every cached entry for your tenant.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/gateway/cache \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "deleted": 1284 }
```

Safe but not free: the next request for each purged prompt runs the model again and is billed normally.

## DELETE `/api/v1/gateway/cache/{cache_key}`

Purges one entry.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `cache_key` | `string` | 64 lowercase hex characters (a SHA-256 digest) |

```bash
curl -X DELETE https://api.callmissed.com/api/v1/gateway/cache/3b9f…c1 \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "deleted": 1 }
```

A key that is not 64 hex characters returns `422`. A well-formed key with no live entry returns `404 Cache entry not found`.

The API does not currently return cache keys (there is no endpoint that lists entries), so in practice use the full purge above.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `cache:read` / `cache:write` |
| `404` | `Cache entry not found` |
| `422` | `cache_key` is not a 64-character hex digest |

Reading stats and purging never consume credits.
