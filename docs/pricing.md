---
title: "Pricing"
description: "Simple, transparent pricing. Pay only for what you use."
slug: "pricing"
breadcrumb: "Getting Started"
---

# Pricing

Simple, transparent pricing. Pay only for what you use.

## Overview

Visit our [Pricing Page](https://callmissed.com/pricing) for detailed plan comparisons and per-API pricing.

For API-specific pricing and rate limits, see the [Credits & Rate Limits](/docs/credits-rate-limits) page.

For enterprise pricing, [talk to us](/docs/talk-to-us).

### Plan Limits

Each plan tier has monthly usage caps enforced server-side:

| Resource | Free | Starter | Pro | Enterprise |
|----------|------|---------|-----|------------|
| LLM calls | 100 | 5,000 | 50,000 | Unlimited |
| STT calls | 50 | 2,500 | 25,000 | Unlimited |
| TTS calls | 50 | 2,500 | 25,000 | Unlimited |
| Conversations | 50 | 1,000 | 10,000 | Unlimited |
| Storage | 100 MB | 1 GB | 10 GB | Unlimited |
| Team members | 2 | 5 | 20 | Unlimited |

When you exceed your plan limit, the API returns a `429` error with `code: "quota_exceeded"`. Every API response includes usage headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `X-Usage-Warning` at 80% and 95% usage.