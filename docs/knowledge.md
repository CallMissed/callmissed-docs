---
title: "Knowledge Base & RAG"
description: "Store content your bots use to answer questions, plus a tenant-wide vector Knowledge API for semantic (RAG) search."
slug: "knowledge"
breadcrumb: "API Reference"
---

# Knowledge Base & RAG

Store content your bots use to answer questions, plus a tenant-wide vector Knowledge API for semantic (RAG) search.

## Overview

There are two layers of knowledge:

1. **Bot knowledge base** — entries scoped to a single bot, passed as context to that bot's LLM. Add plain text or upload PDF/DOCX/TXT (max 20 MB); text is extracted automatically.
2. **Knowledge API (RAG)** — a tenant-wide vector store. Ingest text, URLs, or PDFs as **sources**, then run semantic search to retrieve the most relevant passages for retrieval-augmented generation.

> The Knowledge API requires the `knowledge:read` / `knowledge:write` scopes on your key.

## Bot Knowledge Base