# journal/

Append-only daily journals, one file per agent per day.

## Filename convention

`YYYY-MM-DD-<voice>.md`, e.g. `2026-05-20-jarvis.md`.

## What goes here

- Agent's notable observations, decisions, problems encountered.
- Inter-agent messages (e.g. Valentina → Steve_Job's coordination).
- User interactions worth remembering across restarts (replaces today's lost-on-restart in-memory `tg-<voice>` histories).

## What does NOT go here

- Every chat token. Journal entries should be **distilled** — a few lines per meaningful event, not raw transcripts.
- Secrets, customer PII (real customer names, addresses, etc.). Redact before writing.

## Format (suggested)

```markdown
---
agent: jarvis
date: 2026-05-20
---

## [HH:MM] <one-line headline>
Body (1–5 lines). Reference wiki pages with [[wikilinks]] when applicable.
```

## Lifecycle

- Phase 2 (Journal writes — pending) — agents append here via the `journal_note` tool.
- Periodic distillation (Phase 4 — optional) — older journal entries get summarised into curated wiki pages, and the originals can be archived.

Until Phase 2 ships, this directory is empty by design.
