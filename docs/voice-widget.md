---
title: "Voice Widget"
description: "Put a talk-to-us button on your website that starts a live voice call with one of your agents, locked to the sites you list."
slug: "voice-widget"
breadcrumb: "Voice Agents"
---

# Voice Widget

Put a talk-to-us button on your website that starts a live voice call with one of your agents, locked to the sites you list.

## Overview

A **voice widget** puts a floating call button on your website. A visitor clicks it, allows the microphone, and talks to one of your voice agents in the browser. The call runs the agent exactly as it is configured (prompt, voice, tools), and is billed like any other voice session.

Each widget has a **publishable key** (`pk_...`). The key is public by design: it lives in your page's HTML. It is safe there because it only works from the sites on the widget's `allowed_origins` list, and each widget carries its own caps.

For logged-in experiences you can turn on `require_signed_token`. Your server then mints a short-lived token for each visitor with your API key, and can attach the visitor's id and per-call variables to it.

## Authentication

The management endpoints on this page take `Authorization: Bearer cm_your_api_key`. Reading widgets needs the `bots:read` scope; creating, changing, deleting and minting tokens needs `bots:write`. A key without the scope gets `403`.

Never put a `cm_` key in a web page. The browser only ever sees the publishable key or a signed token.

## Embed the button

Paste this before `</body>` on every page that should show the button:

```html
<script
  src="https://api.callmissed.com/api/v1/voice-widget/loader.js"
  data-publishable-key="pk_your_publishable_key"
  data-label="Talk to us"
  data-color="#111827"
  data-position="bottom-right"
  async
></script>
```

| Attribute | Required | Description |
|-----------|----------|-------------|
| `data-publishable-key` | Yes, unless `data-token-url` is set | The widget's `publishable_key` |
| `data-token-url` | For signed-token widgets | A URL on your own site that returns `{"token": "vwt_..."}` (see below). The button calls it with a `POST` and the visitor's cookies |
| `data-label` | No | Button text and accessible name, up to 40 characters. Default `Talk to us` |
| `data-color` | No | Button colour as a hex value, for example `#0f766e` |
| `data-position` | No | `bottom-right` (default) or `bottom-left` |

The button asks for the microphone when the visitor clicks it, shows the call state next to it, and ends the call on a second click. The page that embeds it must be served over HTTPS (browsers only grant the microphone on secure pages; `http://localhost` works for development).

### Your own button

The script also exposes `window.CallMissedVoice`, so you can start a call from your own UI:

```html
<script src="https://api.callmissed.com/api/v1/voice-widget/loader.js" async></script>
<script>
  async function talk() {
    const call = await window.CallMissedVoice.start({
      publishableKey: "pk_your_publishable_key",
      onState: (state, detail) => console.log(state, detail?.error),
      onTranscript: (t) => console.log(t.role, t.text, t.final),
    });
    // later: await call.end();  await call.setMuted(true);
  }
</script>
```

`state` is one of `connecting`, `connected`, `ended` or `error`. Each transcript event has an `id`; an utterance arrives as several events with the same `id`, the last one with `final: true`.

## Widgets

### Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `bot_id` | uuid | required | The voice agent the widget calls. Set at creation |
| `allowed_origins` | string[] | required | 1 to 20 origins, each `https://host` or `https://host:port`. No paths and no wildcards; `http://` only for `localhost` and `127.0.0.1` |
| `require_signed_token` | bool | `false` | Refuse the bare publishable key; every call needs a signed token |
| `enabled` | bool | `true` | A disabled widget starts no calls |
| `max_session_seconds` | int | `300` | Longest call, 30 to 3600 seconds |
| `daily_session_cap` | int | `200` | Calls this widget may start in any 24 hours, 1 to 100000 |
| `per_ip_hourly_cap` | int | `10` | Calls one visitor IP may start per hour, 1 to 1000 |
| `theme` | object | `null` | Optional `color` (hex), `position` (`bottom-right` or `bottom-left`) and `label` (up to 40 characters), kept for your own snippet generation |

An origin must match **exactly**: a widget allowed on `https://shop.example.com` does not run on `https://example.com` or `https://www.shop.example.com`. List each site you use.

### POST `/api/v1/voice-widgets`

```bash
curl -X POST https://api.callmissed.com/api/v1/voice-widgets \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "bot_id": "6f1c2c1e-0b7a-4d55-9b2e-6a0c7d1f9a10",
    "allowed_origins": ["https://shop.example.com", "https://www.shop.example.com"],
    "max_session_seconds": 300,
    "per_ip_hourly_cap": 5
  }'
```

Returns `201` with the widget:

```json
{
  "id": "0d3a9a52-6a43-4c7e-8f8e-5f2f2f7d3b11",
  "bot_id": "6f1c2c1e-0b7a-4d55-9b2e-6a0c7d1f9a10",
  "publishable_key": "pk_Q2x1c3RlcktleUV4YW1wbGVWYWx1ZQ",
  "allowed_origins": ["https://shop.example.com", "https://www.shop.example.com"],
  "require_signed_token": false,
  "enabled": true,
  "max_session_seconds": 300,
  "daily_session_cap": 200,
  "per_ip_hourly_cap": 5,
  "theme": null,
  "created_at": "2026-10-08T09:30:00Z",
  "updated_at": "2026-10-08T09:30:00Z"
}
```

### GET `/api/v1/voice-widgets`

Lists your widgets, newest first. Optional query parameters: `bot_id`, `limit` (1 to 100, default 50) and `offset`.

### GET / PATCH / DELETE `/api/v1/voice-widgets/{widget_id}`

`PATCH` takes any of the fields above except `bot_id`; an omitted field keeps its value. To point a site at another agent, create a new widget. `DELETE` returns `204`, and the publishable key stops working at once.

## Signed tokens

### POST `/api/v1/voice-widgets/{widget_id}/tokens`

Mint one token per visitor on your server, and hand it to the page. A token is valid for **5 minutes**, works only for this widget, and is checked against the same `allowed_origins`.

| Field | Type | Description |
|-------|------|-------------|
| `end_user_id` | string, optional | Your id for the visitor, up to 128 characters. Stored on the call's session `metadata` |
| `variables` | object, optional | Per-call [`{{variable}}`](/docs/voice-sessions-api) values for the agent's prompt and greeting. Up to 20 names (letters, digits, underscores), string values up to 500 characters, 4 KB in all |

```bash
curl -X POST https://api.callmissed.com/api/v1/voice-widgets/0d3a9a52-6a43-4c7e-8f8e-5f2f2f7d3b11/tokens \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"end_user_id": "cust_1042", "variables": {"customer_name": "Asha"}}'
```

```json
{
  "token": "vwt_eyJ1IjoiY3VzdF8xMDQyIn0.c2lnbmF0dXJl",
  "expires_at": "2026-10-08T09:35:00Z",
  "expires_in": 300
}
```

A typical setup: your site exposes `POST /voice-token`, which checks the visitor's own login, calls this endpoint with your API key, and returns `{"token": ...}`. Point the button at it with `data-token-url="/voice-token"`, or pass the token yourself: `CallMissedVoice.start({ token })`.

Variables in a token count as explicit values: one that does not fit the agent's declared type, or a missing required variable, refuses the call.

## The calls

Every call a widget starts is an ordinary voice session: it appears in your call history, its transcript is under [Voice Sessions](/docs/voice-sessions-api), and your `voice_session.*` webhooks fire for it. The session's `metadata` carries `source: "widget"`, the `widget_id`, and the `end_user_id` when a token supplied one.

A call starts only when your account could start any other voice session: enough credits, inside your monthly budget, and under your plan's concurrent-call limit. When it cannot, or a widget cap is reached, the visitor sees a short "not available" or "busy" message; balances and plan details are never shown to visitors.

## Errors

Management endpoints:

| Status | Meaning |
|--------|---------|
| `403` | API key missing the `bots:read` / `bots:write` scope |
| `404` | Widget (or agent) not found in your account |
| `422` | Invalid field: a bad origin, a value out of range, or token variables over the limits |
