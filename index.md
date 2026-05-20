# Index

One-line catalog of every page in this wiki. Grouped by directory. Updated on
every ingest. The maintainer reads this first when answering queries.

## Agents
- [[jarvis]] — primary butler; `nl-BE-ArnaudNeural`; sales/offertes
- [[zoe]] — assistant voice (shares prompt with [[valentina]])
- [[sara]] — mail-management agent
- [[valentina]] — waarnemend directeur, orchestrates everyone
- [[steve_jobs]] — dev-team lead (autonomy + Telegram tool-use)
- [[nick]], [[siebert]] — dev-team developers
- **Prompt 3 nieuw — adverteam (Marketing):**
  - [[brand_lead]] (Marlies) — brand lead, kan Pim/Lotte delegeren
  - [[creative]] (Pim) — copy + visuele ideeën
  - [[media_buyer]] (Lotte) — kanaal-budgetten, ROI
- **Prompt 3 nieuw — solo-agents:**
  - [[mario]] (Mario Belastingvrij) — louche boekhouder
  - [[victor]] (Victor Strateeg) — strategisch adviseur

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
