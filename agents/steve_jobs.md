---
title: Steve_Job's
type: agent
voice: steve_jobs
status: live
updated: 2026-05-20
tags: [agent, devteam, autonomy]
---

# Steve_Job's

Visionary lead developer of the dev-team. Plans changesets, hands work off to
[[nick|Nick]] and [[siebert|Siebert]], and runs the apply pipeline. Demanding,
design-driven, decisive about quality.

## Identity
- TTS voice: `steve_jobs` (entry in extended `EDGE_VOICES`).
- Persona: visionary, design-gedreven, makes calls and guards quality.
- Reports to: [[valentina|Valentina]] (waarnemend directeur).

## Plumbing
- System prompt: dev-team branch in `characters.build_system_prompt` (`src/characters.py:125`).
- Driven via `src/autonomy.py` (`_steve_dev` ~every 40 min), `src/devteam.py` (sandbox writes + apply pipeline).
- Runs LLM text generation via `_gen()` — **no tool-use loop**, so no dynamic tool calls during whispers.

## Tools
- **Phase 2: no shared-brain tools.** Dev-team agents talk via `_gen()` (text only) so they can't call `read_wiki_page` / `journal_note`. Wiki preload also off (kept lean for cost).
- Apply pipeline: `devteam.apply(tid)` → director-approved via Telegram (`apply <id>` / `skip <id>` in chat 7855958540).

## Preload
- None (Phase 2). May be added in Phase 3 if dev-team gets its own tool-use loop.

## Journal
- Not applicable (no tool access). Steve's work is logged via `logs/autonomy.log` + Telegram changeset notifications.

## Related
- [[habbo-hq]] — autonomy + dev-team architecture.
- [[nick]], [[siebert]] — co-developers.
- [[valentina]] — coordinator he reports to.

<!-- Seed: stub created at Phase 2 bootstrap. -->
