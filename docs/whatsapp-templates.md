---
title: "Message Templates"
description: "Create, list, delete and sync WhatsApp message templates, including authentication templates and the AI drafting endpoint."
slug: "whatsapp-templates"
breadcrumb: "WhatsApp"
---

# Message Templates

Create, list, delete and sync WhatsApp message templates, including authentication templates and the AI drafting endpoint.

A message template is pre-approved copy you can send **outside** the 24-hour customer service window. Order updates, delivery notices, reminders and one-time codes are all template sends. Templates are created on WhatsApp, reviewed by Meta, and mirrored locally so you can list and filter them without a Meta round trip.

All endpoints are under `https://api.callmissed.com/api/v1/whatsapp`.

## Lifecycle

:::flow
icon:gateway | Create | `POST /templates` validates the copy locally, then submits it to WhatsApp
icon:llm | Review | Meta reviews it. The template sits at `PENDING`
icon:done | Approved | A status webhook flips it to `APPROVED` and it becomes sendable
:::

Statuses you will see: `PENDING`, `APPROVED`, `REJECTED`, `PAUSED`, `DISABLED`, `IN_APPEAL`. Only `APPROVED` templates can be sent. A rejected template carries a `rejection_reason`.

## Choosing the WABA

Template endpoints act on a WhatsApp Business Account rather than a phone number. Supply **exactly one**:

| Field | Type | Where it comes from |
|---|---|---|
| `account_id` | UUID | The `id` from `GET /accounts` |
| `waba_id` | string, max 64 | Meta's WABA id |

Omitting both returns `400` with `"Either account_id (UUID) or waba_id (Meta) is required"`. On `GET /templates` these are optional filters instead.

## Create a template

`POST /api/v1/whatsapp/templates` · scope `whatsapp:write`

| Field | Type | Required | Notes |
|---|---|---|---|
| `account_id` / `waba_id` | UUID / string | One of | The WABA to create under |
| `name` | string, 1 to 512 chars | Yes | Must match `^[a-z0-9_]+$`: lowercase letters, digits and underscores only |
| `category` | string | Yes | `MARKETING`, `UTILITY` or `AUTHENTICATION` |
| `language` | string, 2 to 12 chars | Yes | Locale, for example `en_US`, `hi`, `es_MX` |
| `components` | array of objects, at least 1 | Yes | Header, body, footer and button spec. Must include a `BODY` |
| `parameter_format` | string | No | `POSITIONAL` or `NAMED`, selecting the variable syntax |
| `allow_category_change` | boolean | No | Let Meta re-categorise the template. Defaults to on, so only send this to opt out |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "name": "order_shipped",
    "category": "UTILITY",
    "language": "en_US",
    "components": [
      { "type": "HEADER", "format": "TEXT", "text": "Your order is on its way" },
      {
        "type": "BODY",
        "text": "Hi {{1}}, order {{2}} shipped today and should arrive in 2 to 3 days.",
        "example": { "body_text": [["Priya", "AC-10294"]] }
      },
      { "type": "FOOTER", "text": "Acme Coffee" }
    ]
  }'
```

`components` is forwarded to WhatsApp unchanged, so any component type WhatsApp supports works, including button blocks. `BODY`, `FOOTER`, text `HEADER` and the three marketing formats below ([carousel](#carousel-templates), [limited-time offer](#limited-time-offer-templates), [coupon code](#coupon-code-templates)) are checked locally first; everything else is validated by Meta.

**Response (200 OK)**

```json
{
  "template_id": "1234567890123456",
  "status": "PENDING",
  "template": {
    "id": "3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e",
    "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
    "template_id": "1234567890123456",
    "name": "order_shipped",
    "language": "en_US",
    "category": "UTILITY",
    "status": "PENDING",
    "quality_score": "UNKNOWN",
    "rejection_reason": null,
    "components": [],
    "last_meta_synced_at": null,
    "created_at": "2026-04-19T12:00:00Z",
    "updated_at": "2026-04-19T12:00:00Z"
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `template_id` | string, nullable | Meta's template id |
| `status` | string | Initial lifecycle state, typically `PENDING` |
| `template` | object | The mirrored row, described in [the template object](#the-template-object) |

The local row is written only after WhatsApp accepts the create, so a rejection leaves nothing behind.

### Validation before submission

Copy is checked locally first, so a guaranteed rejection does not cost a Meta round trip. Each of these returns `400` with the reason:

| Rule | Message you get |
|---|---|
| Name outside `^[a-z0-9_]+$` | `template name must match ^[a-z0-9_]+$ (lowercase letters, digits, and underscores only)` |
| No `BODY` component | `A BODY component is required.` |
| Empty `BODY` text | `The BODY component requires non-empty text.` |
| `BODY` over 1024 characters | `BODY text exceeds 1024 characters.` |
| `FOOTER` over 60 characters | `FOOTER text exceeds 60 characters.` |
| `{{N}}` variables with no example | `A component with {{N}} variables requires an 'example.body_text' array.` |
| Example count does not match the variable count | `The example provides 1 value(s) but the text has 2 {{N}} variable(s).` |
| An `example` on a component with no variables | `Omit the 'example' object on a component with no {{N}} variables -- Meta rejects an empty example.` |

`example.body_text` is an **array of arrays**: one inner array holding a sample value per variable. A text `HEADER` with variables uses `example.header_text`, a flat array.

### Authentication templates

One-time-code templates have a different body shape. Meta owns the verification copy, so you must **not** send `BODY` text:

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "name": "acme_login_code",
    "category": "AUTHENTICATION",
    "language": "en_US",
    "components": [
      { "type": "BODY", "add_security_recommendation": true }
    ]
  }'
```

The only body option is the boolean `add_security_recommendation`, and there is no `example` because there is no sender-supplied variable in the body. Sending `BODY` text on an `AUTHENTICATION` template returns `400` telling you the verification-code copy is fixed by Meta, and a non-boolean `add_security_recommendation` returns `400` as well. Any additional button or footer options come from Meta's authentication-template reference and are passed through unchanged.

Once approved, send the code through [`POST /messages/template`](/docs/whatsapp-messages#send-a-template-message), passing it as the body or button parameter.

### Carousel templates

A carousel pairs a normal message `BODY` with a swipeable row of cards, each with its own media header and buttons. Add a `CAROUSEL` component alongside the `BODY`.

Carousels are **`MARKETING` only**. Under any other category the create returns `400` naming the format.

| Rule | Detail |
|---|---|
| `cards` | 2 to 10. The count is fixed at creation: an approved template can only send the number of cards it was created with |
| Card `HEADER` | Required on every card, and always media. `format` is `IMAGE` or `VIDEO` |
| Card header media | `example.header_handle` must be a non-empty array holding an uploaded media handle |
| Card `BODY` | Optional, but if one card has it every card must. Text max 160 characters, far shorter than the 1024-character message body. Variables need an `example` object |
| Card `BUTTONS` | Optional, at most 2 per card, of type `QUICK_REPLY`, `URL` or `PHONE_NUMBER` |
| Uniform structure | Every card must carry the same components in the same order, and the same button types. Cards render at a shared height, so a body or button on one card is required on all |
| Top-level `BODY` | Still required, alongside the `CAROUSEL` component |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "name": "summer_blends_carousel",
    "category": "MARKETING",
    "language": "en_US",
    "components": [
      {
        "type": "BODY",
        "text": "Hi {{1}}, our cold brew blends are 20% off this week.",
        "example": { "body_text": [["Priya"]] }
      },
      {
        "type": "CAROUSEL",
        "cards": [
          {
            "components": [
              {
                "type": "HEADER",
                "format": "IMAGE",
                "example": { "header_handle": ["4::aW1hZ2UvanBlZw==:ARZ1"] }
              },
              { "type": "BODY", "text": "Ratnagiri Dark Roast, notes of cocoa and dried fig." },
              {
                "type": "BUTTONS",
                "buttons": [
                  { "type": "QUICK_REPLY", "text": "Send me a sample" },
                  { "type": "URL", "text": "Shop now", "url": "https://acme.example.com/dark-roast" }
                ]
              }
            ]
          },
          {
            "components": [
              {
                "type": "HEADER",
                "format": "IMAGE",
                "example": { "header_handle": ["4::aW1hZ2UvanBlZw==:ARZ2"] }
              },
              { "type": "BODY", "text": "Chikmagalur Medium Roast, bright and citrus-forward." },
              {
                "type": "BUTTONS",
                "buttons": [
                  { "type": "QUICK_REPLY", "text": "Send me a sample" },
                  { "type": "URL", "text": "Shop now", "url": "https://acme.example.com/medium-roast" }
                ]
              }
            ]
          }
        ]
      }
    ]
  }'
```

Every rule above is checked before submission, and the `400` names the card index and the field, so you do not have to reverse-engineer a generic rejection.

### Limited-time offer templates

A limited-time offer adds an offer banner with an optional countdown. Add a `LIMITED_TIME_OFFER` component. `MARKETING` only.

| Rule | Detail |
|---|---|
| `limited_time_offer` | Required object: `{ "text": string (max 16), "has_expiration": boolean }`. `text` is the offer label |
| `BODY` | Max 600 characters on this format, stricter than the usual 1024 |
| `HEADER` | Optional, but when present must be `IMAGE` or `VIDEO`. A text header is not supported |
| `FOOTER` | Not supported at all. Sending one returns `400` |
| `BUTTONS` | Only `COPY_CODE` and `URL`. When both are present the `COPY_CODE` button must be declared first, because it is fixed at button index 0 and the URL button at index 1 |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "name": "monsoon_offer",
    "category": "MARKETING",
    "language": "en_US",
    "components": [
      {
        "type": "HEADER",
        "format": "IMAGE",
        "example": { "header_handle": ["4::aW1hZ2UvanBlZw==:ARZ1"] }
      },
      {
        "type": "BODY",
        "text": "Hi {{1}}, take 20% off your next bag of coffee.",
        "example": { "body_text": [["Priya"]] }
      },
      {
        "type": "LIMITED_TIME_OFFER",
        "limited_time_offer": { "text": "20% off", "has_expiration": true }
      },
      {
        "type": "BUTTONS",
        "buttons": [
          { "type": "COPY_CODE", "example": "MONSOON20" },
          { "type": "URL", "text": "Shop now", "url": "https://acme.example.com/shop" }
        ]
      }
    ]
  }'
```

`has_expiration: true` renders a countdown, whose expiry is supplied per send as a component parameter on [`POST /messages/template`](/docs/whatsapp-messages#send-a-template-message). `components` is forwarded to WhatsApp unchanged on a template send, so the parameter shape is WhatsApp's own.

### Coupon code templates

A `COPY_CODE` button gives the customer a one-tap copy of a discount code. It works on its own marketing template, and is also the button an LTO template uses. `MARKETING` only.

| Rule | Detail |
|---|---|
| Button shape | `{ "type": "COPY_CODE", "example": "<CODE>" }`. The button's label is fixed, so there is no `text` to set |
| `example` | Required, a sample coupon code, max 20 characters. The same cap applies to the code you pass at send time |
| Count | At most one `COPY_CODE` button per template |
| Companions | A `QUICK_REPLY` button may accompany it |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "name": "welcome_coupon",
    "category": "MARKETING",
    "language": "en_US",
    "components": [
      {
        "type": "BODY",
        "text": "Welcome to Acme, {{1}}. Here is 15% off your first order.",
        "example": { "body_text": [["Priya"]] }
      },
      {
        "type": "BUTTONS",
        "buttons": [
          { "type": "COPY_CODE", "example": "WELCOME15" },
          { "type": "QUICK_REPLY", "text": "Browse blends" }
        ]
      }
    ]
  }'
```

The `example` is a sample for review, not the code you ship. The real code goes in a `coupon_code` button parameter per send, capped at the same 20 characters, so one approved template can issue a different code to every customer.

An `AUTHENTICATION` template's `{ "type": "OTP", "otp_type": "COPY_CODE" }` button is a different component on a different template family and is not subject to these rules.

## List templates

`GET /api/v1/whatsapp/templates` · scope `whatsapp:read`

Reads the local mirror, newest updated first. All parameters are optional filters.

| Param | Type | Default | Notes |
|---|---|---|---|
| `account_id` | UUID | none | Filter to one connected account |
| `waba_id` | string, max 64 | none | Filter by Meta WABA id |
| `status` | string | none | `APPROVED`, `PENDING`, `REJECTED`, `PAUSED`, `DISABLED`, `IN_APPEAL`. Case-insensitive |
| `category` | string | none | `MARKETING`, `UTILITY` or `AUTHENTICATION`. An unknown value returns `400` |
| `language` | string, max 12 | none | Filter by locale |
| `limit` | integer, 1 to 500 | 100 | Max rows |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/templates?status=APPROVED&limit=100" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
[
  {
    "id": "3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e",
    "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
    "template_id": "1234567890123456",
    "name": "order_shipped",
    "language": "en_US",
    "category": "UTILITY",
    "status": "APPROVED",
    "quality_score": "GREEN",
    "rejection_reason": null,
    "components": [
      { "type": "BODY", "text": "Hi {{1}}, order {{2}} shipped today and should arrive in 2 to 3 days." },
      { "type": "FOOTER", "text": "Acme Coffee" }
    ],
    "last_meta_synced_at": "2026-04-19T13:02:44Z",
    "created_at": "2026-04-19T12:00:00Z",
    "updated_at": "2026-04-19T13:02:44Z"
  }
]
```

The mirror is kept current by status webhooks and an hourly reconciliation sweep. For up-to-the-second consistency, call [sync](#sync-from-whatsapp) first.

### The template object

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | CallMissed's id. Use it on get and delete |
| `account_id` | UUID | The owning WABA |
| `template_id` | string, nullable | Meta's template id. Null if Meta never confirmed the create |
| `name` | string | Template name |
| `language` | string | Locale |
| `category` | string | `MARKETING`, `UTILITY` or `AUTHENTICATION` |
| `status` | string | Lifecycle state |
| `quality_score` | string | Meta's quality signal, for example `GREEN` or `UNKNOWN` |
| `rejection_reason` | string, nullable | Why Meta rejected it |
| `components` | array of objects | The approved component spec |
| `last_meta_synced_at` | datetime, nullable | Last reconciliation against Meta |
| `created_at` / `updated_at` | datetime | ISO 8601 UTC |

## Get one template

`GET /api/v1/whatsapp/templates/{template_uuid}` · scope `whatsapp:read`

`{template_uuid}` is the `id` field, not Meta's `template_id`. Returns the template object, or `404` if it is not on your workspace.

```bash
curl https://api.callmissed.com/api/v1/whatsapp/templates/3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e \
  -H "Authorization: Bearer cm_your_api_key"
```

## Delete a template

`DELETE /api/v1/whatsapp/templates/{template_uuid}` · scope `whatsapp:write`

Deletes on WhatsApp and drops the local row. Returns `204 No Content` with an empty body.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/whatsapp/templates/3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e \
  -H "Authorization: Bearer cm_your_api_key"
```

If the template was already deleted in WhatsApp Manager, the local row is cleaned up anyway. A template that never got a Meta id is simply dropped locally.

> **Deleting an approved template starts a 30-day cooldown** before the same **name** can be reused. Reusing it sooner fails at create time.

## Sync from WhatsApp

`POST /api/v1/whatsapp/templates/sync` · scope `whatsapp:write`

Pulls every template for a WABA from WhatsApp and upserts the local mirror. An hourly sweep does this automatically, so call it when you have just edited templates in WhatsApp Manager and want them reflected immediately.

| Field | Type | Required | Notes |
|---|---|---|---|
| `account_id` / `waba_id` | UUID / string | One of | The WABA to reconcile |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates/sync \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "waba_id": "102290129340398" }'
```

**Response (200 OK)**

```json
{ "waba_id": "102290129340398", "fetched": 12, "inserted": 2, "updated": 10 }
```

| Field | Type | Notes |
|---|---|---|
| `waba_id` | string | The WABA that was reconciled |
| `fetched` | integer | Templates WhatsApp returned |
| `inserted` | integer | New local rows |
| `updated` | integer | Existing rows refreshed |

Templates WhatsApp no longer returns are not deleted by this call. The background sweep owns that.

## Draft a template with AI

`POST /api/v1/whatsapp/ai/draft_template` · scope `whatsapp:read`

Turns a plain-language intent into a Meta-compliant draft, with an approval-risk assessment. It is read-only: nothing is submitted to WhatsApp, so review the draft and then post it to [create](#create-a-template) yourself. The generation is billed to your workspace.

| Field | Type | Required | Notes |
|---|---|---|---|
| `intent` | string, 10 to 1000 chars | Yes | What the template should say and when it is sent |
| `language` | string, 2 to 8 chars | No | Default `en` |
| `category` | string | No | Force `UTILITY`, `MARKETING` or `AUTHENTICATION`. Omit to let the model choose |
| `emojis` | boolean | No | Allow emojis in the body. Default `false`, which is safest for approval |
| `include_header` | boolean | No | Allow a header line. Default `true` |
| `include_footer` | boolean | No | Allow a footer line. Default `true` |
| `tone` | string, max 40 | No | For example `friendly`, `formal`, `concise` |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/ai/draft_template \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "intent": "Tell a customer their coffee subscription renews in three days and they can skip or change the blend before then.",
    "language": "en",
    "category": "UTILITY",
    "tone": "friendly"
  }'
```

**Response (200 OK)**

```json
{
  "name": "subscription_renewal_reminder",
  "category": "UTILITY",
  "language": "en",
  "body": "Hi {{1}}, your Acme coffee subscription renews on {{2}}. Reply SKIP to pause this delivery or CHANGE to pick a different blend.",
  "header_text": "Your subscription renews soon",
  "footer_text": "Acme Coffee",
  "components": [
    { "type": "HEADER", "format": "TEXT", "text": "Your subscription renews soon" },
    {
      "type": "BODY",
      "text": "Hi {{1}}, your Acme coffee subscription renews on {{2}}. Reply SKIP to pause this delivery or CHANGE to pick a different blend.",
      "example": { "body_text": [["Priya", "22 April"]] }
    },
    { "type": "FOOTER", "text": "Acme Coffee" }
  ],
  "approval_risk": "low",
  "rejection_risks": [],
  "compliance_notes": "Transactional reminder tied to an existing subscription, so UTILITY is the correct category."
}
```

| Field | Type | Notes |
|---|---|---|
| `name` | string | Suggested template name, already matching Meta's naming rules |
| `category` | string | The category chosen or forced |
| `language` | string | Echoes the requested locale |
| `body` / `header_text` / `footer_text` | string, nullable for header and footer | The drafted copy |
| `components` | array of objects | Ready to post to `POST /templates` as-is |
| `approval_risk` | string | The model's read on how likely Meta is to approve it |
| `rejection_risks` | array of strings | Specific things that could get it rejected. Empty when none were found |
| `compliance_notes` | string, nullable | Why the category and wording were chosen |

`422` when the model cannot produce a valid draft. Shorten or clarify the intent and retry.
