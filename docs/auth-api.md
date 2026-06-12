---
title: "Authentication API"
description: "Full REST reference for account auth — register, login, OAuth, OTP, session management, two-factor (TOTP), and passkeys."
slug: "auth-api"
breadcrumb: "API Reference"
---

# Authentication API

Full REST reference for account auth — register, login, OAuth, OTP, session management, two-factor (TOTP), and passkeys.

> These endpoints power the dashboard's auth flows. They use **JWT** (access + refresh tokens), not `cm_` API keys. Access tokens last 24h; refresh tokens last 7d and rotate on use. Auth endpoints are rate-limited per IP (5–10 req/min).