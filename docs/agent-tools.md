---
title: "Agent Tools"
description: "Create, test and manage the custom REST tools your WhatsApp and voice agents call at reply time, using the same definition the console writes."
slug: "agent-tools"
breadcrumb: "API Reference"
---

# Agent Tools

Create, test and manage the custom REST tools your WhatsApp and voice agents call at reply time, using the same definition the console writes.

> A custom tool is one of your own read-only HTTP endpoints, described so a model can call it mid-conversation. These routes are the API behind the console's **Tools** screen: create the definition, test it against your real endpoint, and attach it to one agent or to all of them.

:::cards
/docs/connect-your-store | Connect Your Store | store | The guide these endpoints automate, including how to bind a lookup to the customer
/docs/bots | Bots | bot | Built-in tools, enabled by name through `config.tools`
:::

## Custom tools vs built-in tools

Two different mechanisms, and this page covers only the first.

| | Custom REST tools | Built-in tools |
| --- | --- | --- |
| What it is | Your endpoint, described by you | Ours: WhatsApp sends, CRM reads, storefront connectors |
| Managed by | These endpoints | `config.tools` on the bot, listed by [`GET /api/v1/bots/tool-catalog`](/docs/bots) |
| Attached by | `bot_id` on the tool row | Tool name in the bot's `config` |

A tool with `bot_id: null` is available to **every** agent in the workspace. Set `bot_id` to scope it to one.

## Credential class: dashboard JWT **or** a `cm_` key

```
Authorization: Bearer cm_your_api_key
```

| Caller | Requirement |
| --- | --- |
| `cm_` API key | `bots:read` to list and read. `bots:write` to create, update, delete and test |
| Dashboard JWT | Read for any member. Create, update, delete and test require **owner or admin** (`403 Only owners/admins can manage agent tools` otherwise) |

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`.

<Callout type="warn">
  A custom tool makes real outbound requests with credentials you store, exactly like a webhook. `bots:write` is a powerful scope: give it only to keys that need to author tools, and use a `bots:read` key for anything that only inspects them.
</Callout>

## The tool object

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "bot_id": null,
  "name": "search_products",
  "description": "Search the catalogue by keyword. Returns price, stock, the product link and an image URL.",
  "method": "GET",
  "url": "https://api.yourstore.com/agent/products/search",
  "header_params": [
    { "name": "X-API-Key", "type": "string", "description": "", "required": false, "source": "static", "secret": true, "value": null }
  ],
  "path_params": [],
  "query_params": [
    { "name": "q", "type": "string", "description": "what the customer is looking for", "required": true, "source": "llm", "secret": false, "value": null }
  ],
  "body_params": [],
  "response_mapping": {},
  "timeout_seconds": 10,
  "phase": "on_call",
  "enabled": true,
  "has_secrets": true,
  "created_at": "2026-09-08T12:00:00Z",
  "updated_at": "2026-09-08T12:00:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `name` | `string` | 1-64 chars. Must start with a letter; letters, digits and underscores only. This is the name the model sees, so write it as a verb: `search_products`, `get_order_status`. Unique per workspace |
| `description` | `string` | Max 2000 chars. The **only** thing telling the model when to reach for this tool. Write it as an instruction, not a label |
| `method` | `string` | `GET`, `POST`, `PUT`, `PATCH` or `DELETE`. Default `GET` |
| `url` | `string` | Max 2048 chars, `http` or `https`, with a hostname. May contain `{placeholders}` in the path or query only |
| `header_params` | `array` | Up to 20 per location. See [Parameters](#parameters) |
| `path_params` | `array` | Every declared name must appear as a `{placeholder}` in the URL, and vice versa |
| `query_params` | `array` | |
| `body_params` | `array` | |
| `response_mapping` | `object` | `{output_key: "dotted.path"}`, up to 20 entries. Empty means the raw body is returned |
| `timeout_seconds` | `integer` | 1-30, default 10 |
| `phase` | `string` | `pre_call`, `on_call` or `post_call`. Only `on_call` is resolved into the live tool list today, so leave it at the default |
| `enabled` | `boolean` | A disabled tool is never offered to the model |
| `has_secrets` | `boolean` | Whether an encrypted value is stored. The values themselves are never returned |

<Callout type="info">
  A secret parameter always serialises as `"secret": true, "value": null`. There is no endpoint that reads a stored secret back, including for the account that wrote it.
</Callout>

## Parameters

Each parameter says where it goes on the wire (its **location**) and where its value comes from (its **source**).

**Locations:** `header`, `path`, `query`, `body`. At most 20 each.

**Sources:**

| `source` | Who supplies the value | Use it for |
| --- | --- | --- |
| `llm` | The model, per call | The search term, the order number, anything the customer states |
| `static` | You, once, in `value` | Your API key, a fixed `store_id`, a version pin |
| `context` | The server, per turn, from the live conversation | The identity of the person actually in the chat |

A `context` parameter's `value` is a **key**, not a literal, and must be one of `contact_phone`, `contact_name`, `contact_email`, `conversation_id`. The model cannot see or set it, and that is what makes an order lookup safe. If the conversation has no such value (a voice call with no linked contact), the parameter is omitted unless you mark it `required`, in which case the call fails cleanly instead of running an unscoped query.

| Field | Type | Notes |
| --- | --- | --- |
| `name` | `string` | 1-64 chars, starting with a letter or digit; letters, digits, `_`, `.` and `-`. Unique within its location (headers case-insensitively). Two `llm` parameters may not share a name across locations |
| `type` | `string` | `string`, `integer`, `number` or `boolean`. Header and path parameters must be `string` |
| `description` | `string` | Truncated at 500 chars. Shown to the model for an `llm` parameter |
| `required` | `boolean` | Default `false` |
| `source` | `string` | `llm`, `static` or `context`. Default `llm` |
| `value` | `string` | Required for `static` (max 2048 chars, no control characters) and for `context` (a context key). Ignored otherwise |
| `secret` | `boolean` | Only meaningful with `static`. **Defaults to `true` for a static header**, because that is where API keys live. Elsewhere it defaults to `false` |

<Callout type="warn">
  These headers cannot be set, because the transport computes them or they would let a definition address one origin while connecting to another: `Host`, `Content-Length`, `Transfer-Encoding`, `Connection`, `Upgrade`, `Keep-Alive`, `TE`, `Trailer`, `Expect`, `Proxy-Authorization`, `Proxy-Connection`.
</Callout>

## Endpoint requirements

Your URL must be reachable on the public internet. Requests to private, loopback and cloud-metadata addresses are refused at save time **and** again on every request, including after each redirect, so a hostname re-pointed later does not get through either. At most 3 redirects are followed.

A response is read up to **32 KB** and truncated past it, with `truncated: true` on the result. Return the handful of fields a customer actually asks about rather than a full catalogue document. `/docs/connect-your-store` covers how to shape a response for a model.

## GET /api/v1/agent-tools

Lists the workspace's tools, newest first, up to 500. Requires `bots:read`.

| Parameter | Type | Notes |
| --- | --- | --- |
| `bot_id` | `uuid` | Narrows to that agent's effective set: its own tools **plus** the workspace-wide ones |

```bash
curl "https://api.callmissed.com/api/v1/agent-tools?bot_id=b1f2c3d4-5678-90ab-cdef-1234567890ab" \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns an array of tool objects.

## POST /api/v1/agent-tools

Creates a tool. Returns `201` and the tool object. Requires `bots:write`.

Takes every field of the tool object except the server-set ones (`id`, `tenant_id`, `has_secrets`, `created_at`, `updated_at`), plus `bot_id`, `phase` and `enabled`.

```bash
curl -X POST https://api.callmissed.com/api/v1/agent-tools \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "get_order_status",
    "description": "Look up the caller'\''s most recent orders and their delivery status. Use when a customer asks where their order is.",
    "method": "GET",
    "url": "https://api.yourstore.com/agent/orders",
    "header_params": [
      { "name": "X-API-Key", "source": "static", "value": "your_endpoint_key" }
    ],
    "query_params": [
      { "name": "phone", "source": "context", "value": "contact_phone", "required": true }
    ],
    "timeout_seconds": 10
  }'
```

| Status | Meaning |
| --- | --- |
| `409` | A tool with that name already exists in this workspace |
| `422` | The definition is invalid. The `detail` names the problem: an undeclared path placeholder, a duplicate model-facing name, a static parameter with no value, a forbidden header, a context key outside the allowlist, a timeout outside 1-30 |
| `422` | Tool limit reached. A workspace holds up to 100 tools |
| `404` | `bot_id` is not an agent in this workspace |

## GET `/api/v1/agent-tools/{tool_id}`

One tool object. `404` if it belongs to another workspace. Requires `bots:read`.

## PATCH `/api/v1/agent-tools/{tool_id}`

Every field is optional. Returns the updated tool object. Requires `bots:write`.

The **whole** definition is re-validated on each update, because the URL and the path parameters constrain each other. Omitting a parameter list keeps the stored one; sending a list **replaces** it.

```bash
curl -X PATCH https://api.callmissed.com/api/v1/agent-tools/7c9e6679-7425-40de-944b-e07fc1f90ae7 \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "enabled": false }'
```

<Callout type="info">
  To keep a stored secret, send the parameter with `"secret": true` and no `value`. To rotate it, send the new value. There is no way to read the old one first.
</Callout>

## DELETE `/api/v1/agent-tools/{tool_id}`

Returns `204`. Requires `bots:write`.

## POST /api/v1/agent-tools/test

Runs a definition that has not been saved, so you can check an endpoint before committing to it. Requires `bots:write`.

Takes the same body as `POST /api/v1/agent-tools`, plus `arguments`: the values a model would have supplied for the `llm` parameters.

```bash
curl -X POST https://api.callmissed.com/api/v1/agent-tools/test \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "search_products",
    "description": "Search the catalogue by keyword.",
    "method": "GET",
    "url": "https://api.yourstore.com/agent/products/search",
    "query_params": [ { "name": "q", "source": "llm", "required": true } ],
    "arguments": { "q": "line follower kit" }
  }'
```

## POST `/api/v1/agent-tools/{tool_id}/test`

The same run for a **saved** tool, using its stored secrets. Requires `bots:write`.

```bash
curl -X POST https://api.callmissed.com/api/v1/agent-tools/7c9e6679-7425-40de-944b-e07fc1f90ae7/test \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "arguments": { "q": "line follower kit" } }'
```

## Test results

Both test endpoints return `200` with a result object, whether or not your endpoint answered. A failure of yours is reported in the body, not as an HTTP error on ours.

```json
{
  "ok": true,
  "status": 200,
  "result": { "count": 1, "products": [{ "id": "kit-104", "price": 1180 }] },
  "truncated": false
}
```

| Field | Notes |
| --- | --- |
| `ok` | `true` when your endpoint answered `2xx` |
| `status` | Your endpoint's status code, or `null` when no response was received |
| `result` | The body, or the mapped subset when `response_mapping` is set |
| `truncated` | `true` when the response exceeded 32 KB |
| `error` | Present in place of `result` only when no usable response came back. See the vocabulary below |
| `detail` | Present only with `invalid_arguments`, naming the offending argument |

<Callout type="warn">
  A `4xx` or `5xx` **from your endpoint** is not an `error`. It comes back as `ok: false` with `status` set and `result` holding what your endpoint said, so you can read the reason. Branching on `error` alone will miss it: check `ok`, then look at `status` and `result`.
</Callout>

`error` comes from a fixed vocabulary so that nothing about the failure leaks back through the model. `status` is `null` with it unless the failure happened part-way through a redirect chain, in which case it carries the status of the last hop:

| `error` | Meaning |
| --- | --- |
| `invalid_arguments` | An argument was missing, the wrong type, or too long |
| `url_not_allowed` | The URL resolved somewhere a tool may not reach |
| `timeout` | Your endpoint did not answer within `timeout_seconds` |
| `request_failed` | The request could not be completed |
| `too_many_redirects` | More than 3 redirects |

<Callout type="info">
  A test runs outside any conversation, so there is no real customer for a `context` parameter to bind to. Obvious placeholders are sent instead: `+10000000000`, `Test Contact`, `test@example.com` and a zero UUID. Expect a correctly-built order lookup to answer "not found". Check the real behaviour from a live chat.
</Callout>

## At reply time

A tool is offered to the model when all of these hold: `enabled` is `true`, `phase` is `on_call`, and the tool is either scoped to the agent handling the conversation or workspace-wide (`bot_id: null`).

Connecting the tool is half the job. The agent's instructions decide whether it uses it well, and [Connect Your Store](/docs/connect-your-store) has a prompt you can adapt.
