---
title: "Web Search API"
description: "Search the live web through a single endpoint with a normalised result shape. From ₹1 per search."
slug: "web-search"
breadcrumb: "Images & Search"
---

# Web Search API

Search the live web through a single endpoint with a normalised result shape. From ₹1 per search.

## Overview

One endpoint for live web search. Pick a `mode`, or name a search `provider` explicitly. We route the query, fail over to another provider if one is down, and return the same normalised response shape whichever provider answered.

**Endpoint:** `POST /v1/search`

**Auth:** `Authorization: Bearer cm_your_key` — the key must have the `search` permission (or `*`).

**Cost:** 1 credit (= ₹1) per successful search. Searches answered by `exa` add 0.1 credit for each result above 10 (so `num_results: 50` on `exa` costs 5 credits). Failed upstream calls are not charged.

:::flow
icon:app | Your app | Send a `query` + `mode` to `POST /v1/search`
icon:gateway | CallMissed gateway | Check the `search` permission and pick the provider for the mode
icon:search | Search provider | Picked from `mode` or `provider`, with automatic failover
icon:done | Your app | Receive a normalized result list; you are charged only on success
:::

## Basic Usage

:::tabs
```python [Python]
import httpx

r = httpx.post(
    "https://api.callmissed.com/v1/search",
    headers={"Authorization": "Bearer cm_your_key"},
    json={
        "query": "latest Indian AI startups raising funding",
        "mode": "shorter",       # or "detailed" / "auto"
        "num_results": 10,
    },
    timeout=15,
)
print(r.json()["results"][:3])
```
```javascript [JavaScript]
const res = await fetch("https://api.callmissed.com/v1/search", {
  method: "POST",
  headers: {
    "Authorization": "Bearer cm_your_key",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    query: "latest Indian AI startups raising funding",
    mode: "shorter",            // or "detailed" / "auto"
    num_results: 10,
  }),
});
const data = await res.json();
console.log(data.results.slice(0, 3));
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/search \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "latest Indian AI startups raising funding",
    "mode": "shorter",
    "num_results": 10
  }'
```
:::

## Modes

| `mode` | Provider | Default `search_type` |
|------|------|------|
| `shorter` | `serper` | `search` |
| `detailed` | `serper` | `auto` |
| `auto` (default) | your account's default mode if one is set, otherwise the platform default | — |

Set `provider` to `serper`, `exa`, `firecrawl` or `linkup` to choose directly; when both `mode` and `provider` are set, `provider` wins. If the chosen provider fails, the request is retried on another provider your key allows, so the `provider` field in the response may differ from the one you asked for. All providers return the same normalised shape.

Operators can set the **tenant default** from **Settings → Web search default**.

## Request Body

| Field | Type | Default | Notes |
|---|---|---|---|
| `query` | string | (required) | 1–2000 chars |
| `mode` | string | `"auto"` | `auto` / `shorter` / `detailed` |
| `provider` | string | — | Optional raw override: `serper` / `exa` / `firecrawl` / `linkup`. Wins over `mode` |
| `num_results` | int | `10` | 1–50 |
| `search_type` | string | mode default | `exa`: `auto`/`fast`/`instant`/`deep-lite`/`deep`. `serper`: `search`/`news`/`images`. Ignored by `linkup` |
| `include_domains` | string[] | — | Up to 50. Honoured by `exa` and `linkup` |
| `exclude_domains` | string[] | — | Up to 50. Honoured by `exa` and `linkup` |
| `start_published_date` | `YYYY-MM-DD` | — | Honoured by `exa` and `linkup`. Any other format returns `422` |
| `end_published_date` | `YYYY-MM-DD` | — | Honoured by `exa` and `linkup`. Any other format returns `422` |
| `include_content` | bool | `false` | Fetch page text into `content`. Honoured by `exa` and `firecrawl` (`firecrawl` also does this in `detailed` mode) |
| `gl` | string | — | Country code (e.g. `in`, `us`). Honoured by `serper` |
| `hl` | string | — | Language code. Honoured by `serper` |
| `tbs` | string | — | Time filter, e.g. `qdr:d` (past day). Honoured by `serper` |

A filter the serving provider does not honour is ignored, not rejected. Name the `provider` explicitly when a filter must apply.

## Response Shape

Responses are **normalised across providers** — same keys regardless of which backend ran the query.

```json
{
  "query": "latest Indian AI startups raising funding",
  "mode": "shorter",
  "provider": "serper",
  "results": [
    {
      "title": "Acme AI raises $...",
      "url": "https://example.com/article",
      "snippet": "Acme AI announced a funding round led by...",
      "content": "Full text if include_content=true, else null",
      "published_date": "2026-04-10T00:00:00.000Z",
      "score": 0.92,
      "source": "example.com"
    }
  ],
  "answer": "Optional grounded answer (provider-dependent)",
  "images": null,
  "credits_used": 1,
  "balance": 495.0,
  "request_id": "search-a3f8c1d2e0b9",
  "latency_ms": 920
}
```

## Pricing & Credits

- **Base rate:** 1 credit per successful search. 1 credit = ₹1.
- **`exa` surcharge:** +0.1 credit per result above 10, charged on the provider that actually served the search.
- Failed requests (upstream 5xx, rate limits, etc.) are **not charged**.
- The charge is visible immediately in the `credits_used` + `balance` fields on the response, and in your credit history in the dashboard.
- Per-key budget caps and the account's monthly budget cap both apply — hitting either returns HTTP 402.

## Permissions

API keys must have `search` (or `*`) in their service permissions. Edit a key in your dashboard (↳ **Profile → API Keys → Edit**) and tick the **Search** chip.

You can also restrict which **search providers** a single key may call by editing **Allowed web-search providers** on the same edit panel. A key restricted to `serper` can still use the endpoint, but calls with `mode: "detailed"` (or `provider: "exa"`) will return **HTTP 403 `search_provider_not_allowed`** before any upstream request is made.

## Errors

Error envelope matches the rest of `/v1`:

```json
{ "error": { "message": "...", "type": "...", "code": "..." } }
```

| Status | Code | Meaning |
|---|---|---|
| 401 | `invalid_api_key` | Bad or revoked key |
| 402 | `insufficient_credits` | Balance &lt; 1 credit or monthly budget exhausted |
| 403 | `permission_denied` | Key lacks `search` permission |
| 403 | `search_provider_not_allowed` | You named a `provider` the key does not allow, or no provider the key allows is available |
| 422 | — | Validation failed: `query` length, `num_results` outside 1–50, an unknown `provider`, a malformed date |
| 429 | `rate_limit_exceeded` | Per-key RPM exceeded |
| 503 | `provider_error` | No search provider could answer. Retry shortly |
