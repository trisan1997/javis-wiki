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

## [2026-05-20] ingest | Phase 4 shipped — journal distillatie (loop closure)
Nieuwe Valentina-only tool `distill_recent_journals(window_hours=48,
max_proposals=3)`. Sluit de loop: Phase 2-journals accumuleren → Phase 4
distilleert patronen → Phase 3 zet ze als voorstellen in de queue.

Implementatie: `wiki.collect_recent_journals(window, cap=60KB)` leest alle
agent-journals binnen het venster (tail-truncatie bij grote corpora).
Dispatch promptet Anthropic met de huidige wiki-index + corpus, parseert
JSON-output ({proposals: [{target, heading, content, rationale}]}), valideert
elk voorstel (whitelist services/concepts/decisions/incidents + existence-
check — v1 is append-only, dus geen new_page voorstellen). Stage'd via
`wiki.stage_proposal` zoals een gewone propose_wiki_edit aanroep.

On-demand only in v1 — geen cron-worker. Smoke-test toonde dat de tool
correct "geen patronen" returnt bij thin journals (één Sara-entry), maar
ook 2 echte voorstellen produceert wanneer er materiaal is. Bewust voice-
gated naar Valentina (Jarvis/Zoë/Sara zien de tool niet).

## [2026-05-20] ingest | Phase 3.5 shipped — Tristan + Valentina commanderen het dev-team (met interruptie)
Single-task-at-a-time model: nieuwe `devteam.submit()` cancelt eerst alle nog-
lopende taken via coöperatieve cancellation (per-task `threading.Event` +
`_check_cancel(tid)` op 3 punten in `_run()`: start, na plan, vóór elke file).
Geannuleerde taken krijgen `status="geannuleerd"`. + `list_recent`, `current_id`,
`cancel_current` helpers.

**Tristan via Telegram** (chat 7855958540): `dev: <opdracht>` → submit
(onderbreekt lopende), `dev list`, `dev status [<id>]`, `dev stop`.
`apply <id>` blijft exclusief Tristan (downtime affect customer offerte-pagina).

**Valentina via LLM-tools** (voice-gated; Jarvis/Zoë/Sara zien ze niet):
`submit_dev_task`, `list_dev_tasks`, `get_dev_task_status`, `cancel_current_dev_task`.
Nieuwe lijst `tools.VALENTINA_TOOLS = TOOLS + _DEVTEAM_VALENTINA_TOOLS`. Chat-
dispatch en `_tg_run_agent` dispatchen `voice == "valentina"` → VALENTINA_TOOLS.

**Voice-routing fix**: `_tg_voice_for("valentina")` returnt nu echt `"valentina"`
(was `"zoe"`). Telegram-histories ook gescheiden (`tg-valentina` vs `tg-zoe`).
Label-map split: `zoe → "Zoë"`, `valentina → "Valentina"`. Heeft als
neveneffect dat Valentina/Zoë nu echt twee aparte agents zijn in Telegram
(zelfde prompt-body via characters.py else-branch, maar verschillende voice +
journal + tools). Zie [[habbo-hq]] sectie "Commando-pijplijn".

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

## [2026-05-20] decision | Optie B gekozen voor publieke architectuur
Tristan koos Optie B uit [[2026-05-20-public-website-architecture]]: split
frontend op one.com + Tailscale Funnel API. Status verschoven van planned
naar decided. Implementatie staat ingepland voor Prompt 4 (publieke
offerte + facturatie). Drie deliverables: CORS-middleware, `tailscale funnel
--bg 443` enablen, statisch frontend bouwen + FTP-deploy.

## [2026-05-20] ingest | Prompt 2 shipped — leren uit fouten (flag_mistake + correctie-capture)
Sluit de leerlus voor fouten. Drie samenhangende stukken:

(1) **Expliciet**: nieuwe tool `flag_mistake(what_i_did, what_was_wrong,
lesson)` beschikbaar voor alle interactieve agents. Slaat op in journal
als 'MISTAKE: ...' regel met gestructureerde body.

(2) **Impliciet**: `_maybe_capture_correction(session_id, voice, user_msg)`
in `src/server.py` draait vóór elke chat-turn én elke `_tg_run_agent`-turn.
Regex-detectie van user-correcties (8 patterns NL+EN: 'nee dat klopt niet',
'ik bedoelde', 'no that's wrong', etc.). Bij hit: vorige assistant-turn +
correctie-tekst → MISTAKE-entry in agent's journal. Best-effort; mag chat
nooit breken.

(3) **Distillatie aangepast**: meta-prompt in `tools.execute(
"distill_recent_journals", ...)` zoekt nu expliciet naar 'MISTAKE:' regels
en stelt voor om patronen vast te leggen als 'common pitfalls' secties of
nieuwe constraints in self-pages.

Cost: nul extra Anthropic-calls voor de capture zelf (regex is lokaal).
Distillatie kost zelfde als Phase 4 — alleen het meta-prompt is iets langer.

Smoke-test: 8/8 positives + 4/4 negatives op regex, flag_mistake direct
+ via dispatch, end-to-end correction capture met opgeschoonde test-entries.

## [2026-05-20] ingest | Prompt 1 (deel 1) — persistente conversation history (SQLite)
Lost het Phase 2 open-issue op: `histories` in `src/server.py:99` was in-
memory dict die bij elke launchd-kickstart stierf. Vervangen door SQLite
in nieuwe module `src/conversations.py` (~110 lines): `append`, `recent`,
`clear`, `session_count`, `db_path`. WAL-mode + connection-per-call voor
thread-safety. Schema lazy-init bij eerste call.

Server.py 6 plekken gepatcht: declaratie compat-shim, `/api/chat` read+
trim+write, `DELETE /api/chat/{id}`, `_tg_run_agent` read+trim+write.
HISTORY_LIMIT trim-logica weg — SQLite handelt grootte gewoon. Telegram-
`tg-<voice>` keys werken hetzelfde als chat session_ids (één tabel,
verschillende prefixes).

DB-file: `~/javis 2.0/conversations.db` (16KB na init). Gitignored.

Smoke-test: append+recent round-trip, limit-parameter, clear, empty
session_id safe-ignore, persistence over module-reload — allemaal groen.
Live geverifieerd na kickstart (server start schoon op, DB autom. aangemaakt).

## [2026-05-20] decision | thehandycompany.be publieke architectuur (planned)
Nieuwe decision-pagina [[2026-05-20-public-website-architecture]]. Drie
opties geanalyseerd voor publieke toegang vanuit klanten:
- A: Tailscale Funnel + DNS-CNAME (cert-mismatch probleem)
- B: Split — one.com statisch frontend + Tailscale Funnel API (mijn voorkeur)
- C: Cloudflare Tunnel (nieuwe vendor, sterkste cert-story)

Status=planned tot Tristan kiest. Prompt 4 (publieke offerte/facturatie)
hangt hier vanaf.

## [2026-05-20] ingest | Phase 7 shipped — auto-distill cron (wiki groeit autonoom)
Verwijdert de laatste handmatige trigger uit de pipeline. Achtergrond-worker
in `src/autonomy.py` plant dagelijks **04:00 lokaal** een Valentina-led
distillatie via dezelfde code-pad als Phase 4 (`tools.execute(
"distill_recent_journals", {window_hours: 24, max_proposals: 3})`).
Bevat: `_AUTO_DISTILL_ENABLED` env-gate (default 1), `_next_04am()`
timestamp-helper, `_distill_run()` wrapper, en een nieuwe `_next_distill`
slot in de bestaande `_loop()` if-elif-keten (hoogste prioriteit boven
andere beats).

Notificatie-beleid: stil bij 0 voorstellen, Telegram-bericht aan Tristan
met `wiki apply <id>` knoppen zodra er minstens 1 voorstel gestaged is.
Detectie via regex op het 12-hex-id patroon in de distill-output.

Restart-veilig: `_next_distill` wordt fresh berekend bij elke
`_guarded_loop()` herstart. Geen state-file nodig. Bij restart binnen het
04:00-window slaat distillatie die dag over (acceptabel — niet
tijdkritisch). Bij Anthropic-fout: try/except in `_distill_run`, log naar
`logs/autonomy.log`, geen crash van de autonomy thread.

Bewust GEEN nieuwe agent-rol/identiteit ingevoerd — distill draait als
Valentina via `active_voice("valentina")`. Sluit aan op haar bestaande
"waarnemend directeur"-positie als coördinator. Steve/Nick/Siebert blijven
buiten de wiki-loop (zoals sinds Phase 2).

Eerstvolgende run: 2026-05-21 04:00 (geverifieerd in autonomy.log na
kickstart). Pas zichtbaar nut zodra agents over meerdere dagen journals
accumuleren. ~$0.20-0.50/dag aan Sonnet-input tokens.

## [2026-05-20] ingest | Phase 6 shipped — wiki lint (Karpathy triumvirate compleet)
Sluit het Karpathy ingest→query→lint trio. Nieuwe Valentina-only tool
`lint_wiki(stale_days=30)` voert drie statische checks uit (geen LLM-call):
- **Orphans**: pagina's zonder inbound wikilinks (excl. auto-maintained
  index/log/README).
- **Missing concepts**: `[[slug]]`-refs waarvoor geen pagina bestaat.
- **Stale**: pagina's met `updated:` ouder dan threshold EN minimaal 1
  inbound link (stale orphans worden niet dubbel geflagd).

Implementatie: `wiki.lint_wiki()` returnt structured dict; `format_lint_
report()` formatteert naar markdown voor Telegram/agent-consumption.
Resolver matched zowel full-slugs ('concepts/foo') als bare stems ('foo') —
ambiguous stems → missing.

Baseline op huidige wiki: 16 pagina's, 97 links, **0 orphans / 0 missing /
0 stale** — gezonde startstand. Pas zichtbaar nut als wiki organisch groeit
via proposal-edits.

Bewust uit v1: contradictie-detectie (vereist LLM-scan over alle pagina's,
duurder + noisy) en coverage-gap analyse (concepten herhaald in journals
zonder eigen pagina). Beide kandidaten voor Phase 6.5 als echt nut blijkt.

## [2026-05-20] ingest | Phase 5 shipped — expanded change_types (new_page + replace_section)
Heffen van de append-only beperking. `propose_wiki_edit` ondersteunt nu drie
`change_type`s: `append_section` (zoals voorheen), `new_page` (creëer nieuwe
pagina, target mag niet bestaan), en `replace_section` (vervang bestaande
sectie geïdentificeerd via heading-tekst, case-insensitive). Distillatie kan
nu dus ook nieuwe pagina's en correcties voorstellen — Phase 4-bug waar
Valentina `concepts/shared-brain` wou aanmaken is opgelost.

Backups: elke apply die een bestaande pagina raakt schrijft eerst de oude
versie naar `.bak/<slug>.<timestamp>.bak` (gitignored). Reversibility zonder
git.

Implementatie:
- `wiki.stage_proposal` herschreven met optionele `change_type` en
  `section_heading` parameters. Per-type validatie (new_page weigert
  bestaande targets; replace_section weigert ontbrekende targets).
- `wiki.apply_proposal` route nu naar `_apply_append_section`,
  `_apply_new_page`, of `_apply_replace_section`. Backup vóór elke wijziging
  via `_backup_target`. Replace zoekt heading regex case-insensitief en
  bepaalt section-einde via volgende heading van gelijk-of-lager level.
- `tools.py` `propose_wiki_edit` schema uitgebreid met `change_type` enum
  en `section_heading` veld. Distill-prompt updated zodat Valentina nu alle
  drie types kan voorstellen.

Smoke-test: alle 6 paden (3 happy + 3 edge cases) groen — zie javis-2.0
commit voor de details.
