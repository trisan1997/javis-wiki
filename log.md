# Log

Append-only chronological record. Newest at bottom. Entries start with
`## [YYYY-MM-DD] <op> | <title>` so they are greppable
(`grep "^## \[" log.md | tail -10`).

---

## [2026-05-20] bootstrap | Wiki created
Initial scaffold + 7 seed pages (3 services, 3 concepts, 1 decision) generated
from Claude Code memory files: `jarvis-startup`, `jarvis-voice-tts`,
`sara-mail-agent`, `virtueel-kantoor`. No `raw/` sources yet — pages note their
provenance as "memory" and need verification against current code before being
trusted as fact.

## [2026-05-20] query | What's the deploy gotcha?
Three things: (1) `JARVIS_NO_RELOAD=1` — no auto-reload, edits don't go live
on save; (2) requires `launchctl kickstart -k gui/$(id -u)/com.jarvis.server`
which causes brief downtime — also affects the customer offerte page in
[[habbo-hq]], so ask first; (3) launchd uses system Python **3.9** not the
`.venv` 3.12, keep service-loaded code 3.9-compatible. Sources:
[[always-on-launchd]], [[jarvis-server]], [[habbo-hq]].

## [2026-05-20] lint | Python runtime claim was wrong across 3 pages
Found during Phase-1 smoke testing: [[always-on-launchd]], [[jarvis-server]],
and [[habbo-hq]] all repeated the seed-memory claim "system Python 3.9 (not
the `.venv` 3.12)". `scripts/start.sh` actually launches `.venv/bin/python`,
which IS Python 3.9.6 (the venv is built from system `python3 -m venv`). There
is no `.venv` 3.12 anywhere in the project — the parenthetical was misleading.
All three pages corrected. Source memory `jarvis-startup` still contains the
old claim; user can decide whether to amend.

## [2026-05-20] ingest | Bootstrap of "shared brain" Phase 1
Added `agents/` and `journal/` to wiki layout (schema updated in CLAUDE.md).
Created [[jarvis]] self-page (other agent stubs deferred to next pass).
Code-side: new `src/wiki.py` bridge (read-only API, hardened path validation,
naive BM25 search), two tool schemas `read_wiki_page` + `search_wiki` added
to Jarvis-only `TOOLS` list in `src/tools.py`, `build_system_prompt("jarvis", …)`
now appends a wiki postscript (index.md + agents/jarvis.md preloaded into
Jarvis's system prompt). Pending: kickstart approval from Tristan.

## [2026-05-20] ingest | Phase 2 shipped — write side (journals) + all interactive agents
Extended shared brain to Zoë / Sara / Valentina (read + write). Added
`journal_note` tool to `_SHARED`-equivalent in all three agent tool lists,
new threading-local `active_voice` context manager in `src/tools.py`
(wrapped around the 3 tool-use loops in `src/server.py`: /api/chat, sara
sweep, _tg_run_agent), 4 new helpers in `src/wiki.py` for journal read/write/
preload with cap, and extended `_wiki_postscript` to include today's journal
in the preload. Wrote 6 missing `agents/*.md` stub pages (zoe, sara, valentina,
steve_jobs, nick, siebert). Dev-team kept out of preload (autonomy-only, no
tool-use, cost reason). Found a pre-existing dormant bug: `build_system_prompt
('valentina', ...)` raises `KeyError` because `characters.DEFAULTS` doesn't
list valentina — no current caller hits it, so non-blocking; flag for Phase 2.5.

## [2026-05-20] query | Phase 2 verified end-to-end (Sara wrote her first journal entry)
Ran `/tmp/verify_wiki_phase2.py` against the live runtime: Sara received the
8586-char system prompt with wiki+journal preload, called `journal_note` on
her first turn, wrote `journal/2026-05-20-sara.md` (252 B) with correct
frontmatter, and `active_voice` was cleared after the with-block (no leak).
First wiki entry from an agent: «phase 2 actief — Eerste live test van Phase
2 shared brain: wiki lezen en journaal schrijven werkt correct — bevestigd
door Tristan vandaag om 09:18.» Summary pushed to Telegram.

## [2026-05-20] query | Phase 1 verified end-to-end via Telegram
Ran one-shot verification (`/tmp/verify_wiki_phase1.py`) using the live MODEL,
TOOLS list, and built system prompt. Question: "Wat is de deploy gotcha …?".
Jarvis called `read_wiki_page` twice unprompted (services/jarvis-server +
concepts/always-on-launchd), then synthesised an answer citing all 7 expected
wiki facts (`JARVIS_NO_RELOAD`, `kickstart`, `.venv`, `3.9`, `launchctl`,
`offerte`, `downtime`). Crucially, he used the **corrected** Python claim
("3.9.6 from system python3"), confirming he's reading the current wiki, not
training data. Summary pushed to Tristan via `send_telegram`. Phase 1 is
functionally complete.

## [2026-05-20] ingest | Phase 3 shipped — agent-staged wiki proposals + dual approval
New `proposals/` directory. Agents call `propose_wiki_edit(slug, heading,
content, rationale)` to stage an `append_section` proposal (v1 — no replace
or delete). Target whitelist enforced server-side in `wiki._validate_proposal
_target`: services/concepts/decisions/incidents free for any agent;
`agents/<self>` self-only; everything else blocked (CLAUDE.md, index.md,
log.md, raw/, journal/, proposals/, .obsidian/, other agents' pages).
Approval has THREE channels, all applying the same atomic write:
(1) Tristan via HQ panel (`GET/POST /api/hq/wiki-proposal*`, director-token);
(2) Tristan via Telegram (`wiki apply <id>` / `wiki skip <id>`);
(3) Valentina via `approve_wiki_proposal` tool (voice-gated; cannot self-
approve — `apply_proposal` refuses if `approver == proposer == "valentina"`).
Schema + Telegram + tool routes all smoke-tested green; pre-kickstart.
