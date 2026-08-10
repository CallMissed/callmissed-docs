---
title: "Migrate from Meta Cloud API"
description: "Point an existing WhatsApp Cloud API integration at CallMissed by changing only the host and the token. Same path, same request bodies, same response and error envelopes."
slug: "migrate-from-meta"
breadcrumb: "WhatsApp"
---

# Migrate from Meta Cloud API

Point an existing WhatsApp Cloud API integration at CallMissed by changing only the host and the token. Same path, same request bodies, same response and error envelopes.

## Overview

If you already send WhatsApp messages through Meta's Cloud API (directly, or through a BSP that mirrors it), you can move to CallMissed by changing **two things**: the **host** and the **token**. The path, the request bodies, the success envelope and the error envelope are Meta's own — your existing code keeps working.

```diff
- https://graph.facebook.com/v21.0/{phone-number-id}/messages
+ https://api.callmissed.com/api/v1/whatsapp/{phone-number-id}/messages

- Authorization: Bearer EAAG...           # Meta access token
+ Authorization: Bearer cm_your_api_key   # CallMissed API key
```

Your **phone number ID** stays in the path. CallMissed maps it to your registered number and scopes everything to your tenant. API-key callers need the `whatsapp:send` scope.

> **This is a compatibility surface, not a separate product.** Everything that is a *policy* rather than a wire format — tenant isolation, credit pre-flight, template category resolution, error mapping — is the same code path as the native WhatsApp API, so behaviour and billing are identical. New integrations can use the [native WhatsApp API](/docs/whatsapp-api); this page is for moving an existing Cloud API codebase with minimal edits.

## Send a message

```
POST /api/v1/whatsapp/{phone-number-id}/messages
Authorization: Bearer cm_your_api_key
Content-Type: application/json
```

The request body is Meta's, verbatim. `messaging_product` must be `"whatsapp"`. Supported `type` values: `text`, `template`, `image`, `audio`, `video`, `document`, `sticker`, `interactive`, `location`, `contacts`, `reaction`.

:::tabs
```bash [cURL]
curl -X POST \
  https://api.callmissed.com/api/v1/whatsapp/123456789012345/messages \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "messaging_product": "whatsapp",
    "recipient_type": "individual",
    "to": "919876543210",
    "type": "text",
    "text": { "body": "Hello from CallMissed" }
  }'
```
```python [Python]
import httpx

httpx.post(
    "https://api.callmissed.com/api/v1/whatsapp/123456789012345/messages",
    headers={"Authorization": "Bearer cm_your_api_key"},
    json={
        "messaging_product": "whatsapp",
        "recipient_type": "individual",
        "to": "919876543210",
        "type": "text",
        "text": {"body": "Hello from CallMissed"},
    },
)
```
```javascript [Node.js]
await fetch(
  "https://api.callmissed.com/api/v1/whatsapp/123456789012345/messages",
  {
    method: "POST",
    headers: {
      Authorization: "Bearer cm_your_api_key",
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      messaging_product: "whatsapp",
      recipient_type: "individual",
      to: "919876543210",
      type: "text",
      text: { body: "Hello from CallMissed" },
    }),
  },
);
```
:::

### Success response

Meta's success shape, unchanged:

```json
{
  "messaging_product": "whatsapp",
  "contacts": [{ "input": "919876543210", "wa_id": "919876543210" }],
  "messages": [{ "id": "wamid.HBgL..." }]
}
```

## Two deliberate differences

Everything matches Meta except these two, and both exist so a Meta-written client keeps working correctly:

1. **A text body over 4096 characters is rejected, not split.** The native CallMissed endpoint splits a long body across several messages and returns every id in `wamids`. Meta hard-rejects instead, and a Meta-written client has no `wamids` field to read — so this surface matches Meta's rejection with error code **`100`** and the message *"Param text.body must be at most 4096 characters long."*
2. **Meta's numeric error code is preserved.** A migrated integration branches on `error.code` (e.g. `131047` → fall back to a template, `130429` → back off). So this surface returns the real code — exactly what Meta would have told you.

## Error envelope

Errors use Meta's shape (not CallMissed's usual `{"detail": "..."}`), and the HTTP status matches Meta:

```json
{
  "error": {
    "message": "(#131047) Message failed to send because more than 24 hours have passed since the customer last replied to this number.",
    "type": "OAuthException",
    "code": 131047,
    "error_data": {
      "messaging_product": "whatsapp",
      "details": "Message failed to send because more than 24 hours have passed since the customer last replied to this number."
    },
    "fbtrace_id": "A1b2C3..."
  }
}
```

The 24-hour-window error (`131047`) comes back as a real code and as HTTP `400`, so your existing "send a template instead" branch fires unchanged.

## What is unchanged on your side

- **Your seven existing `/messages/*` calls, if any, still work** and still return CallMissed's `{"detail": "..."}` shape. This compat surface is additive — it does not replace them.
- **Inbound messages and delivery statuses** still arrive through your [webhook subscriptions](/docs/whatsapp-api). This page covers sending; receiving is unchanged.
- **Templates, media, interactive, location, contacts and reactions** all take Meta's payloads for those types.

## When to use the native API instead

If you are building fresh, the [native WhatsApp API](/docs/whatsapp-api) and [Sending Messages](/docs/whatsapp-messages) give you CallMissed's own richer response shape and helpers. The Meta-compat surface exists purely to make an *existing* Cloud API integration a two-line migration.
