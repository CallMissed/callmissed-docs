---
title: "Web Search API"
description: "Search the live web through a single endpoint. Two modes — shorter (Serper / Google) and detailed (Exa / neural). Flat ₹1 per search."
slug: "web-search"
breadcrumb: "API Guides & Tutorials"
---

# Web Search API

Search the live web through a single endpoint. Two modes — shorter (Serper / Google) and detailed (Exa / neural). Flat ₹1 per search.

## Overview

One endpoint, two modes. Pick **shorter** (Serper — Google SERP, sub-second) or **detailed** (Exa — neural search with optional page content + highlights). We route for you, charge a flat ₹1 per search, and return a normalised response shape.

**Endpoint:** `POST /v1/search`

**Auth:** `Authorization: Bearer cm_your_key` — the key must have the `search` permission (or `*`).

**Cost:** 1 credit (= ₹1) per successful search, regardless of mode or number of results. Failed upstream calls are not charged.

:::flow
icon:app | Your app | Send a `query` + `mode` to `POST /v1/search`
icon:gateway | CallMissed gateway | Check the `search` permission and pick the provider for the mode
icon:search | Search provider | **shorter** → Serper (Google SERP) · **detailed** → Exa (neural)
icon:done | Your app | Receive a normalized result list and get charged ₹1 only on success
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

| `mode` | Underlying | Best for | p50 latency |
|------|------|------|------|
| `shorter` | [Serper](https://serper.dev) Google SERP | freshness, location-aware, familiar Google-style results | ~1s |
| `detailed` | [Exa](https://exa.ai) neural search | semantic / conceptual queries, optional page content + highlights | ~1–3s |
| `auto` | tenant default → platform default | let CallMissed pick | depends |

You can also set `provider: "firecrawl"` for our managed search backend, which returns the same normalised shape; with `include_content: true` it returns full page content alongside each result.

You can also override with `provider: "exa" | "serper"` directly. When both `mode` and `provider` are set, `provider` wins.

Operators can set the **tenant default** from **Settings → Web search default**.

## Request Body

| Field | Type | Default | Notes |
|---|---|---|---|
| `query` | string | (required) | 1–2000 chars |
| `mode` | string | `"auto"` | `auto` / `shorter` / `detailed` |
| `provider` | string | — | Optional raw override: `exa` / `serper` / `firecrawl`. Wins over `mode` |
| `num_results` | int | `10` | 1–50 |
| `search_type` | string | mode default | Exa: `auto`/`fast`/`instant`/`deep-lite`/`deep`. Serper: `search`/`news`/`images` |
| `include_domains` | string[] | — | detailed mode only |
| `exclude_domains` | string[] | — | detailed mode only |
| `start_published_date` | `YYYY-MM-DD` | — | detailed mode only |
| `end_published_date` | `YYYY-MM-DD` | — | detailed mode only |
| `include_content` | bool | `false` | detailed mode only — fetch page text + highlights |
| `gl` | string | — | shorter mode only — country ISO (e.g. `in`, `us`) |
| `hl` | string | — | shorter mode only — language ISO |
| `tbs` | string | — | shorter mode only — time filter, e.g. `qdr:d` (past day) |

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

- **Flat rate:** 1 credit per successful search. 1 credit = ₹1.
- Failed requests (upstream 5xx, rate limits, etc.) are **not charged**.
- The charge is visible immediately in `credits_used` + `balance` fields on the response and in your `/api/v1/credits/transactions` history.
- Per-key budget caps and the tenant monthly budget cap both apply — hitting either returns HTTP 402 `insufficient_credits`.

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
| 400 | `invalid_request_error` | Missing or malformed body |
| 401 | `invalid_api_key` | Bad or revoked key |
| 402 | `insufficient_credits` | Balance < 1 credit or monthly budget exhausted |
| 403 | `permission_denied` | Key lacks `search` permission |
| 403 | `search_provider_not_allowed` | Key's Allowed search providers excludes the requested provider |
| 429 | `rate_limit_exceeded` | Per-key RPM exceeded |
| 503 | `provider_error` | Upstream search provider unavailable |