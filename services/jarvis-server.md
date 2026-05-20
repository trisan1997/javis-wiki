---
title: Jarvis Server
type: service
status: live
updated: 2026-05-20
sources: 0
tags: [jarvis, runtime, network, fastapi, uvicorn]
---

# Jarvis Server

The always-on FastAPI/uvicorn process at the heart of Javis 2.0. Serves the UI,
the agent endpoints (Jarvis, Zoë, [[sara-mail-agent|Sara]], Valentina,
dev-team), and the [[habbo-hq|virtual office]] at `/`.

## How it runs

- LaunchAgent: `~/Library/LaunchAgents/com.jarvis.server.plist`, label `com.jarvis.server`.
- Pattern: see [[always-on-launchd]]. `RunAtLoad` + `KeepAlive`.
- Entrypoint: `scripts/start.sh` with env `JARVIS_NO_RELOAD=1` (service mode disables uvicorn `--reload` to avoid log-file restart loops).
- Interactive run: `cd "/Users/tristanklaasse/javis 2.0" && ./scripts/start.sh` (reload enabled in that mode).
- Python: **system Python 3.9** (not the `.venv` 3.12). Keep code 3.9-compatible (`from __future__ import annotations`).
- Logs: `~/Library/Logs/jarvis.out.log`, `~/Library/Logs/jarvis.err.log`.
- Manage: `launchctl unload/load -w <plist>`, `launchctl list | grep jarvis`.
- Restart after code change: `launchctl kickstart -k gui/$(id -u)/com.jarvis.server` — causes brief downtime that also affects the customer offerte page, so **ask Tristan first**.

## Access

- URL: HTTPS on `:8080`, self-signed cert.
- Local LAN: `https://192.168.129.33:8080` (cert warning).
- Remote: `https://imac-van-tristan.tail5dedea.ts.net` via [[tailscale-access|Tailscale]] — primary remote method.
- Health: `GET /healthz` — intentionally unauthenticated for uptime checks.
- Auth: app-wide HTTP Basic via pure-ASGI `BasicAuthMiddleware` in `src/server.py`. Creds: `APP_USERNAME` / `APP_PASSWORD` in `.env`. Pure-ASGI is critical so it doesn't buffer the `/api/speak` audio stream. Empty password disables auth (local-only mode).

## Important pieces in `src/server.py`

- `BasicAuthMiddleware` — pure-ASGI; on a 401-with-body it drains the request body + sends `Connection: close` to avoid a uvicorn desync ("Expected ASGI message 'http.response.body'").
- `/api/transcribe` — offline [[voice-tts|faster-whisper]] (`small`, cpu, int8). Lazy singleton. PyAV decode (no system ffmpeg).
- `/api/speak` — [[voice-tts|edge-tts]] (free neural). Buffers full clip. Raises `HTTPException(502)` on failure so the frontend does ONE consistent browser-TTS fallback.
- `_tg_poller` thread — long-polls Telegram `getUpdates`. Routes "Sara/Valentina/Jarvis, …" commands. **Do not call `getUpdates` manually** — it steals messages from the poller.
- `_clean_speech` — strips symbols/emoji before TTS.

## Network

- iMac wired `en0`, MAC `d0:11:e5:4e:93:1e`, LAN IP `192.168.129.33` (DHCP — needs reservation or it drifts; saw .33/.34).
- Router/admin `192.168.128.1`, subnet `/23` (`255.255.254.0`).
- Public IP `80.201.57.48` (residential, likely dynamic).
- Port-forward path **abandoned** — see [[2026-05-15-access-local-only]].

## Related

- [[always-on-launchd]] — the launchd pattern this service follows.
- [[voice-tts]] — TTS/STT subsystem this server hosts.
- [[tailscale-access]] — how remote access works.
- [[sara-mail-agent]], [[habbo-hq]] — major subsystems running inside this server.

<!-- Seed source: Claude Code memory `jarvis-startup` (2026-05-15 snapshot). Verify against current code before quoting as fact. -->
