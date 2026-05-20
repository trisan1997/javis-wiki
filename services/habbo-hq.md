---
title: Virtual Office (Habbo HQ)
type: service
status: live
updated: 2026-05-20
sources: 0
tags: [office, hq, autonomy, devteam, frontend]
---

# Virtual Office (Habbo HQ)

Habbo-style virtual office at `/` (formerly `/hq`) where Tristan can see all
agents walking around and collaborating. Cartoon look, procedural canvas (no
assets). Started **2026-05-17**.

## Architecture

- `/` = office (`office.html`). Old funnel moved to `/classic` (instant revert: set `root()` back).
- Roles: `tristan` = code-login, talks to all AIs / `klant` (customer) = account.
- Customer flow (7a–7d **done**): one-time register (voornaam/naam/adres/email/gsm + password, pbkdf2) → menu: offer (Jarvis, prefilled), reference photos (Valentina, placeholder), on-site visit (Valentina).
- Customer accounts: `config/customers.json`. Offer counter: `config/quote_counter.json` (starts 41/year).
- Offer name format: `<adres> · <jaar> · <nr>` in `create_and_send_quote`.

## Autonomy (Fase 2 — live)

- `src/autonomy.py` — background thread; real LLM inter-agent whispers via `office.whisper()` + solo work-beats; rate-limited; `AUTONOMY_ENABLED=0` to disable; log `logs/autonomy.log`.
- Agents learn from each other via `characters.evolve()` after every conversation.
- Beats every 12–35s. Heavy `_meeting` (~24%) + `_exchange` (~38%).
- `_meeting()` — 2–4 agents walk to VERGADERZAAL (16-seat table) and back.
- `_boss_rounds()` — Tristan-sprite walks past offices (canned, no LLM).
- Interim agents commute visibly between devteam ↔ interns area.
- `_learn_tick()` — every tick a different agent evolves; everyone learns through the day.
- `_ask_tristan()` — Valentina sends a Telegram improvement-question every ~30–60 min (first ~10–20 min). Answers route through existing `_tg_poller` ([[jarvis-server]]).

## Dev-team (Fase 3 — live, safety-verified)

- `src/devteam.py`. Steve_Job's plans; Nick, Siebert + interims write code **only** to `devteam_workspace/<tid>/`. Padguard. No shell/exec/auto-apply/restart.
- Endpoint: `/api/hq/devtask`. UI: click dev-teamlid → opdracht insturen + status-poll.
- E2E tested: live `src/` provably untouched.
- Apply pipeline: `devteam.apply(tid)` copies sandbox→live with backup, path validation, **blocks `.env`/`certs`/`.git`/secrets**. No shell, no restart.
- Endpoint: `/api/hq/devtask/apply` (director-token only).
- `autonomy._steve_dev()` ~every 40 min — autonomously creates a changeset and pings Tristan via Telegram with the task-id.
- `_tg_poller` recognises **"apply &lt;id&gt;"** / **"skip &lt;id&gt;"** (Tristan's chat = authorization) → `devteam.apply`. UI button not built yet (Telegram path works).

## Director / proposals (7e — live)

- Director token from `/api/hq/auth`; `/api/hq/devtask` + `/api/hq/proposal(s)` server-side behind `is_director(token)`.
- Idea system: agents (autonomy `_idea()`) → `office.add_proposal` → director approves/rejects in panel bottom-right.
- Valentina coordinator: agents report problems to her (`autonomy._report_issue` → `_issues`); ~8 dev meetings/day (`_dev_meeting`); she feeds evaluation back via `office.add_proposal`.

## Rendering

- Realistic thin-wall look; walls 4× tall (back `WB=104` / front `WF=16`); opaque; **fade when you enter the room** (`world.active`).
- Animated full-height doors. No windows, no logos. Per-room wall/floor tint follows agent colour (room JSON can set `accent`, e.g. interim office yellow).
- A* pathfinding (`office._route` / `_astar` / `_walkable`, `_GB`) — agents only via doors/corridors. 0 wall-passes (tested).
- Per-agent TTS voices (server `EDGE_VOICES` extended for `steve_jobs` / `nick` / `siebert` / `tristan` / `valentina` / `interim` — see [[voice-tts]]).
- 🔊 toggle plays new bubbles audibly. 🎵 toggle plays procedural soft background music (Web Audio, no asset).
- Browser blocks non-gesture audio → `office_app.js` unlocks 1 reused `<audio>` on first click (capture).
- **Depth-render reverted** — global depth sort broke walls. Back on working `world.drawStatic` + separate `agents.draw` (2 layers). Occlusion (agents hidden behind walls) = open for a separate safe attempt later. `collect` / `drawFloors` / `drawSigns` + `agents.collect/drawBubbles` kept as dead code.

## Calendar — on-site visits

- `plan_onsite_quote_visit`: only Tue–Thu **after 18:00** (slots 18/19/20h). Auto-shift on conflict. Books to calendar **"offerte"** (`OFFERTE_CALENDAR_NAME`). Checks free/busy against `CALENDAR_NAME` ("Tristan afspraken") + offerte.
- Valentina prompt forbids `add_zoe_calendar_event` for this.

## Deploy gotcha

Live server runs via launchd (`com.jarvis.server`, `JARVIS_NO_RELOAD=1`, **`.venv/bin/python`** = Python 3.9.6). Code changes only go live after `launchctl kickstart -k gui/$(id -u)/com.jarvis.server` — brief downtime affects the customer offerte page too, so **ask Tristan first**. Keep `src/office.py` 3.9-compatible.

## Related

- [[jarvis-server]] — the server hosting this.
- [[always-on-launchd]] — deploy/restart pattern.
- [[voice-tts]] — per-agent voices.

<!-- Seed source: Claude Code memory `virtueel-kantoor` (2026-05-18 snapshot). Verify against current code. -->
