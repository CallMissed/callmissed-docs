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

Everything is billed in credits: **1 credit = ₹1 ≈ US$0.0104 (US$1 = ₹96).** US$ prices are the ₹ price divided by 96.

For API-specific pricing and rate limits, see the [Credits & Rate Limits](/docs/credits-rate-limits) page.

For help choosing a plan, [talk to us](/docs/talk-to-us).

### Plan Limits

Free and Starter have monthly **call caps** that are enforced server-side — LLM, STT, TTS, and image generation. Pro and above have no monthly call caps; per-minute rate limits apply instead:

| Resource | Free | Starter | Pro and above |
|----------|------|---------|---------------|
| LLM calls | 100 | 5,000 | No cap |
| STT calls | 50 | 2,500 | No cap |
| TTS calls | 50 | 2,500 | No cap |
| Image generations | 50 | 500 | No cap |

When you exceed one of these call caps on Free or Starter, the API returns a `429` error with `code: "quota_exceeded"`. Every API response includes usage headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `X-Usage-Warning` at 80% and 95% usage.

Your actual spend is always governed by your credit balance and any monthly budget cap you set — the call caps above (Free and Starter) are an additional guardrail on top of that.

#### Included allowances

Every plan also includes a storage allowance and a number of team members. Both are enforced: once your account has used its storage, new knowledge-base uploads are refused until you free space or upgrade, and once every member seat is taken, new invitations are refused until you remove a member or upgrade. Existing members and content are never removed. Conversations are not capped on any plan. Apart from these, the limits that apply are the monthly call caps above, your credit balance, your monthly budget cap, your per-key request rate, and model access.

| Allowance | Free | Starter | Pro | Enterprise and above |
|-----------|------|---------|-----|----------------------|
| Storage | 100 MB | 1 GB | 10 GB | Unlimited |
| Team members | 2 | 5 | 20 | Unlimited |

### Plans above Pro

Each tier includes everything in the tier below it. None has a monthly call quota ("No cap"); usage is metered from your credits at the same per-model rates as every other plan. On Scale, Business and Corporate the first month adds 30% to the plan price in credits, and every month after adds 5%, 10% and 15% respectively.

| Plan | Price | Credits | Requests / min | Phone calls at once | Adds |
|------|-------|---------|----------------|---------------------|------|
| Pro | ₹4,999/mo ≈ US$52.07 | 6,000 first month, then 5,000 | 3,000 | 10 | No monthly call caps |
| Enterprise | ₹20,000/mo ≈ US$208.33 | 26,000 first month, then 20,000 | 10,000 | 20 | Priority support, require 2FA for all members, session timeout policy |
| Scale | ₹50,000/mo ≈ US$520.83 | 65,000 first month, then 52,500 | 20,000 | 30 | IP allowlist, custom roles |
| Business | ₹1,00,000/mo ≈ US$1,041.67 | 1,30,000 first month, then 1,10,000 | 40,000 | Custom, [talk to sales](mailto:sales@callmissed.com) | Single sign-on (SAML / OIDC) with Google Workspace, Microsoft Entra ID or Okta, and enforced SSO; named account manager; 99.9% uptime target, tracked on our public status page |
| Corporate | ₹2,00,000/mo ≈ US$2,083.33 | 2,60,000 first month, then 2,30,000 | 80,000 | Custom, [talk to sales](mailto:sales@callmissed.com) | Everything above |

Need more than Corporate, or a bank caps a payment this large? [Talk to us](/docs/talk-to-us).
