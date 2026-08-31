---
title: "Messaging"
description: "Read Messenger and Instagram DM threads and reply as the Page or account."
slug: "social-messaging"
breadcrumb: "Facebook & Instagram"
---

# Messaging

Read Messenger and Instagram DM threads and reply as the Page or account.

Read your Facebook Messenger and Instagram Direct inbox threads and reply into
them. Reading needs `whatsapp:read`; sending needs `whatsapp:send`. Messaging on
both channels is **not metered**.

No account id goes in the path anywhere on this page. Reading is scoped to your
whole workspace, and a send names its account in the **body** — with a value you
already hold, because it comes off the thread you are replying to.

One rule shapes this whole surface: **a conversation must be started by the
customer.** You can never message someone who has not messaged your Page or
account first — there is no thread to reply into, and the send is refused with a
`409`.

## The messaging window

Meta only lets a business reply for a limited time after the customer's last
message. CallMissed enforces this **before** any upstream call, measured from the
thread's last inbound message:

| Time since last inbound | What happens |
|---|---|
| Within 24 hours | The reply sends normally. |
| 24 hours – 7 days | The reply sends as a **human-agent** tagged message. This uses Meta's Human Agent feature, which needs its own App Review approval — until that's granted, Meta may still reject the tagged send (surfaced as `502`). |
| Over 7 days | The reply is refused locally with `400`. The customer must message again to reopen the window. |

## List inbox threads

Facebook — `GET /api/v1/facebook/conversations`
Instagram — `GET /api/v1/instagram/conversations`

The most recent 200 threads, newest first. There is no pagination and no cursor —
this is a live inbox view, not an archive export. Message bodies are **not**
included; fetch them per thread from the messages endpoint.

Threads are returned for **every** connected account on the channel, so a
workspace with two Pages sees both inboxes in one call. `account_ref` tells you
which account each thread belongs to, and it is the value you send back when you
reply.

```json
[
  {
    "id": "7f3c1e64-2b8a-4d19-9c02-5a1b6e4f8d20",
    "channel": "facebook",
    "account_ref": "102938475610293",
    "contact_id": "6284719203847561",
    "contact_name": "Priya Sharma",
    "last_inbound_at": "2026-08-24T09:14:02.481930+00:00",
    "last_message_at": "2026-08-24T09:15:37.902144+00:00"
  }
]
```

`contact_id` is the recipient id you send to (a Page-scoped id on Facebook, an
Instagram-scoped id on Instagram). `contact_name` can be `null`.

## List messages in a thread

Facebook — `GET /api/v1/facebook/conversations/{conversation_id}/messages`
Instagram — `GET /api/v1/instagram/conversations/{conversation_id}/messages`

The messages in one thread, oldest first (up to 500). `conversation_id` is the
`id` from the thread list.

```json
[
  {
    "id": "3a5e77c1-9b40-4f8e-a2d6-11c8ee4b7f39",
    "channel": "facebook",
    "direction": "inbound",
    "external_id": "m_AbCdEf1234567890",
    "message_type": "text",
    "text": "Hi, is the store open tomorrow?",
    "status": null,
    "created_at": "2026-08-24T09:14:02.481930+00:00"
  },
  {
    "id": "9c02b418-77de-4a55-b6f1-2d9e0a3c8b7e",
    "channel": "facebook",
    "direction": "outbound",
    "external_id": "m_ZyXwVu0987654321",
    "message_type": "text",
    "text": "Yes, we are open 10am to 8pm tomorrow.",
    "status": "sent",
    "created_at": "2026-08-24T09:15:37.902144+00:00"
  }
]
```

An unknown `conversation_id`, or one from another workspace, returns an **empty
array** (`[]`), not a `404` — so `[]` can mean "no such thread" as well as "no
messages yet". `status` is `null` on inbound messages. `message_type` is `text`,
an attachment type, `reaction` (Instagram), or `postback` (a button tap).

## Send a message

Facebook — `POST /api/v1/facebook/messages`
Instagram — `POST /api/v1/instagram/messages`

A text reply into an existing thread. All three fields are required, and all three
come straight off the thread you are replying to — there is no `?account=` hint
here because the body already carries the account.

**Facebook**

| Field | Type | Description |
|---|---|---|
| `page_id` | string | The connected Page's Meta id — the thread's `account_ref`. |
| `recipient_id` | string | The recipient's id — the thread's `contact_id`. |
| `text` | string | The message text. |

**Instagram**

| Field | Type | Description |
|---|---|---|
| `ig_user_id` | string | The connected Instagram account's Meta id — the thread's `account_ref`. |
| `recipient_id` | string | The recipient's id — the thread's `contact_id`. |
| `text` | string | The message text. |

An `account_ref` that is not connected to your workspace returns `404`, so a send
is authorized against your own accounts rather than trusted from the request.

```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/instagram/messages \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{ "ig_user_id": "17841400000000000", "recipient_id": "1234567890123456", "text": "Yes, we deliver across Pune in 2 days." }'
```

On Instagram a long message is split on word boundaries into multiple sends (the
per-message limit is 1,000 bytes of UTF-8), and the response reports how many
went out. Only the parts Meta actually accepted are recorded, so a partial send is
visible rather than reported as whole:

```json
{ "status": "sent", "sent": 2, "message_ids": ["aWdfZG1fMTo...QUFB", "aWdfZG1fMTo...QkJC"] }
```

Facebook returns a single `message_id` (which can be `null` if Meta omitted it):

```json
{ "status": "sent", "message_id": "m_ZyXwVu0987654321" }
```

Text only — there is no attachment, template or quick-reply parameter on these
endpoints. A send fails with `409` when no thread exists for the recipient (the
window was never opened), `400` when the window has closed, `404` when the account
is not one of yours, and `502` when the upstream send fails.
