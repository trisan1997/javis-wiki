# proposals/

Agent-staged wiki-edit proposals, awaiting approval (Phase 3 of shared-brain).

## How it works

1. **Agent stages** a proposal via `propose_wiki_edit(slug, change_type, heading, section_heading, content, rationale)`.
   - Saved here as `<12-hex-id>.md` with frontmatter (`status: pending`).
   - **Three `change_type`s** (Phase 5):
     - `append_section` — add a new H3 section to an EXISTING page (default).
     - `replace_section` — replace an existing section, identified by its heading text. Used to correct misinformation rather than just adding.
     - `new_page` — create a NEW page (target must not exist). Body auto-wrapped with minimal frontmatter if proposer didn't supply their own.
   - Backups: any apply that modifies an existing page writes a copy of the old version to `.bak/<slug>.<timestamp>.bak` (gitignored).
2. **Approval** can come from three channels (each applies the same atomic write):
   - **Tristan via HQ panel**: `GET /api/hq/wiki-proposals`, `POST /api/hq/wiki-proposal/apply` (director-token).
   - **Tristan via Telegram**: send `wiki apply <id>` or `wiki skip <id>` in chat `7855958540`.
   - **Valentina via LLM tool**: `approve_wiki_proposal(proposal_id, decision)` — voice-gated to Valentina only; she may **never** approve her own proposals (server-side check).
3. **Apply** appends the proposed section to the target page, marks the proposal `status: approved`, and logs an entry to `log.md`.
4. **Reject** marks `status: rejected` + logs.

## Target whitelist

Agents can only propose edits to:
- `services/*`, `concepts/*`, `decisions/*`, `incidents/*` — any agent.
- `agents/<self>` — only the agent's own page (e.g. Jarvis can edit `agents/jarvis`, never `agents/sara`).

**Blocked**: `CLAUDE.md`, `index.md`, `log.md`, `raw/*`, `journal/*`, `proposals/*`, `.obsidian/*`. Enforced server-side in `wiki._validate_proposal_target`.

## Proposal file format

```markdown
---
id: a1b2c3d4e5f6
proposer: jarvis
target: concepts/voice-tts
change_type: append_section
status: pending          # pending → approved | rejected
created: 2026-05-20T10:15:00
approver:                # added on decision
decided_at:              # added on decision
reject_reason:           # added on reject (optional)
---

# Voorstel a1b2c3d4e5f6

**Proposer:** jarvis  
**Target:** [[concepts/voice-tts]]  
**Type:** append_section

## Rationale
<why this is useful — visible to the approver>

## Voorgestelde sectie (wordt toegevoegd aan concepts/voice-tts.md)

### <H3 heading>
<markdown body the agent wants appended>
```

## Why no `delete_section` (yet)

Phase 5 supports append, replace, and new_page — covering 95% of real wiki
maintenance. Pure deletes (remove a section, leaving the rest of the page
intact) are rare in practice (replace with a smaller version usually works)
and add another safety surface. Deferred until real usage shows it's needed.
