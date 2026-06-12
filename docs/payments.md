---
title: "Payments & Invoices"
description: "Top up credits or upgrade your plan via Cashfree, download GST invoices, redeem coupons, and set a monthly credit budget cap."
slug: "payments"
breadcrumb: "API Reference"
---

# Payments & Invoices

Top up credits or upgrade your plan via Cashfree, download GST invoices, redeem coupons, and set a monthly credit budget cap.

## Overview

Payments are processed through **Cashfree**. Use `create-order` to start a plan upgrade or a credit top-up, then complete checkout with the Cashfree SDK using the returned `payment_session_id`. The signed Cashfree **webhook is the source of truth** for settlement — never grant credits on the client-side redirect alone. Successful orders generate a downloadable GST invoice. **1 credit = ₹1.**

> These endpoints authenticate with your **dashboard session (JWT)**, not a `cm_` API key. Creating orders, upgrading plans, and managing billing are owner/admin actions and are not exposed to API keys.

## Payments

Start a credit top-up of 5,000 credits (₹5,000):

```bash
curl https://api.callmissed.com/api/v1/payments/create-order \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"type": "credit_topup", "credits": 5000}'
```

```json
{
  "order_id": "cm_9f2a1c4e6b8d0a13",
  "cf_order_id": "CF...",
  "payment_session_id": "session_...",
  "amount": 5000.0,
  "currency": "INR",
  "env": "production"
}
```

For a plan upgrade send `{"type": "plan_upgrade", "plan": "pro"}` instead. Pass `payment_session_id` to the Cashfree checkout SDK; assert the SDK mode matches the returned `env`.