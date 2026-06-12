# callmissed-docs

Public documentation for the [CallMissed API](https://docs.callmissed.com).
Auto-synced from the main monorepo ? do not edit files in `docs/` manually.

## Context7

Library ID: `/callmissed/callmissed-docs`

Use in AI coding assistants:
```
use context7 /callmissed/callmissed-docs
```

## Structure

```
docs/
  README.md          # Navigation index (auto-generated)
  introduction.md
  quickstart.md
  models.md
  chat-completion.md
  ... (one .md file per doc page)
```

## How it works

Changes to `apps/docs/src/lib/pages.ts` in the private monorepo automatically
open a PR here via the `sync-docs-to-context7` workflow. Merging triggers a
Context7 re-index.