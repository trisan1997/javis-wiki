---
title: Valentina
type: agent
voice: valentina
status: live
updated: 2026-05-20
tags: [agent, coordinator, voice]
---

# Valentina

The **waarnemend directeur** — second-in-command after Tristan. Coordinates
every other agent (Jarvis, Sara, the dev-team, interims). Hidden from
customers; only Tristan and other agents address her.

## Identity
- TTS voice: `valentina` (entry in extended `EDGE_VOICES`, voice-id TBD — verify in `src/server.py:103`).
- Persona: assertive, empathic but realistic, progressive-pragmatic.
- Language: ALWAYS Dutch, no exceptions.
- Shares her system prompt with [[zoe|Zoë]] (`else` branch in `characters.build_system_prompt`).

## Plumbing
- System prompt: `characters.build_system_prompt`'s `else` branch (`src/characters.py:154+`).
- Tool list (when called via `/api/chat`): `tools.TOOLS` (the Jarvis list — `req.voice == "valentina"` falls to the else branch in the tool-list dispatch).
  - **Quirk**: Valentina uses Jarvis-flavored tools, not Zoë's. May or may not be intentional — flag for review.
- Heavy autonomy presence: `_dev_meeting`, `_idea`, `_report_issue`, `_ask_tristan` in `src/autonomy.py` use her prompt.

## Tools
- Shared-brain (Phase 2): `read_wiki_page`, `search_wiki`, `journal_note`.
- Office/orchestration: `office.add_proposal`, plus the broader Jarvis tool set.
- Periodically pings Tristan via Telegram with improvement questions.

## Preload (hybrid context)
- [[index]] + this page + today's `journal/<date>-valentina.md`.

## Journal
- Path: `journal/YYYY-MM-DD-valentina.md`. Use for: coordination decisions, dev-meeting outcomes, escalations to Tristan, observed agent issues, proposals approved/rejected.

## Related
- [[zoe]] — shares prompt.
- [[habbo-hq]] — orchestrates the office.
- [[jarvis-server]] — hosting process.

<!-- Seed: stub created at Phase 2 bootstrap. -->
