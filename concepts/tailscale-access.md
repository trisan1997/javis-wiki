---
title: Tailscale access
type: concept
status: live
updated: 2026-05-20
sources: 0
tags: [tailscale, network, remote-access]
---

# Tailscale access

Primary remote-access method for the iMac running Jarvis. Replaces the
abandoned port-forward / custom-domain path (see
[[2026-05-15-access-local-only]]).

## URL

`https://imac-van-tristan.tail5dedea.ts.net`

## Setup

```bash
tailscale serve --bg https+insecure://localhost:8080
```

- Tailnet-only — devices on Tristan's tailnet can reach it; nothing public.
- Valid cert (Tailscale handles the cert, so no browser warning unlike the LAN address `https://192.168.129.33:8080`).
- Serve enabled in Tailscale admin console.
- Persists across reboots — confirmed LIVE since 2026-05-15.

## What it points to

The same FastAPI server as the LAN address — see [[jarvis-server]]. Behind the
same HTTP Basic auth (`APP_USERNAME` / `APP_PASSWORD` in `.env`).

<!-- Seed source: Claude Code memory `jarvis-startup`. -->
