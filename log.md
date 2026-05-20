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
