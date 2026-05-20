# Index

One-line catalog of every page in this wiki. Grouped by directory. Updated on
every ingest. The maintainer reads this first when answering queries.

## Agents
- [[jarvis]] — primary butler; `nl-BE-ArnaudNeural`, rate `-10%`; **shared-brain: read + journal**
- [[zoe]] — assistant voice; `nl-BE-DenaNeural`, rate `+0%`; **shared-brain: read + journal** (shares prompt with [[valentina]])
- [[sara]] — mail-management agent; voice `sara`; **shared-brain: read + journal**
- [[valentina]] — waarnemend directeur, orchestrates everyone; **shared-brain: read + journal**
- [[steve_jobs]] — dev-team lead, autonomy-only (no tools, no preload)
- [[nick]] — dev-team developer, autonomy-only
- [[siebert]] — dev-team developer, autonomy-only

## Services
- [[jarvis-server]] — always-on launchd FastAPI service on `:8080`; Tailscale-accessible; system Python 3.9
- [[sara-mail-agent]] — third agent; hourly mail-sweep via Apple Mail; Telegram-controlled
- [[habbo-hq]] — virtual office at `/`; autonomy + dev-team + customer offer flow all live

## Concepts
- [[always-on-launchd]] — LaunchAgent + KeepAlive pattern; `JARVIS_NO_RELOAD=1`; `kickstart -k` to deploy
- [[voice-tts]] — TTS = edge-tts (neural, free); STT = faster-whisper offline; ElevenLabs abandoned (quota)
- [[tailscale-access]] — remote access via `imac-van-tristan.tail5dedea.ts.net`; port-forward path abandoned

## Decisions
- [[2026-05-15-access-local-only]] — chose Tailscale over public port-forward + custom domain

## Incidents
- _(none yet)_
