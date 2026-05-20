---
title: Siebert
type: agent
voice: siebert
status: live
updated: 2026-05-20
tags: [agent, devteam, autonomy]
---

# Siebert

Thorough software developer — pays attention to details, tests, and
maintainability. Member of the dev-team under [[steve_jobs|Steve_Job's]],
alongside [[nick|Nick]] and rotating interims.

## Identity
- TTS voice: `siebert` (entry in extended `EDGE_VOICES`).
- Persona: thorough, detail-focused, test-minded, collegial.

## Plumbing
- System prompt: dev-team branch in `characters.build_system_prompt` (`src/characters.py:125`).
- Speaks via `_gen()` in `src/autonomy.py` (text only — no tool-use loop).
- Sandbox-only writes.

## Tools
- **Phase 2: no shared-brain tools.** Same reason as Steve.

## Preload / Journal
- None (Phase 2).

## Related
- [[steve_jobs]], [[nick]] — dev-team peers.
- [[habbo-hq]] — architecture context.

<!-- Seed: stub created at Phase 2 bootstrap. -->
