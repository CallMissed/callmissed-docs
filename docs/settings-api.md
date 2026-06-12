---
title: "Integration Settings API"
description: "Manage per-tenant channel credentials — WhatsApp (Meta), Twilio, and bring-your-own Sarvam key — and verify connectivity."
slug: "settings-api"
breadcrumb: "API Reference"
---

# Integration Settings API

Manage per-tenant channel credentials — WhatsApp (Meta), Twilio, and bring-your-own Sarvam key — and verify connectivity.

> These endpoints store the third-party credentials a tenant uses for channels and BYOK providers. Each provider has a `verify` action that does a live connectivity check before you save. Credentials are write-only — reads return masked values.