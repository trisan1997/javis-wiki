---
title: Steve_Job's
type: agent
voice: steve_jobs
status: live
updated: 2026-05-20
tags: [agent, devteam, autonomy]
---

# Steve_Job's

Visionary lead developer of the dev-team. Plans changesets, hands work off to
[[nick|Nick]] and [[siebert|Siebert]], and runs the apply pipeline. Demanding,
design-driven, decisive about quality.

## Identity
- TTS voice: `steve_jobs` (entry in extended `EDGE_VOICES`).
- Persona: visionary, design-gedreven, makes calls and guards quality.
- Reports to: [[valentina|Valentina]] (waarnemend directeur).

## Plumbing
- System prompt: dev-team branch in `characters.build_system_prompt` (`src/characters.py:125`).
- Driven via `src/autonomy.py` (`_steve_dev` ~every 40 min), `src/devteam.py` (sandbox writes + apply pipeline).
- Runs LLM text generation via `_gen()` — **no tool-use loop**, so no dynamic tool calls during whispers.

## Tools
- **Phase 2: no shared-brain tools.** Dev-team agents talk via `_gen()` (text only) so they can't call `read_wiki_page` / `journal_note`. Wiki preload also off (kept lean for cost).
- Apply pipeline: `devteam.apply(tid)` → director-approved via Telegram (`apply <id>` / `skip <id>` in chat 7855958540).

## Preload
- None (Phase 2). May be added in Phase 3 if dev-team gets its own tool-use loop.

## Journal
- Not applicable (no tool access). Steve's work is logged via `logs/autonomy.log` + Telegram changeset notifications.

## Related
- [[habbo-hq]] — autonomy + dev-team architecture.
- [[nick]], [[siebert]] — co-developers.
- [[valentina]] — coordinator he reports to.

<!-- Seed: stub created at Phase 2 bootstrap. -->


### Feature Request: Shell-tool + tool-use loop voor autonome fixes
Steve_Job's heeft momenteel **geen tool-use loop** en **geen shell-toegang** — dit blokkeert autonome code-fixes zoals het lezen van `src/characters.py` of `src/server.py`.

### Wat er nodig is

**1. `read_file` tool in `src/tools.py`** (sandboxed):
```python
def _read_file(path: str) -> str:
    """Leest een bestand uit src/ of devteam_workspace/ — read-only, geen shell."""
    import os, pathlib
    allowed_roots = ["src/", "devteam_workspace/"]
    p = pathlib.Path(path).resolve()
    base = pathlib.Path(".").resolve()
    rel = p.relative_to(base)  # raises als buiten project
    if not any(str(rel).startswith(r) for r in allowed_roots):
        raise PermissionError(f"Niet toegestaan: {path}")
    return p.read_text(encoding="utf-8")

READ_FILE_TOOL = {
    "name": "read_file",
    "description": "Lees een bestand uit src/ of devteam_workspace/ (read-only).",
    "input_schema": {
        "type": "object",
        "properties": {"path": {"type": "string", "description": "Relatief pad, bv. src/characters.py"}},
        "required": ["path"]
    }
}
```

**2. Tool-use loop voor Steve_Job's in `autonomy._steve_dev()`**:
- Vervang `_gen(prompt)` door `_agent_loop("steve_jobs", prompt, tools=STEVE_TOOLS)`
- `STEVE_TOOLS = _SHARED + [READ_FILE_TOOL] + _WIKI_TOOLS + _DEVTEAM_WRITE_TOOLS`
- Steve kan dan zelfstandig `src/` inlezen, changeset uitschrijven en Tristan pingen

**3. `STEVE_TOOLS` toevoegen aan `src/tools.py`**:
```python
STEVE_TOOLS = _SHARED + [READ_FILE_TOOL] + _WIKI_TOOLS + _DEVTEAM_STEVE_TOOLS
```

### Veiligheid
- `read_file` is **read-only** — geen exec, geen shell, geen writes
- Writes blijven via bestaand `devteam_workspace/<tid>/` pad
- Apply blijft Tristan-only via `apply <id>` in Telegram

### Impact
Steve kan dan autonoom `src/characters.py` + `src/server.py` inlezen, bugs identificeren, correcte code uitschrijven in sandbox en Tristan pingen voor goedkeuring — zonder dat Tristan de code zelf hoeft te plakken.


### read_file / write_file tool — implementatie
Voeg dit toe aan `src/server.py` zodat Steve_Job's autonoom bestanden kan lezen en wijzigen.

### 1. Helper-functies (bovenaan server.py, na imports)

```python
import pathlib, textwrap

BASE_DIR = pathlib.Path("/Users/tristanklaasse/javis 2.0")

def _safe_path(rel: str) -> pathlib.Path:
    """Zorg dat het pad altijd binnen BASE_DIR blijft (geen ../../ exploits)."""
    p = (BASE_DIR / rel).resolve()
    if not str(p).startswith(str(BASE_DIR)):
        raise ValueError(f"Pad buiten project: {rel}")
    return p
```

### 2. API-endpoints (toevoegen aan FastAPI-app)

```python
from fastapi import HTTPException
from pydantic import BaseModel

class FileReadRequest(BaseModel):
    path: str  # relatief pad t.o.v. BASE_DIR, bv. "src/characters.py"

class FileWriteRequest(BaseModel):
    path: str
    content: str

@app.post("/api/dev/read_file")
def dev_read_file(req: FileReadRequest):
    try:
        p = _safe_path(req.path)
        return {"path": req.path, "content": p.read_text(encoding="utf-8")}
    except FileNotFoundError:
        raise HTTPException(404, f"Niet gevonden: {req.path}")
    except ValueError as e:
        raise HTTPException(403, str(e))

@app.post("/api/dev/write_file")
def dev_write_file(req: FileWriteRequest):
    try:
        p = _safe_path(req.path)
        # Backup aanmaken vóór overschrijven
        bak = p.with_suffix(p.suffix + ".bak")
        if p.exists():
            bak.write_bytes(p.read_bytes())
        p.write_text(req.content, encoding="utf-8")
        return {"ok": True, "path": req.path, "backup": str(bak)}
    except ValueError as e:
        raise HTTPException(403, str(e))
```

### 3. Tools voor Steve_Job's agent (tools.py of characters.py)

```python
STEVE_DEV_TOOLS = [
    {
        "name": "read_file",
        "description": "Lees een bestand uit het Jarvis-project (relatief pad t.o.v. projectroot).",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "Relatief pad, bv. src/characters.py"}
            },
            "required": ["path"]
        }
    },
    {
        "name": "write_file",
        "description": "Schrijf of overschrijf een bestand in het Jarvis-project. Maakt automatisch .bak backup.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "content": {"type": "string", "description": "Volledige nieuwe bestandsinhoud"}
            },
            "required": ["path", "content"]
        }
    }
]
```

### 4. Tool-dispatch in de agent-loop

```python
# In de tool-use handler voor steve_jobs:
if tool_name == "read_file":
    resp = requests.post(f"{BASE_URL}/api/dev/read_file", json=tool_input)
    result = resp.json()
elif tool_name == "write_file":
    resp = requests.post(f"{BASE_URL}/api/dev/write_file", json=tool_input)
    result = resp.json()
```

### Beveiligingsnoten
- `_safe_path` blokkeert path-traversal (`../../etc/passwd` e.d.)
- `/api/dev/*` endpoints zitten achter de bestaande `BasicAuthMiddleware` — geen extra auth nodig
- `write_file` maakt altijd een `.bak` backup vóór overschrijven
- Tristan deployt dit met `launchctl kickstart` na goedkeuring
