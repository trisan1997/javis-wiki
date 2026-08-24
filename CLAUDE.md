# Javis Wiki — Schema & Maintainer Guide

You are the maintainer of this wiki. The wiki is a persistent knowledge base
about the **Javis 2.0** project at `/Users/tristanklaasse/javis 2.0/` — its
services, concepts, decisions, and incidents. You write and update wiki pages;
the human (Tristan) curates sources, asks questions, and reads the result in
Obsidian.

## Three layers

1. **`raw/`** — immutable sources: configs, transcripts, snapshots, plists, exported chats. **NEVER modify** files in here. Treat them as evidence.
2. **`services/` `concepts/` `incidents/` `decisions/`** — the wiki itself. Markdown pages you create and maintain.
3. **`CLAUDE.md`** (this file) — the schema. Co-evolves as we learn what works. Edit when conventions change.

## Directory layout

- `services/` — one page per running service or daemon (`jarvis-server`, `sara-mail-agent`, `habbo-hq`, …). What it does, how it runs, where its files/logs are.
- `concepts/` — cross-cutting topics (`always-on-launchd`, `voice-tts`, `tailscale-access`, …). The "why" and "how it works" pieces that span multiple services.
- `agents/` — one page per agent (`jarvis`, `zoe`, `sara`, `valentina`, `steve_jobs`, `nick`, `siebert`, …). Persistent self-knowledge: identity, voice, tools, preloaded context, current focus. Agents will read their own page as part of every turn (Phase 1 of shared-brain).
- `journal/` — append-only daily journals, one file per agent per day. Filename: `YYYY-MM-DD-<voice>.md`. Replaces the in-memory `tg-<voice>` histories that die on server restart (Phase 2 of shared-brain).
- `journal/sessions/` — session-brain per Claude Code-sessie: `YYYY-MM-DD-HHMMSS-claude.md` met frontmatter (session_id, intent, status). Aangemaakt door `.claude/helpers/session-brain.mjs` bij SessionStart, geappend bij SessionEnd. Atlas tegen gedeeltelijke herinnering (andere sessie, andere dag).
- `proposals/` — agent-staged wiki-edit proposals awaiting approval (Phase 3 + Phase 5). Filename: `<12-hex-id>.md`. Three `change_type`s: `append_section`, `replace_section`, `new_page`. Approved via Tristan (panel + Telegram) or Valentina (LLM tool, can't self-approve). Backups of pre-change page state land in `.bak/` (gitignored). See `proposals/README.md`.
- `incidents/` — post-mortems. Filename: `YYYY-MM-DD-short-slug.md`.
- `decisions/` — ADRs. Filename: `YYYY-MM-DD-decision-slug.md`. Title, context, decision, consequences.
- `raw/` — source material. Drop new sources here before ingesting. Keep filenames descriptive (`2026-05-15-com.jarvis.server.plist`).
- `index.md` — content catalog. One line per page. Grouped by directory. Update on every ingest.
- `log.md` — append-only chronological record. Format: `## [YYYY-MM-DD] <op> | <title>` then 1–3 lines of body.

## Page conventions

- Filenames: `kebab-case.md`, no spaces, no dates in service/concept names.
- YAML frontmatter on every wiki page:
  ```yaml
  ---
  title: Jarvis Server
  type: service              # service | concept | incident | decision
  status: live               # live | dormant | abandoned | planned
  updated: 2026-05-20
  sources: 2                 # count of raw/ entries referenced
  tags: [jarvis, runtime, network]
  ---
  ```
- Wikilinks: `[[page-name]]` (matches Obsidian). Link liberally — an unresolved `[[link]]` is a useful TODO marker, not an error.
- Cite raw sources inline as `(see raw/<file>)` when stating a specific fact.
- Keep pages under 400 lines. Split if needed.
- Lead with a 1–2 sentence summary so the index line writes itself.

## Workflows

### Ingest
When Tristan adds a source (or names one in `raw/`), or asks "ingest this":
1. Read the source from `raw/`.
2. Discuss key takeaways with Tristan.
3. Update or create relevant pages. A single source typically touches 5–15 pages.
4. Bump `updated:` and `sources:` on each touched page.
5. Update `index.md` with any new pages.
6. Append a log entry: `## [YYYY-MM-DD] ingest | <source title>` + 1–3 lines on what changed.

### Query
When Tristan asks a question:
1. Read `index.md` first to find relevant pages. Drill in from there.
2. Synthesize an answer with `[[page]]` citations.
3. If the answer is non-trivial and reusable, **offer** to file it back as a wiki page (analyses, comparisons, walkthroughs are valuable).
4. Append a log entry: `## [YYYY-MM-DD] query | <question>` + the answer's headline.

### Lint
When Tristan asks "lint" or periodically:
- Find contradictions between pages.
- Find stale claims newer sources have superseded.
- Find orphan pages (no inbound `[[links]]`).
- Find concepts mentioned but lacking their own page.
- Suggest new sources or questions worth pursuing.
- Append log entry: `## [YYYY-MM-DD] lint | <summary>`.

## Rules

- NEVER modify files in `raw/`.
- NEVER modify code in `/Users/tristanklaasse/javis 2.0/`. This wiki documents it; it doesn't change it.
- NEVER invent facts. If unsure, mark `<!-- TODO: verify -->` inline.
- Ask Tristan before deleting wiki pages or restructuring directories.
- When a memory file (from `~/.claude/projects/.../memory/`) seeds a page, treat it as a starting point but note `updated:` to today; memories may be stale — verify against current code before quoting as fact.
- Never commit secrets. If a raw source contains secrets, redact before saving to `raw/`.

## Bootstrap

Wiki created **2026-05-20** from Claude Code memory files. Initial pages seeded
from: `jarvis-startup`, `jarvis-voice-tts`, `sara-mail-agent`,
`virtueel-kantoor`. No `raw/` sources yet — bootstrap content was derived from
memory only and is marked accordingly on each page.
