---
title: Always-on launchd pattern
type: concept
status: live
updated: 2026-05-20
sources: 0
tags: [launchd, macos, deployment]
---

# Always-on launchd pattern

How Javis services run unattended on Tristan's iMac.

## The pattern

LaunchAgent plist under `~/Library/LaunchAgents/`, label `com.jarvis.<thing>`:
- `RunAtLoad` — start when user logs in.
- `KeepAlive` — auto-restart on crash (verified).
- Wraps a shell entrypoint in the project (e.g. `scripts/start.sh`).
- Env `JARVIS_NO_RELOAD=1` to disable uvicorn `--reload` in service mode (avoids log-file restart loops). Interactive runs of the same script keep reload enabled.

## Manage

```bash
launchctl load -w   ~/Library/LaunchAgents/com.jarvis.server.plist
launchctl unload -w ~/Library/LaunchAgents/com.jarvis.server.plist
launchctl list | grep jarvis
launchctl kickstart -k gui/$(id -u)/com.jarvis.server   # restart after code change
```

## Constraint

LaunchAgent only runs while the user is **logged in**. Fine for an always-on
iMac, but for true headless boot you'd need either auto-login or a
LaunchDaemon (root, system-wide). No passwordless sudo on this machine, which
constrains LaunchDaemon installation.

## Loaded jobs (current)

- `com.jarvis.server` — main FastAPI server. See [[jarvis-server]].
- `com.jarvis.sara` — Sara mail-sweep job. See [[sara-mail-agent]].
- `com.jarvis.duckdns` — **unloaded**, dormant. Was the DDNS updater for the abandoned port-forward path. See [[2026-05-15-access-local-only]].

## Logs

`~/Library/Logs/jarvis.out.log` and `jarvis.err.log` for the main server.

## Python gotcha

The loaded service uses **`.venv/bin/python`**, which is **Python 3.9.6** —
`scripts/start.sh` creates the venv with `python3 -m venv .venv`, and on this
Mac `python3` is `/Library/Developer/CommandLineTools/usr/bin/python3` =
3.9.6. There is no `.venv` 3.12 (that was either abandoned or never existed —
earlier memory was misleading). Keep service-loaded code 3.9-compatible
(`from __future__ import annotations`).

<!-- Seed source: Claude Code memories `jarvis-startup`, `virtueel-kantoor`. -->
