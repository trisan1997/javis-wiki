---
title: Remote access — LOCAL-ONLY (Tailscale)
type: decision
status: live
date: 2026-05-15
updated: 2026-05-20
sources: 0
tags: [decision, network, tailscale, port-forward]
---

# Decision — Remote access stays local-only (Tailscale)

## Context

Initial plan was a public custom domain `jarvis.thehandycompany.be` reached via:
- one.com DNS (no DDNS API) → CNAME to DuckDNS `thehandyjarvis.duckdns.org` for the dynamic residential IP.
- Router port-forward TCP 80+443 → `192.168.129.33` (iMac, MAC `d0:11:e5:4e:93:1e`, DHCP reservation required).
- Caddy reverse-proxy (Let's Encrypt cert) in front of the FastAPI server.

The iMac side was fully prepared:
- `bin/caddy` v2.11.3 darwin arm64.
- `config/Caddyfile` (validated; `reverse_proxy https://127.0.0.1:8080` with `tls_insecure_skip_verify`; LE email `Info@thehandycompany.be`).
- `config/com.jarvis.caddy.plist` (LaunchDaemon, root, KeepAlive — not installed).
- DuckDNS updater `scripts/duckdns_update.sh` + LaunchAgent `com.jarvis.duckdns` (RunAtLoad + every 300s, verified resolving).
- No passwordless sudo on this machine.

## Decision

**Local-only access**, via two paths:
1. Home LAN: `https://192.168.129.33:8080` (self-signed, cert warning).
2. Remote (primary): Tailscale `https://imac-van-tristan.tail5dedea.ts.net` — `tailscale serve --bg https+insecure://localhost:8080`. Tailnet-only, valid cert, persists across reboots. LIVE since 2026-05-15. See [[tailscale-access]].

## Why

Proximus b-box port-forward friction. Tailscale was already trusted and gave
valid certs + remote access without exposing anything publicly. Public exposure
wasn't worth the complexity for a single-user/single-household use case.

## Consequences

- DuckDNS LaunchAgent `com.jarvis.duckdns` **unloaded**. CNAME watcher stopped. Both reversible.
- Caddy files (`bin/caddy`, `config/Caddyfile`, `config/com.jarvis.caddy.plist`) remain on disk, **unused / not installed**.
- Dormant leftover: `jarvis.thehandycompany.be` CNAME still at one.com. Harmless.
- **Do NOT resume the port-forward / domain work** unless Tristan explicitly asks.
- Public-internet customers cannot reach the customer offerte page directly. Acceptable for current scope.

## Related

- [[jarvis-server]] — what's being reached.
- [[tailscale-access]] — operational details.
- [[always-on-launchd]] — managing the dormant DuckDNS job.

<!-- Seed source: Claude Code memory `jarvis-startup`. -->
