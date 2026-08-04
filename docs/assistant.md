---
title: "AI Assistant (Copilot)"
description: "A chat assistant docked in the dashboard that can operate CallMissed for you — create bots and voice agents, manage WhatsApp, knowledge bases, keys, webhooks, and your team."
slug: "assistant"
breadcrumb: "LLM & AI"
---

# AI Assistant (Copilot)

A chat assistant docked in the dashboard that can operate CallMissed for you — create bots and voice agents, manage WhatsApp, knowledge bases, keys, webhooks, and your team.

## Overview

The **assistant** is a chat panel built into the dashboard that can operate CallMissed on your behalf. Instead of clicking through settings pages, describe what you want in plain language and it does it — create a bot, wire up a knowledge base, invite a teammate, roll an API key, and more. It's aware of the page you're on, so it can give page-specific help too.

## What It Can Do

The assistant can read and manage every major dashboard domain:

| Domain | Examples |
| --- | --- |
| Bots & voice agents | Create, update, toggle, and verify deployment for bots and voice agents |
| Knowledge bases | Add text or file sources, list, and remove knowledge entries |
| Conversations | Look up conversations and messages, change status, toggle autoreply |
| WhatsApp | Manage templates and campaigns, send messages, check calling settings |
| API keys | Create, scope, and revoke keys |
| Webhooks | Create, test, and inspect webhook deliveries |
| Team | Invite members, change roles, review team settings |
| Account & billing | Credit balance, usage, invoices, integrations (read-only) |

For anything outside those tools it falls back to a read-only lookup, so it can still answer account questions without guessing.

## Confirm Before Action

Reads run immediately. Anything that changes your account — creating, updating, or deleting data — always stops and asks first. The assistant proposes the action as a confirm card describing exactly what it's about to do, and nothing happens until you approve it. Each proposal is single-use and time-limited, so a stale confirm card can't be replayed later.

## Billing

Assistant replies use your account's credits at standard model rates — the same credit pool as every other AI call on the platform. There's no separate assistant plan or quota; it draws from your existing balance and is subject to your plan's usual limits.

## Role Behavior

- **Owner / admin** — full control: every tool, including all mutating actions across bots, knowledge, keys, webhooks, WhatsApp, and team.
- **Agent** — read access everywhere, plus the conversation actions agents already handle day to day (updating conversation status, toggling autoreply). Account- and team-level changes are not available.

## Opening It

Press **⌘J** (Ctrl+J on Windows/Linux) from anywhere in the dashboard to open the assistant panel, or open it from the sidebar. It's available on every dashboard page, and there's also a dedicated full-page view for longer sessions.
