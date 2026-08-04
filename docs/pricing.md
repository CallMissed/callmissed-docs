---
title: "Pricing"
description: "Simple, transparent pricing. Pay only for what you use."
slug: "pricing"
breadcrumb: "Resources"
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
| LLM calls | 100 | 5,000 | 50,000 | No cap |
| STT calls | 50 | 2,500 | 25,000 | No cap |
| TTS calls | 50 | 2,500 | 25,000 | No cap |
| Image generations | 50 | 500 | 5,000 | No cap |
| Conversations | 50 | 1,000 | 10,000 | No cap |
| Storage | 100 MB | 1 GB | 10 GB | Unlimited |
| Team members | 2 | 5 | 20 | Unlimited |

**Enterprise ($200/mo)** grants 26,000 bonus credits/month, the highest rate limit (10,000 req/min), and priority support. It has no monthly call quota ("No cap") — usage is metered pay-as-you-go from your credits at the same per-model rates as every other plan. Need more than Enterprise? [Talk to us](/docs/talk-to-us) for a custom volume deal.

When you exceed your plan limit, the API returns a `429` error with `code: "quota_exceeded"`. Every API response includes usage headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `X-Usage-Warning` at 80% and 95% usage.
