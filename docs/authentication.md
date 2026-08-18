---
title: "Authentication"
description: "Every API request authenticates with a CallMissed API key."
slug: "authentication"
breadcrumb: "API Reference"
---

# Authentication

Every API request authenticates with a CallMissed API key.

## API Key

Every request authenticates with an API key. Create one from your Profile page in the dashboard — keys are prefixed with `cm_` and are passed as a Bearer token:

```
Authorization: Bearer cm_your_api_key_here
```

API keys never expire but can be revoked at any time.

### Anthropic SDK (x-api-key header)

When using the [Anthropic-compatible endpoint](/docs/anthropic-api) (`/v1/messages`), you can also authenticate with the `x-api-key` header:

```
x-api-key: cm_your_api_key_here
```

Both header styles work on the Anthropic endpoint — use whichever your SDK sends by default.

## Permissions vs. scopes

API keys carry two independent access controls.

### Service permissions

Permissions decide which AI services a key may call. They are enforced on the inference endpoints — a key without the matching permission gets `403 permission_denied`. Set any combination of `llm`, `stt`, `tts`, `search`, `image`, or `*` for all (the default for new keys).

| Permission | Gates |
|------------|-------|
| `llm` | `/v1/chat/completions`, `/v1/messages` (+ `/anthropic/v1/messages`) |
| `stt` | `/v1/audio/transcriptions`, `/v1/audio/translations` |
| `tts` | `/v1/audio/speech` |
| `search` | `/v1/search` |
| `image` | `/v1/images/generations` |

### Resource scopes

Scopes gate the resource endpoints under `/api/v1/` — bots, conversations, knowledge, webhooks. Unlike permissions, scopes default to **empty = no resource access** — you opt in explicitly.

| Scope | Gates |
|-------|-------|
| `bots:read` / `bots:write` | View vs. create/update/delete bots |
| `conversations:read` / `conversations:write` | View vs. update conversations |
| `knowledge:read` / `knowledge:write` | View vs. add/remove knowledge entries |
| `webhooks:write` | Manage outbound webhook subscriptions |
| `whatsapp:read` / `whatsapp:write` | Read vs. manage WhatsApp messaging |
| `*` | All resource scopes |

#### Gateway scopes

| Scope | Gates |
|-------|-------|
| `usage:read` | [Usage summaries, logs and CSV export](/docs/usage-api) |
| `prompts:read` / `prompts:write` | [Stored prompts, versions, labels and presets](/docs/gateway-prompts). Rendering counts as a read |
| `cache:read` / `cache:write` | [Cache statistics vs. purging](/docs/gateway-cache) |
| `provider_keys:read` / `provider_keys:write` | [Your own provider credentials](/docs/provider-keys) |

#### CRM scopes

| Scope | Gates |
|-------|-------|
| `companies:read` / `companies:write` | [Companies](/docs/crm-companies) |
| `crm_notes:read` / `crm_notes:write` | [Notes](/docs/crm-notes-tasks) |
| `crm_tasks:read` / `crm_tasks:write` | [Tasks](/docs/crm-notes-tasks) |
| `crm_deals:read` / `crm_deals:write` | [Deals **and** pipelines](/docs/crm-deals) — one pair covers both |
| `crm_timeline:read` | [The activity timeline](/docs/crm-lead-scores#timeline). Read-only, no write half |
| `crm_custom_fields:read` / `crm_custom_fields:write` | [Custom field definitions and values](/docs/crm-custom-fields) |
| `crm_views:read` / `crm_views:write` | [Saved views](/docs/crm-custom-fields#saved-views) |
| `crm_search:read` / `crm_search:write` | [Search and duplicates](/docs/crm-import-export) vs. merging |
| `crm_bulk:write` | [Bulk update and delete](/docs/crm-import-export#bulk-operations). Write-only, no read half |
| `crm_csv:read` / `crm_csv:write` | [CSV export vs. import](/docs/crm-import-export#csv) |
| `crm_scores:read` / `crm_scores:write` | [Lead scoring rules and recompute](/docs/crm-lead-scores) |

#### Support desk scopes

| Scope | Gates |
|-------|-------|
| `support_tickets:read` / `support_tickets:write` | [Tickets](/docs/support-tickets) |
| `sla:read` / `sla:write` | [SLA policies, ticket clocks and breaches](/docs/support-sla) |
| `support_ops:read` / `support_ops:write` | [Macros, tags and routing rules](/docs/support-ops) |
| `csat:read` / `csat:write` | [Surveys and results](/docs/csat). The customer's response page needs no credential at all |

#### Commerce and voice-agent scopes

| Scope | Gates |
|-------|-------|
| `wa_commerce:read` / `wa_commerce:write` | [WhatsApp orders](/docs/whatsapp-orders) |
| `wa_flows:read` / `wa_flows:write` | [WhatsApp Flows](/docs/whatsapp-flows) |
| `evals:read` / `evals:write` | [Eval suites, cases and runs](/docs/voice-evals) |
| `experiments:read` / `experiments:write` | [A/B experiments](/docs/voice-experiments). Assignment needs write |
| `squads:read` / `squads:write` | [Agent squads](/docs/voice-squads). Handoff simulation needs only read |

Watch for the scopes whose read and write halves do not line up with the HTTP verb. `POST /api/v1/gateway/prompts/{id}/render`, `POST /api/v1/support/ops/routing-rules/evaluate` and `POST /api/v1/voice/squads/{id}/simulate-handoff` are all `POST` requests that write nothing, so they need only the **read** scope.

> A key created for plain inference (the common case) needs only service permissions — leave scopes empty.
