---
title: thehandycompany.be — publieke website-architectuur
type: decision
status: decided
date: 2026-05-20
updated: 2026-05-20
sources: 0
tags: [decision, hosting, dns, tailscale, one.com, public]
---

> **Beslissing (2026-05-20, Tristan): Optie B — split frontend op one.com + Tailscale Funnel API.**
> Implementatie staat ingepland voor Prompt 4. CORS, Funnel-enable en
> statisch frontend zijn de drie deliverables die nog ontbreken.

# Decision — Publieke website-architectuur voor thehandycompany.be

## Context

Tristan wil dat klanten via **thehandycompany.be** een autonome offerte
kunnen aanvragen (Jarvis-gespreksinterface + later facturatie). Vereisten:
1. Publieke toegang (klanten zonder Tailscale).
2. Valid HTTPS-certificaat voor `thehandycompany.be` (geen browser-warnings).
3. Mac (`/Users/tristanklaasse/javis 2.0/`) blijft de backend — alles draait
   lokaal, geen cloud-server om te beheren.
4. DNS staat bij one.com (besloten 2026-05-15, geen DDNS-API beschikbaar).

## Context: wat we al hebben uitgesloten

Op **2026-05-15** (zie [[2026-05-15-access-local-only]]) is besloten dat
remote-access via Tailscale loopt (`tailscale serve` — tailnet-only). De
port-forward + Caddy + DuckDNS pijplijn is bewust verlaten wegens
Proximus b-box friction + complexiteit. Caddy-files staan dormant op disk.

Het ACCESS-besluit van mei-15 dekt alleen Tristan/Valentina. Klanten staan
NIET op de tailnet. Daarom is een nieuw architectuur-besluit nodig nu de
publieke offerte-flow concreet wordt.

## Opties

### Optie A — Tailscale Funnel + one.com DNS-CNAME (Tristans eerste keuze)

- `tailscale funnel --bg 443` exposeert Mac publiek via `imac-van-tristan.tail5dedea.ts.net`
- one.com DNS: CNAME `thehandycompany.be → imac-van-tristan.tail5dedea.ts.net`

**Probleem**: Tailscale Funnel's HTTPS-certificaat dekt alleen
`*.tail5dedea.ts.net` — NIET `thehandycompany.be`. Bij een CNAME volgt de
browser het pad maar verifieert het cert tegen de oorspronkelijke hostname.
Resultaat: cert-mismatch waarschuwing in elke klant-browser.

Werkbare submodes:
- A1: Accepteer `imac-van-tristan.tail5dedea.ts.net` als publieke URL. Lelijk + leakt infrastructuur.
- A2: Tailscale Funnel + Caddy/Cloudflare-vóór-Funnel voor cert-termination. Voegt complexiteit toe die we juist verlaten hebben.

### Optie B — Split: one.com statisch frontend + Tailscale Funnel API

- one.com host statisch frontend (HTML/CSS/JS) voor `thehandycompany.be` — valid cert via one.com's eigen TLS.
- Frontend doet `fetch()` calls naar `https://imac-van-tristan.tail5dedea.ts.net/api/*` (Funnel exposeert API).
- CORS in FastAPI configureren voor `Origin: thehandycompany.be`.
- Klant ziet alleen `thehandycompany.be`-URL; cert-warnings alleen voor de API-laag (en dat is OK want de browser legt geen URL daarvan zichtbaar bloot).

**Voordelen**: publieke domein met valid cert, geen port-forward, alle
business-logica blijft op de Mac, FTP-upload-deploy van statisch frontend is
triviaal.

**Nadelen**: dubbele deploy-pijplijn (frontend FTP-upload, backend launchd-kickstart). Statisch frontend kan geen server-side rendering (acceptabel voor SPA).

### Optie C — Cloudflare Tunnel (alternatief)

- `cloudflared` daemon op Mac maakt outbound-tunnel naar Cloudflare.
- Cloudflare termineert TLS voor `thehandycompany.be` met valid cert (Let's Encrypt of Cloudflare-edge).
- Geen port-forward, custom domein werkt out-of-the-box.
- Vereist: domein verhuizen naar Cloudflare DNS (of CNAME setup met validation), `cloudflared` install + auth.

**Voordelen**: één tool doet alles, valid cert voor custom domein, geen extra frontend-laag.

**Nadelen**: nieuwe dependency, eenmalige Cloudflare-setup, één-vendor-lock-in (concept).

## Beslissing

**Pending Tristan's keuze.** Mijn aanbeveling: **Optie B** (split frontend + Funnel API).

Reden: combineert het beste van wat we al hebben (Tailscale werkt al, one.com
host al DNS) zonder nieuwe leveranciers of nieuwe daemons. De dubbele deploy
is geen issue omdat het frontend vrijwel statisch is — wijzigingen daar zijn
zeldzaam.

Optie A werkt alleen met visuele compromis (A1) of teruggekeerd-complexity
(A2). Optie C is sterk maar voegt een nieuwe vendor toe — alleen kiezen als
Tristan dat expliciet wil.

## Te doen (na keuze)

Voor Optie B:
1. CORS-middleware toevoegen in `src/server.py` met allowed origin `https://thehandycompany.be`.
2. Statisch frontend bouwen (kan herbruik van bestaande `static/`-templates).
3. FTP-upload-script in `scripts/`.
4. `tailscale funnel --bg 443` enablen op Mac (env in launchd plist).
5. Bij de eerste klantgesprekken: verifieer end-to-end dat session-history + speak-endpoint werken cross-origin.

## Consequences

- Tailscale Funnel kost niets voor Tristans Personal-plan.
- one.com hosting was er al — geen extra kosten.
- Customer-data (offertes) blijft op de Mac (compliance-vriendelijk).
- Bij Mac-downtime is `thehandycompany.be` bereikbaar (statisch) maar API-calls falen — frontend moet daar elegant mee omgaan.

## Related

- [[2026-05-15-access-local-only]] — vorig access-besluit (interne toegang).
- [[jarvis-server]] — de backend die publiek bereikbaar wordt.
- [[tailscale-access]] — huidige Tailscale-setup (Funnel ≠ Serve).

<!-- Decision page; status=planned tot Tristan een optie kiest. -->
