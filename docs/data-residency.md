---
title: "Data Residency & Retention"
description: "Where CallMissed stores your messages and your recipients' phone numbers and email addresses, how long it keeps them, and how to get a data processing agreement."
slug: "data-residency"
breadcrumb: "Resources"
---

# Data Residency & Retention

Where CallMissed stores your messages and your recipients' phone numbers and email addresses, how long it keeps them, and how to get a data processing agreement.

## Where your data is stored

CallMissed runs its production systems on Microsoft Azure data centres in India (Central India region). The data you send through the API is stored there, and that includes:

- **WhatsApp:** message content, recipient phone numbers, and each message's delivery status and timestamps.
- **Email:** send records (sender, recipients, subject, status and timestamps), per-recipient delivery outcomes, inbound mail and suppressions.
- **Records:** contacts, conversations, call recordings and transcripts, and webhook delivery logs.

Encrypted backups of production data are also kept within India.

## When data leaves India

Some processing happens outside India because of how a channel works or which features you turn on:

- **WhatsApp.** Messages travel over the WhatsApp Business Platform, which Meta operates. While a message is in transit, and for whatever Meta keeps on its side, Meta's own terms apply. When you register a number you can choose where Meta stores its data at rest with [`data_localization_region`](/docs/whatsapp-numbers), for example `"IN"`.
- **Email.** Outbound mail is handed to an email delivery partner, which carries it to your recipients' mail servers. Inbound mail reaches us the same way in reverse.
- **AI features.** When you use a model, speech or voice feature, the content involved can go to an AI service partner that processes it outside India, and only to produce the output you asked for. For LLM requests, [zero data retention](/docs/gateway-controls) limits a request to routes where the provider keeps no prompt or response.

We send personal data only to countries the Government of India has not restricted, and only under a contract with each partner, as Section 16 of the Digital Personal Data Protection Act, 2023 requires.

## How long we keep it

| Data | How long we keep it |
| --- | --- |
| WhatsApp messages, including content, phone numbers, delivery statuses with timestamps, and error codes | For the life of your account, unless you delete them first |
| WhatsApp campaign recipients and their per-recipient statuses | For the life of your account |
| Raw WhatsApp webhook events | For the life of your account |
| Email send records, including recipients, subject, status, timestamps, acceptance confirmation and per-recipient delivery outcomes | For the life of your account. The rendered message body is **not** kept after it has been handed off for delivery |
| Inbound email | The parsed message is kept for the life of your account. The raw MIME is not kept after parsing |
| [Webhook](/docs/webhooks) delivery logs, API usage logs and audit logs | For the life of your account by default. Account owners can set a shorter retention period in the dashboard (at least 30 days) |
| [Email webhook](/docs/email-webhooks) delivery logs | For the life of your account |
| LLM request logs on an API key | Deleted after 30 days |
| Batch input and output files | 30 days ([Batch API](/docs/batch)) |

Delivery statuses and their timestamps are kept as long as the message they belong to. That makes them usable as a long-term proof-of-service record.

## Deleting your data

- **Individual records.** You can delete contacts, agent memories and other records you own through their own endpoints, or from the dashboard.
- **Closing your account.** When an account owner deactivates the account, you have a 30-day window to reactivate it. After that, the account and all of its data are permanently deleted, including WhatsApp messages, campaign recipients, email send records, inbound mail and webhook logs.
- **Backups.** Encrypted backup copies are purged on a rolling schedule, ordinarily within 90 days of deletion from primary systems.

An address that has asked never to be emailed again stays on our unsubscribe list after the account is deleted, so that the request keeps being honoured.

## Data processing agreement

Our standard [Data Processing Agreement](https://www.callmissed.com/dpa-agreement) covers processing on your behalf, data residency, security, breach notification and deletion when the agreement ends. It is designed to align with the Digital Personal Data Protection Act, 2023.

To request a countersigned copy, the current sub-processor schedule or details of cross-border transfers, email **karan@callmissed.com**.
