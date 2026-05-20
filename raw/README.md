# raw/

Immutable source material. The maintainer reads from here but **never modifies
files in this directory**. Drop new sources here before asking for an ingest.

## What lives here

- Snapshots of plists (`com.jarvis.server.plist`, etc.).
- Exported chat transcripts (Telegram, customer conversations) — redacted.
- Config snapshots (`.env.example`, `Caddyfile`, redacted `customers.json`).
- Log slices for post-mortems.
- Screenshots, if relevant to a page.
- Anything else worth treating as evidence rather than narrative.

## What does NOT live here

- Anything with secrets (real `.env`, real `customers.json` with hashes, raw Telegram chat IDs unless redacted).
- Source code from `/Users/tristanklaasse/javis 2.0/` itself — link to it from a wiki page instead. Only snapshot specific files here if you need a frozen point-in-time copy for a decision/incident page.

## Naming

Descriptive kebab-case. Date-prefix when snapshotting something that changes
over time: `2026-05-15-com.jarvis.server.plist`.
