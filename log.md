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

## [2026-05-20] ingest | Prompt 6 shipped — universele commando-dispatch
Elke agent (incl. dev-team) is nu direct adresseerbaar via Telegram met
zijn eigen tool-use loop, en team-leads kunnen delegeren via een nieuwe
`assign_to_teammate` tool. Vier veranderingen:

**1. Dev-team direct adresseerbaar.** `Steve, refactor X` / `Nick, ...` /
`Siebert, ...` route nu naar hun eigen tool-use loop in `_tg_run_agent`
(niet meer alleen via `dev: <opdracht>` collectief). `_TG_NAME_RE`
uitgebreid met `steve|steve_jobs|steve_job's|nick|siebert`. Label-map
uitgebreid met de drie devs.

**2. Nieuwe DEVTEAM_MEMBER_TOOLS lijst** = `_SHARED + _WIKI_TOOLS +
[_TEAMLEAD_TOOL]` (15 tools). Geen mail/calendar (niet hun werk), geen
quote-tools (Jarvis-only). Centralized via nieuwe `_tools_for_voice()`
helper die de oude if/elif-keten op 4 plekken vervangt.

**3. assign_to_teammate tool** (voice-gated). Valentina mag iedereen
behalve zichzelf delegeren; Steve_Job's alleen Nick of Siebert.
Interne aanroep van `_tg_run_agent(target, task)` — teammate doet zijn
eigen tool-use loop, antwoord komt terug als tool-result aan de lead.

**4. Recursion guard via `threading.local()` _recursion_local`** —
maximaal 2 niveaus delegatie (lead → teammate, geen verdere ketens).
Voorkomt oneindige loops.

**Bonus: team-prefix syntax** `team:dev <opdracht>` in Telegram —
alias voor de bestaande `dev: ...` dispatch. Toekomstige teams
(adverteam, boekhouding) worden hier toegevoegd zodra ze bestaan
(zie Prompt 2 — adverteam/louche boekhouder/adviseur).

**Autonomy-light gate**: dev-team voices kregen voorheen geen wiki-
preload omdat autonomy._gen() ze ook gebruikt voor 1-zins whispers
(kosten). Nu: preload ALLEEN als `active_voice == voice` (set door
server.py's tool-use wrappers). Whispers blijven lean (~1KB), tool-use
turns krijgen volledige preload (~4-5KB).

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

## [2026-05-20] wiki-apply | proposal 25c11475f24e (append_section) → agents/steve_jobs
Proposer: steve_jobs. Approver: tristan-telegram. sectie toegevoegd (1963 chars). (backup: .bak/agents-steve_jobs.md.20260520T213120.bak)

## [2026-05-20] wiki-apply | proposal 659929e18a7d (append_section) → agents/steve_jobs
Proposer: steve_jobs. Approver: tristan-telegram. sectie toegevoegd (3250 chars). (backup: .bak/agents-steve_jobs.md.20260520T213429.bak)

## [2026-05-20] wiki-reject | proposal 41859de44fa9
Proposer: steve_jobs. Approver: tristan. Reden: outdated — Valentina/Zoë bugs reeds gefixt via Phase 2.5 (valentina alias in characters.load()). Steve las log.md niet vóór hij voorstelde..

## [2026-05-20] wiki-reject | proposal 48897c3a8c36
Proposer: steve_jobs. Approver: tristan. Reden: outdated — duplicate van 41859de44fa9; zelfde bugs zijn al gefixt..

## [2026-05-20] ingest | Prompt 4 (backend) — quote-persistence + invoicing + CORS + admin endpoints
Backend-fundamenten voor de publieke offerte/facturatie flow op
thehandycompany.be (Optie B — one.com statisch frontend + Tailscale Funnel
API). Frontend en Funnel-enable volgen in een volgende sessie.

Nieuwe modules:
- `src/quotes.py` — persistente quote-opslag in `config/quotes.json`
  (gitignored). add/list_all/get/update/mark_status. Hook in
  `tools._create_and_send_quote` schrijft elke offerte ook persistent weg
  + markeert hem als `status="sent"`.
- `src/invoicing.py` — facturatie met jaarsequentiële nummering
  (`INV-YYYY-NNNN`). `create_from_quote()` rekent BTW 21%, rendert HTML-
  snapshot in `static/invoice_<nr>.html`, linkt heen-en-weer met de quote.
  Status: `draft → sent → (paid | overdue | cancelled)`. `paid` is read-only.
  Dubbele facturatie geblokkeerd.

Server-side: CORS-middleware voor `https://thehandycompany.be`,
BasicAuth exempt nu ook `/api/public/*` + OPTIONS preflights.

Admin endpoints (director-token gated): 6 routes in office.py voor
quotes (list/get/update) + invoices (list/get/from-quote/update/paid).

Gitignore extensions: alle customer-data uit git (quotes.json,
invoices.json, *-counter.json, static/invoice_*.html, static/quote_preview.html).

Smoke-test groen: end-to-end flow van quote-add → invoice-create
(INV-2026-0001 met BTW 21% van 435.60 = 91.48, totaal 527.08) →
mark_paid persisteert correct. Live na kickstart (PID 77604).

Niet in deze turn: publieke `/api/public/offerte` route, statisch frontend
voor one.com, Tailscale Funnel enablen. Backend is klaar om die laag te dragen.

## [2026-05-20] ingest | Prompt 4 (frontend) — admin UI + one.com bundle + publiek chat endpoint
Frontend voor de publieke offerte/facturatie flow op thehandycompany.be is klaar.

**Nieuwe backend endpoints in src/server.py:**
- `POST /api/public/chat` — auth-vrij chat endpoint voor klanten. Voice
  forced op Jarvis (de sales-agent), session-id krijgt automatisch `pub-`
  prefix zodat publieke gesprekken niet kruisen met Tristan-side chats.
  Volledige tool-use loop (create_and_send_quote werkt). Smoke-getest:
  Jarvis antwoordt correct met zijn OFFERTESTROOM-prompt zonder Basic auth.
- `GET /admin` — serveert `static/admin.html`. Achter BasicAuth (gewone
  app-auth), daarna doet de UI zelf een director-token login.

**static/admin.html** (~430 lines vanilla JS/CSS, single-page):
- Login via OFFICE_PASSPHRASE → director-token in localStorage
- Twee tabs: Offertes + Facturen
- Lijst-view + detail-view met inline edit-formulier
- Action-knoppen per item: "Maak factuur" (vanuit quote), "Markeer betaald",
  "Geaccepteerd"/"Afgewezen", "Wijzigingen opslaan"
- Status-pills met kleur per status
- Linkt naar HTML-snapshot van facturen (browser print-to-PDF)
- Toast-notificaties bij success/error

**frontend/public/ bundle** (klaar voor FTP-upload naar one.com):
- `index.html` — landing page + offerte-chat in één pagina, sticky topbar,
  hero met CTA, contact-sectie, footer
- `style.css` — branding (oranje accent #e8772a, donker contrast)
- `app.js` — chat-logica via `fetch()` naar Tailscale Funnel URL. Session-id
  bewaard in localStorage zodat klanten hun gesprek kunnen voortzetten.
  IntersectionObserver focus inputveld bij scroll-into-view.
- `README.md` — deploy-instructies (Tailscale Funnel enable, FTP-upload,
  lokaal testen, architectuur-diagram)

**Stap die Tristan ZELF moet doen (eenmalig):**
```bash
tailscale funnel --bg 443
```
Dit exposeert de Mac-backend publiek op port 443 via de tail5dedea.ts.net
hostname. Vereist Funnel-rechten in de Tailscale admin console.

Daarna: drie files van `frontend/public/` uploaden naar one.com via hun
File Manager (vervangt het default `index.html`).

**Live geverifieerd:** kickstart schone startup (no errors), `/api/public/chat`
gaf 200 + Jarvis-reply, `/admin` gaf 401 zonder BasicAuth (correct gated).

## [2026-05-21] wiki-apply | proposal 907d621fc5c7 (new_page) → concepts/agent-memory-protocol
Proposer: valentina. Approver: tristan-telegram. nieuwe pagina aangemaakt (2102 chars).

## [2026-06-02] wiki-apply | proposal 230f165c5f50 (new_page) → incidents/2026-05-20-duplicate-mail-processing
Proposer: valentina. Approver: tristan. nieuwe pagina aangemaakt (1628 chars).

## [2026-06-02] wiki-apply | proposal 28e5adbbbf8a (new_page) → incidents/pricing-discrepancies-june-2026
Proposer: valentina. Approver: tristan. nieuwe pagina aangemaakt (1181 chars).

## [2026-06-02] wiki-apply | proposal 3b7671b9fea1 (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (1038 chars). (backup: .bak/services-sara-mail-agent.md.20260602T024441.bak)

## [2026-06-02] wiki-apply | proposal 834eed877dc4 (new_page) → concepts/upsell-tracking
Proposer: valentina. Approver: tristan. nieuwe pagina aangemaakt (714 chars).

## [2026-06-02] wiki-apply | proposal 9d7f945ab2aa (new_page) → concepts/quote-pricing
Proposer: valentina. Approver: tristan. nieuwe pagina aangemaakt (1303 chars).

## [2026-06-02] wiki-reject | proposal 9dec374e10d2
Proposer: valentina. Approver: tristan. Reden: (geen).

## [2026-06-02] wiki-reject | proposal 42c0917668bc
Proposer: valentina. Approver: tristan. Reden: (geen).

## [2026-06-02] wiki-reject | proposal 987bb3d2e6d6
Proposer: valentina. Approver: tristan. Reden: (geen).

## [2026-06-02] wiki-reject | proposal 9f6c5d126a45
Proposer: valentina. Approver: tristan. Reden: (geen).

## [2026-06-02] wiki-apply | proposal b6f45d287bf9 (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (1160 chars). (backup: .bak/services-sara-mail-agent.md.20260602T024716.bak)

## [2026-06-02] wiki-reject | proposal d3014bd16d27
Proposer: valentina. Approver: tristan. Reden: (geen).

## [2026-06-02] wiki-apply | proposal fa4f72bf71da (append_section) → services/jarvis-server
Proposer: valentina. Approver: tristan. sectie toegevoegd (593 chars). (backup: .bak/services-jarvis-server.md.20260602T024749.bak)

## [2026-06-02] wiki-apply | proposal fa9705d4808f (replace_section) → agents/jarvis
Proposer: jarvis. Approver: tristan. sectie 'Tools' vervangen (7 regels → 26 regels). (backup: .bak/agents-jarvis.md.20260602T024803.bak)

## [2026-07-13] wiki-apply | proposal 0f35aaee0578 (new_page) → incidents/sara-mail-sync-issue
Proposer: valentina. Approver: tristan. nieuwe pagina aangemaakt (1369 chars).

## [2026-07-13] wiki-apply | proposal 101db71438f2 (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (591 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154635.bak)

## [2026-07-13] wiki-apply | proposal 20170cc76854 (new_page) → incidents/mail-sync-delete-failure
Proposer: valentina. Approver: tristan. nieuwe pagina aangemaakt (1616 chars).

## [2026-07-13] wiki-apply | proposal 4d94d9b4e8aa (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (797 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154638.bak)

## [2026-07-13] wiki-apply | proposal 669687af7063 (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (798 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154639.bak)

## [2026-07-13] wiki-apply | proposal 75ad84516f89 (append_section) → concepts/voice-tts
Proposer: valentina. Approver: tristan. sectie toegevoegd (506 chars). (backup: .bak/concepts-voice-tts.md.20260713T154640.bak)

## [2026-07-13] wiki-apply | proposal 89265cc806ca (append_section) → concepts/voice-tts
Proposer: valentina. Approver: tristan. sectie toegevoegd (659 chars). (backup: .bak/concepts-voice-tts.md.20260713T154641.bak)

## [2026-07-13] wiki-apply | proposal 923c1398bbeb (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (916 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154642.bak)

## [2026-07-13] wiki-apply | proposal 980aa5f595c6 (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (803 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154643.bak)

## [2026-07-13] wiki-apply | proposal 9877dec0d80f (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (739 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154644.bak)

## [2026-07-13] wiki-apply | proposal a936cf46170a (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (862 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154652.bak)

## [2026-07-13] wiki-apply | proposal cff87c232877 (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (770 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154654.bak)

## [2026-07-13] wiki-apply | proposal d90f897fb534 (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (767 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154655.bak)

## [2026-07-13] wiki-apply | proposal df6eab026a64 (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (533 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154657.bak)

## [2026-07-13] wiki-apply | proposal f1d1d22e5bfa (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (664 chars). (backup: .bak/services-sara-mail-agent.md.20260713T154706.bak)

## [2026-07-13] wiki-reject | proposal ee6636f82adb
Proposer: valentina. Approver: tristan. Reden: (geen).

## [2026-07-13] wiki-reject | proposal a56c36c10c0f
Proposer: valentina. Approver: tristan. Reden: bug.

## [2026-07-14] wiki-apply | proposal 00aa06a348e4 (append_section) → concepts/voice-tts
Proposer: valentina. Approver: tristan. sectie toegevoegd (391 chars). (backup: .bak/concepts-voice-tts.md.20260714T145009.bak)

## [2026-07-14] wiki-apply | proposal a813f8d43b0f (new_page) → incidents/offerte-prijscalculatie-renovatie
Proposer: valentina. Approver: tristan. nieuwe pagina aangemaakt (881 chars).

## [2026-07-14] wiki-apply | proposal d66048659a12 (append_section) → services/jarvis-server
Proposer: valentina. Approver: tristan. sectie toegevoegd (530 chars). (backup: .bak/services-jarvis-server.md.20260714T145012.bak)

## [2026-07-16] wiki-apply | proposal 8fa0a6e3487a (append_section) → services/sara-mail-agent
Proposer: valentina. Approver: tristan. sectie toegevoegd (489 chars). (backup: .bak/services-sara-mail-agent.md.20260716T020054.bak)

## [2026-07-20] wiki-apply | proposal 37adfdd98d2c (new_page) → concepts/mail-handling-strategy
Proposer: valentina. Approver: tristan. nieuwe pagina aangemaakt (946 chars).

## [2026-08-17] update | Voice — TTS & STT
Sectie toegevoegd: mail-agents ([[sara]]) hebben geen afhankelijkheid van de
spraakstack — geverifieerd dat `src/routing.py` en `src/office.py` geen
`/api/speak` of `/api/transcribe` aanroepen. Voorkomt dat er audio-pipelines
worden opgetuigd voor tekst-only taken.
