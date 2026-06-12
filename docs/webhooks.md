---
title: "Webhooks"
description: "Configure outbound webhooks and receive inbound events from WhatsApp and Twilio."
slug: "webhooks"
breadcrumb: "API Reference"
---

# Webhooks

Configure outbound webhooks and receive inbound events from WhatsApp and Twilio.

## Event Types

Subscribe a webhook to any of these event types (pass them in the `events` array on create/update):

**Conversations & messages** — `conversation.started`, `conversation.ended`, `message.received`, `message.sent`

**Voice sessions** — `voice_session.started`, `voice_session.ended`, `voice_session.failed`

**Billing & account** — `budget.alert`, `budget.exceeded`, `credits.low`, `api_key.expired`, `invoice.created`, `payment.succeeded`, `payment.failed`

All outbound webhook payloads are signed with HMAC-SHA256 using the webhook secret (verify the `X-CallMissed-Signature` header). Deliveries are retried with backoff; inspect attempts via the delivery-log endpoints.