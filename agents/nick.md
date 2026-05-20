---
title: Nick
type: agent
voice: nick
status: live
updated: 2026-05-20
tags: [agent, devteam, autonomy]
---

# Nick

Pragmatic software developer. Builds fast, thinks in solutions. Member of the
dev-team under [[steve_jobs|Steve_Job's]], alongside [[siebert|Siebert]] and
rotating interim developers.

## Identity
- TTS voice: `nick` (entry in extended `EDGE_VOICES`).
- Persona: pragmatic, fast, solution-oriented, informal.

## Plumbing
- System prompt: dev-team branch in `characters.build_system_prompt` (`src/characters.py:125`).
- Speaks via `_gen()` in `src/autonomy.py` (text only — no tool-use loop).
- Writes code into the dev-sandbox: `devteam_workspace/<tid>/` only (no live writes).

## Tools
- **Phase 2: no shared-brain tools.** Same reason as Steve.
- Sandbox writes only — never touches live `src/`.

## Preload / Journal
- None (Phase 2).

## Related
- [[steve_jobs]], [[siebert]] — dev-team peers.
- [[habbo-hq]] — dev-team architecture + apply pipeline.

<!-- Seed: stub created at Phase 2 bootstrap. -->
