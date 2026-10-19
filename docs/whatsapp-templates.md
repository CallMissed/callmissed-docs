---
title: "Message Templates"
description: "Create, edit, unpause, list, delete and sync WhatsApp message templates, start from Meta's template library, upload header media, and draft templates with AI."
slug: "whatsapp-templates"
breadcrumb: "WhatsApp"
---

# Message Templates

Create, edit, unpause, list, delete and sync WhatsApp message templates, start from Meta's template library, upload header media, and draft templates with AI.

A message template is pre-approved copy you can send **outside** the 24-hour customer service window. Order updates, delivery notices, reminders and one-time codes are all template sends. Templates are created on WhatsApp, reviewed by Meta, and mirrored locally so you can list and filter them without a Meta round trip.

All endpoints are under `https://api.callmissed.com/api/v1/whatsapp`.

## Lifecycle

:::flow
icon:gateway | Create | `POST /templates` validates the copy locally, then submits it to WhatsApp
icon:llm | Review | Meta reviews it. The template sits at `PENDING`
icon:done | Approved | A status webhook flips it to `APPROVED` and it becomes sendable
:::

Statuses you will see: `PENDING`, `APPROVED`, `REJECTED`, `PAUSED`, `DISABLED`, `IN_APPEAL`. Only `APPROVED` templates can be sent. A rejected template carries a `rejection_reason`. To change copy, [edit](#edit-a-template) the template rather than deleting and recreating it, and [unpause](#unpause-a-template) a paused one.

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
| `language` | string, 2 to 12 chars | Yes | Locale, for example `en_US`, `hi`, `es_MX`. Must be one of Meta's supported template languages, otherwise `400` |
| `components` | array of objects, at least 1 | One of | Header, body, footer and button spec. Must include a `BODY`. Omit when using `library_template_name` |
| `library_template_name` | string, max 512 | One of | Create from a [Template Library](#browse-the-template-library) preset instead of `components`. Sending both, or neither, returns `422` |
| `library_template_body_inputs` | object | No | Library path only. Body opt-ins such as `add_contact_number`, `add_learn_more_link`, `add_security_recommendation`, `add_track_package_link`, `code_expiration_minutes` |
| `library_template_button_inputs` | array of objects | No | Library path only. Values the preset cannot know, such as your URL, your phone number or the OTP type |
| `parameter_format` | string | No | `positional` (`{{1}}`) or `named` (`{{customer_name}}`), case-insensitive. Omit to infer it from the placeholders. Ignored on the library path |
| `allow_category_change` | boolean | No | Let Meta re-categorise the template. Defaults to on, so only send this to opt out. Ignored on the library path |

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

`components` is forwarded to WhatsApp unchanged, so any component type WhatsApp supports works, including button blocks. A media header (`IMAGE`, `VIDEO`, `DOCUMENT`) needs a sample in `example.header_handle`, which you get from [upload header media](#upload-header-media). `BODY`, `FOOTER`, text `HEADER` and the three marketing formats below ([carousel](#carousel-templates), [limited-time offer](#limited-time-offer-templates), [coupon code](#coupon-code-templates)) are checked locally first; everything else is validated by Meta.

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
    "parameter_format": "positional",
    "sub_category": null,
    "correct_category": null,
    "previous_category": null,
    "components": [],
    "last_meta_synced_at": null,
    "created_at": "2026-04-19T12:00:00Z",
    "updated_at": "2026-04-19T12:00:00Z"
  },
  "policy_warnings": []
}
```

| Field | Type | Notes |
|---|---|---|
| `template_id` | string, nullable | Meta's template id |
| `status` | string | Initial lifecycle state, typically `PENDING` |
| `template` | object | The mirrored row, described in [the template object](#the-template-object) |
| `policy_warnings` | array of `{code, message}` | Advisory risks the rules check found, for example a likely re-categorisation. They never block the create |

The local row is written only after WhatsApp accepts the create, so a rejection leaves nothing behind.

### Starting from the Template Library

Meta's Template Library holds pre-written utility and authentication templates that are approved faster. Find a preset with [`GET /templates/library`](#browse-the-template-library), then pass its name instead of `components`:

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "name": "acme_order_confirmation",
    "category": "UTILITY",
    "language": "en_US",
    "library_template_name": "order_confirmation_1",
    "library_template_button_inputs": [
      { "type": "URL", "url": { "base_url": "https://acme.example.com/orders/{{1}}", "url_suffix_example": "https://acme.example.com/orders/AC-10294" } }
    ]
  }'
```

The preset owns the body, header, footer, variable format and category, so `parameter_format` and `allow_category_change` do not apply. The response has the same shape as a hand-built create.

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
| `{{1}}` and `{{name}}` variables in one template | `A template can't mix {{1}} (positional) and {{name}} (named) variables -- pick one format for the whole template.` |
| `parameter_format` disagrees with the placeholders | `parameter_format='named' but the template uses positional variables. Change one to match the other.` |

`example.body_text` is an **array of arrays**: one inner array holding a sample value per variable. A text `HEADER` with variables uses `example.header_text`, a flat array.

Named templates use a different example shape: `example.body_text_named_params` (or `header_text_named_params`) is a flat array of `{ "param_name": "customer_name", "example": "Priya" }` objects, one per distinct variable, and the names must match the placeholders exactly.

After these structural checks, a rules check looks for anything Meta states it will reject, such as a body that starts or ends with a variable or a footer with a variable. A template that breaks one returns `422` listing every problem: `This template breaks WhatsApp's template rules and would be rejected. Fix these first: (1) ...`. Softer risks come back as `policy_warnings` on a successful create.

### Errors from WhatsApp

Every template route that reaches WhatsApp maps its error to a short, stable message rather than echoing Meta's text:

| Code | Meaning |
|---|---|
| `400` | WhatsApp rejected the request shape, for example a missing sample value |
| `401` | The stored WhatsApp business token is invalid or expired. Reconnect the number |
| `404` | The WABA is not on your workspace (`WhatsApp account not found`), or the template no longer exists on WhatsApp |
| `409` | A template with this name and language already exists, the WABA is disconnected, the WABA has no active phone number, or the template is paused for low quality |
| `422` | Any other WhatsApp rejection: `WhatsApp rejected this template request. Check the details and try again.` |
| `503` | WhatsApp itself failed. Retry |

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
    "parameter_format": "positional",
    "sub_category": null,
    "correct_category": null,
    "previous_category": null,
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
| `parameter_format` | string, nullable | `positional` or `named` |
| `sub_category` | string, nullable | Meta's sub-classification, when it sends one |
| `correct_category` | string, nullable | The category Meta believes the template belongs in. Worth surfacing: a `UTILITY` to `MARKETING` move changes the price of every send. Null when Meta agrees with yours |
| `previous_category` | string, nullable | The category before Meta re-categorised it |
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

## Edit a template

`PATCH /api/v1/whatsapp/templates/{template_uuid}` · scope `whatsapp:write`

Also served as `POST /api/v1/whatsapp/templates/{template_uuid}/edit` for clients that cannot send `PATCH`. Same body, same response.

Editing keeps the template's name, so it avoids the 30-day name lock a delete-and-recreate costs. `name` and `language` are never editable.

| Field | Type | Required | Notes |
|---|---|---|---|
| `components` | array of objects | One of | A **full replacement** list. WhatsApp replaces every component, it does not merge. Validated exactly like a create |
| `category` | string | One of | `MARKETING`, `UTILITY` or `AUTHENTICATION`. WhatsApp rejects a category change on an `APPROVED` template |
| `parameter_format` | string | No | `positional` or `named`. Selects which variable syntax the new components are checked against |

```bash
curl -X PATCH https://api.callmissed.com/api/v1/whatsapp/templates/3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "components": [
      {
        "type": "BODY",
        "text": "Hi {{1}}, order {{2}} has shipped and should arrive within 3 days.",
        "example": { "body_text": [["Priya", "AC-10294"]] }
      },
      { "type": "FOOTER", "text": "Acme Coffee" }
    ]
  }'
```

**Response (200 OK)**: the [template object](#the-template-object) with your new components, plus `policy_warnings`. The status is not changed here: WhatsApp re-reviews the edit and the result arrives by status webhook (or the next [sync](#sync-from-whatsapp)).

| Code | Meaning |
|---|---|
| `400` | Neither `components` nor `category` was sent, or the components failed validation |
| `404` | The template is not on your workspace |
| `409` | The template is `PENDING` (only `APPROVED`, `REJECTED` and `PAUSED` templates can be edited), or WhatsApp never confirmed it. Sync, then retry |
| `422` | The new components break a template rule, or WhatsApp refused the edit |

WhatsApp limits edits to an `APPROVED` template to 10 in 30 days and 1 in 24 hours. `REJECTED` and `PAUSED` templates can be edited without limit. WhatsApp enforces this count, so an edit over the limit comes back as a WhatsApp rejection.

## Unpause a template

`POST /api/v1/whatsapp/templates/{template_uuid}/unpause` · scope `whatsapp:write`

Lifts a pause on a template. A pause caused by low quality expires on its own, but a template paused by WhatsApp's template pacing stays paused until it is unpaused, either here or in WhatsApp Manager. No body.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates/3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e/unpause \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "template_id": "1234567890123456",
  "unpaused": true,
  "template": { "id": "3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e", "status": "PAUSED", "name": "order_shipped" }
}
```

| Field | Type | Notes |
|---|---|---|
| `template_id` | string | Meta's template id |
| `unpaused` | boolean | `true` when WhatsApp accepted the request |
| `template` | object | The [template object](#the-template-object) as stored when you asked. Its `status` is not rewritten here: the new status arrives by webhook or [sync](#sync-from-whatsapp) |

`404` if the template is not on your workspace, `409` if WhatsApp never confirmed it.

## Delete a template

`DELETE /api/v1/whatsapp/templates/{template_uuid}` · scope `whatsapp:write`

Deletes on WhatsApp and drops the local row. Returns `204 No Content` with an empty body.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/whatsapp/templates/3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e \
  -H "Authorization: Bearer cm_your_api_key"
```

If the template was already deleted in WhatsApp Manager, the local row is cleaned up anyway. A template that never got a Meta id is simply dropped locally.

> **Deleting an approved template starts a 30-day cooldown** before the same **name** can be reused. Reusing it sooner fails at create time.

### Delete every language of a template

`DELETE /api/v1/whatsapp/templates/{template_uuid}/languages` · scope `whatsapp:write`

Deletes the template's **name** on WhatsApp, which removes every language variant of it, not only the one `{template_uuid}` points at. Use the single delete above to remove one language.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/whatsapp/templates/3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e/languages \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "name": "order_shipped",
  "deleted_local_rows": 3,
  "deleted_languages": ["en_US", "es_MX", "hi"],
  "name_locked_days": 30
}
```

| Field | Type | Notes |
|---|---|---|
| `name` | string | The template name that was deleted |
| `deleted_local_rows` | integer | Rows removed, one per language |
| `deleted_languages` | array of strings | Every language variant that went |
| `name_locked_days` | integer | Days before this name can be created again |

### Delete in bulk

`POST /api/v1/whatsapp/templates/bulk_delete` · scope `whatsapp:write`

Deletes up to 100 templates in one WhatsApp call.

| Field | Type | Required | Notes |
|---|---|---|---|
| `template_uuids` | array of UUIDs, 1 to 100 | Yes | CallMissed template `id`s. All must belong to the same WABA |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates/bulk_delete \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "template_uuids": ["3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e", "8b2c4d6e-1a3f-4c5d-9e7f-0a1b2c3d4e5f"] }'
```

**Response (200 OK)**

```json
{
  "requested": 2,
  "deleted": 2,
  "meta_template_ids": ["1234567890123456", "1234567890123457"],
  "skipped_local_only": 0
}
```

| Field | Type | Notes |
|---|---|---|
| `requested` | integer | Ids you sent |
| `deleted` | integer | Rows removed |
| `meta_template_ids` | array of strings | Meta ids sent to WhatsApp's bulk delete |
| `skipped_local_only` | integer | Templates WhatsApp never confirmed, removed locally without a WhatsApp call |

The batch is **all or nothing**. One unknown id returns `404` and nothing is deleted; templates from more than one WABA return `400`; a WhatsApp rejection leaves every template in place.

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

## Browse the Template Library

`GET /api/v1/whatsapp/templates/library` · scope `whatsapp:read`

Searches Meta's Template Library of pre-written utility and authentication templates. Pass a WABA (`account_id` or `waba_id`) so the search runs with your credentials; the library itself is the same for every account. Create from a result by passing its `name` as [`library_template_name`](#starting-from-the-template-library).

| Param | Type | Default | Notes |
|---|---|---|---|
| `account_id` / `waba_id` | UUID / string, max 64 | none | One is required |
| `search` | string, max 256 | none | Matches template content, name, header, body or footer |
| `topic` | string | none | For example `ACCOUNT_UPDATE`, `CUSTOMER_FEEDBACK`, `ORDER_MANAGEMENT`, `PAYMENTS` |
| `usecase` | string, max 64 | none | For example `ORDER_CONFIRMATION`. Passed to WhatsApp as-is |
| `industry` | string | none | For example `E_COMMERCE`, `FINANCIAL_SERVICES` |
| `language` | string, max 12 | none | Locale filter |
| `name` | string, max 512 | none | Exact preset name |
| `limit` | integer, 1 to 100 | WhatsApp's default | Page size |
| `after` | string, max 512 | none | Cursor from the previous page's `paging` |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/templates/library?waba_id=102290129340398&topic=ORDER_MANAGEMENT&language=en_US" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "data": [
    {
      "name": "order_confirmation_1",
      "language": "en_US",
      "category": "UTILITY",
      "topic": "ORDER_MANAGEMENT",
      "usecase": "ORDER_CONFIRMATION",
      "body": "Hi {{1}}, your order {{2}} is confirmed. We will let you know when it ships.",
      "body_params": ["Priya", "AC-10294"],
      "buttons": [{ "type": "URL", "text": "View order" }]
    }
  ],
  "paging": null
}
```

`data` holds WhatsApp's library objects as returned, so fields can vary by preset. Library placeholders are positional: bind `body_params` by index. `paging` carries a cursor when there are more results.

## Check business verification

`GET /api/v1/whatsapp/templates/business_verification` · scope `whatsapp:read`

Whether the Meta business that owns a WABA is verified. Some template features and higher messaging limits need a verified business.

| Param | Type | Notes |
|---|---|---|
| `account_id` / `waba_id` | UUID / string, max 64 | One is required |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/templates/business_verification?waba_id=102290129340398" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "waba_id": "102290129340398", "status": "verified", "verified": true }
```

| Field | Type | Notes |
|---|---|---|
| `waba_id` | string | The WABA checked |
| `status` | string, nullable | WhatsApp's verification status, for example `verified`, `pending`, `not_verified`. Null when it could not be read |
| `verified` | boolean, nullable | Null when `status` is unknown |

The result is cached for a few minutes. A failed lookup returns `200` with `status: null` rather than an error; an unknown WABA still returns `404`.

## Upload header media

`POST /api/v1/whatsapp/media/resumable` · scope `whatsapp:write`

A template with an `IMAGE`, `VIDEO` or `DOCUMENT` header (including every carousel card) must carry a sample file in `example.header_handle`. This endpoint uploads the sample to WhatsApp and returns that handle. It is not the same as [`POST /media`](/docs/whatsapp-messages#upload-media), which returns a `media_id` for sending messages; a `media_id` is not accepted as a header handle.

Send the file as `multipart/form-data` in a field named `file`. Accepted types: `image/jpeg`, `image/jpg`, `image/png`, `video/mp4`, `application/pdf`. This path accepts larger bodies than the rest of the API so a real video sample fits.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/media/resumable \
  -H "Authorization: Bearer cm_your_api_key" \
  -F "file=@dark-roast.jpg;type=image/jpeg"
```

**Response (200 OK)**

```json
{ "handle": "4::aW1hZ2UvanBlZw==:ARZ1", "file_type": "image/jpeg", "size_bytes": 184233 }
```

Put `handle` as the single element of the header's `example.header_handle` array.

| Code | Meaning |
|---|---|
| `400` | Missing or unsupported `Content-Type`, or an empty file |
| `413` | The file is over the upload size limit |
| `422` | WhatsApp rejected the upload. Check the file type and size |

## Draft a template with AI

`POST /api/v1/whatsapp/ai/draft_template` · scope `whatsapp:write`

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
  "compliance_notes": "Transactional reminder tied to an existing subscription, so UTILITY is the correct category.",
  "policy_review": { "errors": [], "warnings": [], "tips": [], "score": 100 }
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
| `policy_review` | object, nullable | The same rules check `POST /templates` runs, applied to the draft: `{ errors, warnings, tips, score }`, each finding a `{code, message}` and `score` from 0 to 100. Any `errors` here would make the create return `422` |

`422` when the model cannot produce a valid draft. Shorten or clarify the intent and retry.
