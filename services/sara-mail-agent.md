---
title: Sara — Mail Agent
type: service
status: live
updated: 2026-05-20
sources: 0
tags: [sara, mail, telegram, autonomy]
---

# Sara — Mail Agent

Third Jarvis agent (alongside Jarvis and Zoë). Voice `sara`, page `/sara`.
Manages all mail accounts via Apple Mail with an autonomous hourly sweep.
Tristan gave "go" on **2026-05-17**; the launchd job `com.jarvis.sara` is
loaded and active.

## What she does

- Reads inbox per account in batches (`read_emails`, ~20 mails/round). **Don't** use `search_emails` with sentences — the sweep-prompt is hardened against that.
- Permanently deletes: ads, training mails, security mails.
- No training mails until **2026-11-17** (env `SARA_TRAINING_RESUME_DATE`).

## Architecture

- `POST /api/sara/sweep` starts the round in a **background thread** and returns immediately: `{"status":"started"}` or `{"status":"already_running"}` (concurrency lock).
- Result → `logs/sara_result.log` (`START` / `KLAAR` / `FOUT`) and pushed to Telegram by the server itself.
- Bootstrap marker: `config/.sara_bootstrapped`. Delete it → next sweep redoes bootstrap.

## Telegram (active channel)

- `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID=7855958540` in `.env`.
- Tools: `send_telegram`, `send_telegram_document`, `send_email_attachments_via_telegram`.
- **WhatsApp Business API not set up** — text only, no attachments. Telegram is the channel.
- Two-way: `_tg_poller` in `src/server.py` long-polls `getUpdates`. Address agents by name ("Sara/Valentina/Jarvis, …"); no name = last addressed. Only chat `7855958540` accepted.
- Telegram history is **in-memory per agent** (`tg-<voice>`) — lost on server restart. Open request to make persistent. Same caveat affects Valentina and Jarvis chats.

## Caveats

- macOS Automation permission for **Reminders** missing → `add_reminder` fails → Sara writes a backup note. Tristan must enable in Systeeminstellingen → Privacy → Automatisering.
- `_get_email_folders` / `_move_email_to_folder` fixed (recursive per account, handler-binnen-`tell`).

## Related

- [[jarvis-server]] — Sara runs inside this process.
- [[always-on-launchd]] — launchd pattern for the sweep job.

<!-- Seed source: Claude Code memory `sara-mail-agent` (2026-05-18 snapshot). Verify against current code. -->


### Mappenstructuur en sorteerregels

Sara sorteert inkomende mails aan de hand van een vaste mappenlogica. Onderstaande tabel geeft de belangrijkste regels weer zoals die in de praktijk zijn vastgesteld:

| Afzender / type | Doelmap |
|---|---|
| Apple facturen | `Facturen THC BV/2026` |
| Doccle eBox / FOD Financiën eBox | `e-Box` |
| Doccle Watergroep | `Facturen THC BV/2026` |
| Doccle Acerta (fiche, sociale bijdragen) | `Acerta` |
| Doccle KBC / KBC-berichten | `KBC` |
| Billtobox facturen | `Billtobox` |
| Boekhouder-replies | `Boekhouder` |
| Vertuoza-mails | `Vertuoza` |
| Inkomende klantfacturen (bv. schildersbedrijf) | `inkomen facturen/2026` |
| Zorginstelling — personeelszaken (ziekteattesten e.d.) | `Personeel` |
| Vastgoedpartner / leads | `Klanten/leads` |
| Reclame / commercieel zonder waarde | Definitief verwijderd |
| Security-meldingen (Facebook, Google) | Definitief verwijderd |

**Twijfelgevallen** (gevoelige, persoonlijke of onduidelijke mails) worden nooit autonoom verwerkt maar altijd via Telegram aan Tristan voorgelegd.


### Bekende mailcategorieën en sorteerregels

Sara herkent en verwerkt de volgende terugkerende mailstromen automatisch:

| Afzender / Type | Actie | Doelmap |
|-----------------|-------|---------|
| Apple-facturen | Markeer gelezen + sorteer | Facturen THC BV/2026 |
| Doccle eBox | Markeer gelezen + sorteer | e-Box |
| Doccle Watergroep | Markeer gelezen + sorteer | Facturen THC BV/2026 |
| Doccle Acerta Fiche | Markeer gelezen + sorteer | Acerta |
| Doccle KBC (overeenkomst / bankkaart / afspraak) | Markeer gelezen + sorteer | KBC |
| Reclame / promotiemail | Definitief verwijderen | — |
| Security-notificaties (bv. Google gekoppelde services) | Definitief verwijderen | — |
| Onbekende afzender of onduidelijke inhoud | Via Telegram voorleggen aan Tristan | — |

**Twijfelgevallen:** Sara legt onbekende of ambigue mails voor via Telegram en wacht op instructie voordat er actie wordt ondernomen. Uitstaande twijfelgevallen worden elke ronde herhaald in het journaal totdat Tristan reageert.

**Fallback:** Bij `Apple Mail error -10006` schakelt Sara automatisch over op directe IMAP-toegang. Zie ook [[incidents/2026-05-21-apple-mail-error-10006]].


### Sync Discrepancy Between Read and Write Tools
Er doet zich een terugkerend probleem voor waarbij `read_emails` een lijst van ongelezen berichten retourneert, maar vervolgbewerkingen (`delete_email`, `mark_email_read`, `move_email_to_folder`) falen met de melding "E-mail niet gevonden" voor exact dezelfde items. Dit wijst op een state-desynchronisatie tussen de lees-component en de manipulatietools van de Apple Mail-integratie, mogelijk door account-contextverschillen of caching. Bij constatering dient dit geëscaleerd te worden voor tool-reparatie in plaats van handmatig ingrijpen.


#### Sync-problemen bij verwijderen van spam
Bij het gebruik van `delete_email` kan de tool melden "E-mail niet gevonden", terwijl `read_emails` de berichten nog steeds als ongelezen toont. Dit wijst op een sync-vertraging tussen Mail.app en de onderliggende Gmail-accounts. 

**Actie:** Forceer geen herhaalde verwijderpogingen bij deze foutmelding. Markeer het bericht indien mogelijk als gelezen en log de incidentie voor handmatige inspectie.

#### Kritieke uitsluiting: Bobex-leads
Bobex-leads (herkenbaar aan afzender 'bobex' en onderwerp 'Nieuwe aanvraag') mogen **nooit** automatisch verwerkt, gemarkeerd of verwijderd worden door de agent. 

**Consequence:** Het per ongeluk verwijderen van een lead leidt tot direct verlies van omzet en vereist onmiddellijke escalatie naar de beheerder.


## Protocol voor Bobex-leads

E-mails met het onderwerp "Nieuwe aanvraag voor u op Bobex.be" vormen een speciale categorie die **nooit** automatisch verwerkt, verplaatst of verwijderd mag worden door de agent.

**Regels:**
1. **Lezen maar niet aanraken:** De agent mag deze mails lezen om te tellen hoeveel er zijn, maar mag geen `delete_email`, `mark_email_read` of `move_email` acties uitvoeren op deze items.
2. **Doelgroep:** Deze leads zijn exclusief bestemd voor handmatige ophaling door Tristan via het CRM-systeem.
3. **Rapportage:** Vermeld in het dagjournaal enkel het aantal aangetroffen Bobex-leads (bijv. "16 Bobex-leads intact gelaten") zonder verdere actie te ondernemen.

**Risico:** Automatische verwerking kan leiden tot het missen van commerciële kansen of duplicatie in het CRM.


### Beperking bij verwijdering van gesynchroniseerde oude e-mails

De `delete_email` en `mark_email_read` tools kunnen falen met de melding "E-mail niet gevonden" voor oudere berichten (bijv. >1 jaar oud) die reeds door de lokale mailclient (Apple Mail) zijn gearchiveerd of gesynchroniseerd naar een server-side map die niet schrijfbaar is via de API.

**Symptomen:**
- Mails blijven terugkeren als 'ongelezen' in de uurlijkse sweep.
- Tools rapporteren dat het bericht niet bestaat, terwijl het zichtbaar is in de Mail.app interface.
- Vooral voorkomend bij promotionele/security mails van grote aanbieders (bijv. streamingdiensten, tech-platforms).

**Workaround:**
- Sla deze specifieke berichten over tijdens de automatische sweep om loops te voorkomen.
- Markeer ze handmatig als verwerkt in het logboek zonder ze te verwijderen via de agent.
- Verwijdering dient indien nodig handmatig via de GUI te gebeuren.


### Known Limitations & Multi-Account Handling

De agent werkt momenteel met meerdere mailaccounts (bijv. Gmail vs. Thehandycompany). Er is een bekend patroon waarbij `read_emails` mails uit alle gekoppelde accounts toont, maar `delete_email` faalt met "E-mail niet gevonden" wanneer de target-mail in een ander account staat dan de standaard context van de delete-tool.

**Symptoom:** Oude securitymails of reclame worden wel gelezen maar kunnen niet verwijderd worden.
**Oorzaak:** De delete-tool zoekt standaard in de primaire inbox, terwijl de mail zich in een secundair account bevindt.
**Workaround:** Voorlopig moeten deze mails handmatig geïdentificeerd worden op account-niveau voordat verwijdering geprobeerd wordt; automatische bulk-verwijdering over account-grenzen heen is nog niet robuust.


### ### Bekende beperkingen en multi-account sync
De `delete_email` en `mark_email_read` tools werken soms niet zoals verwacht bij bepaalde security- en reclamemails (bijv. van Apple, Google, LinkedIn). Mails lijken verwijderd maar keren terug bij de volgende leesronde, of worden niet gevonden ondanks dat ze in de ongelezen lijst staan.

**Oorzaak:** Waarschijnlijk een sync-probleem in Mail.app of de aanwezigheid van meerdere mailbox-accounts waarbij de tool standaard slechts één account aanspreekt. Mails kunnen in een ander account staan dan het primaire doelwit.

**Workaround:** Bij hardnekkige mails die niet verwijderen, moet handmatig gecontroleerd worden of deze in een ander account staan. De agent dient per account apart te lezen/verwijderen als de standaardoperatie faalt.


## Beperkingen bij het verwijderen van e-mail

Er is een bekend technisch probleem waarbij bepaalde e-mails (vaak oudere security-meldingen van Apple/Google of nieuwsbrieven) wel zichtbaar zijn via `read_emails`, maar niet verwijderd kunnen worden met `delete_email` of gemarkeerd als gelezen.

**Symptomen:**
- Tool response: "E-mail niet gevonden" ondanks dat de mail in de lijst staat.
- Mails blijven staan als 'ongelezen' na herhaalde pogingen.
- Doet zich vaak voor bij mails uit vorige jaren (bijv. 2025) of specifieke systeemberichten.

**Workaround:**
- Accepteer dat deze specifieke mails niet automatisch verwijderd kunnen worden door de agent.
- Markeer ze niet als 'fout' maar log ze als 'technische beperking mailbox'.
- Waarschuw de gebruiker (Tristan) via Telegram als er meer dan 3 dergelijke mails accumuleren, in plaats van te blijven retryen.


## Bekende beperkingen en valkuilen

Bij het uitvoeren van `delete_email` of `mark_email_read` kunnen bepaalde e-mails (vaak beveiligingsmeldingen of reclame van grote providers zoals Apple, Google, TradeZero) foutief rapporteren als "niet gevonden", zelfs als ze wel terugkomen in `read_emails`.

**Mogelijke oorzaken:**
- De e-mails bevinden zich in een specifieke submailbox die niet standaard wordt doorzocht door de delete-tool.
- Een tijdelijke desynchronisatie met de Apple Mail backend.

**Aanbevolen actie:**
1. Negeer de foutmelding als het gaat om niet-kritieke mail (reclame/security alerts).
2. Start de Mail.app dienst handmatig herstart als het probleem aanhoudt.
3. Verwijder problematische e-mails eventueel handmatig via de UI als automatisering faalt.


### Gmail Delete Synchronization

Er is een bekend probleem waarbij `delete_email` en `mark_email_read` falen met de melding "E-mail niet gevonden" voor oudere spam- of beveiligingsmails, zelfs nadat `read_emails` deze succesvol heeft opgesomd.

**Oorzaak:** Dit wijst waarschijnlijk op een synchronisatie-issue tussen de Apple Mail-implementatie en de onderliggende Gmail-accountstructuur (bijv. mails die alleen in 'Alle berichten' staan maar niet correct gemapped zijn in de lokale mailbox-view die de tools gebruiken).

**Workaround:** Informeer de beheerder (Tristan) als dit patroon zich voordoet bij specifieke afzenders (zoals nieuwsbrieven of security alerts). Handmatige interventie in de web-interface kan nodig zijn om deze items permanent te verwijderen.


### Veiligheidsrestricties en Uitsluitingen

Om data-verlies te voorkomen gelden de volgende absolute restricties voor de `delete_outlook_email` tool:

- **Bobex-leads**: Mails met afzender 'Bobex' of onderwerp 'Nieuwe aanvraag' mogen **nooit** worden verwijderd of verplaatst. Deze moeten intact blijven in de INBOX voor ophaling via het CRM door de beheerder.
- **Controle**: Voordat een delete-actie wordt uitgevoerd, moet de agent expliciet controleren of de mail aan bovenstaande criteria voldoet. Bij twijfel: niet verwijderen.


## Gmail-delete synchronisatie
Bij het verwerken van oudere of ongelezen mails (zoals security alerts of nieuwsbrieven) kan het voorkomen dat de `delete_email` en `mark_email_read` tools fouten geven ("E-mail niet gevonden"), terwijl `read_emails` deze wel toont.

**Oorzaak:** Dit wijst vaak op een synchronisatieprobleem tussen de Apple Mail-implementatie en de onderliggende Gmail-accountstructuur, specifiek bij mappen die niet correct gekoppeld zijn aan de delete-tools.

**Actie:** Als dit patroon zich voordoet (mails blijven terugkomen in elke ronde), moet de beheerder (Tristan) onmiddellijk geïnformeerd worden voor handmatige account-koppeling controle.


### Multi-Account Blind Spots
De agent zoekt en verwijdert e-mails standaard alleen binnen het primaire geconfigureerde account. Security-meldingen (zoals Apple Find My of iCloud login) die in een secundair of vergeten mailbox-account terechtkomen, worden door de sweep niet gevonden en blijven ten onrechte als 'ongelezen' markeren.

**Workaround:** Handmatige verificatie van alle gekoppelde accounts bij hardnekkige 'spook-mails' voordat er wordt aangenomen dat de delete-functie faalt.


### Mail Deletion Discrepancies
Er doet zich regelmatig een fenomeen voor waarbij `read_emails` bepaalde berichten (zoals security-notificaties of reclame) wel toont, maar `delete_email` of `delete_outlook_email` vervolgens melden dat deze "niet gevonden" zijn.

**Mogelijke oorzaken:**
- De mails bevinden zich feitelijk in een ander gekoppeld account dan waar de delete-opdracht op wordt uitgevoerd.
- De mails zijn reeds verwerkt/verplaatst door een ander proces, maar de lokale cache van Apple Mail toont ze nog als aanwezig.
- Sync-vertraging tussen de mailserver en de lokale client.

**Aanbevolen actie:**
Voer bij een "not found" error na een succesvolle read altijd een expliciete refresh van de mailbox uit of controleer de 'Sent'/'Trash' mappen van alle gekoppelde accounts voordat je concludeert dat de mail nog in de inbox staat.


### Mail Handling Protocols

De agent hanteert specifieke regels voor verschillende soorten inkomende e-mail om dubbel werk met andere systemen te voorkomen:

1. **Bobex Leads:** E-mails met afzender 'bobex' en onderwerp 'Nieuwe aanvraag' worden **bewust ongemoeid gelaten** in de INBOX. Deze worden exclusief opgehaald via de externe CRM-tool door de beheerder. Niet markeren, niet verplaatsen, niet verwijderen.
2. **Security Meldingen:** Automatische beveiligingsmails (bijv. Apple Find My, iCloud login) worden na verificatie direct verwijderd om ruis te verminderen.
3. **Vlag-indeling:** De volgende kleurcodering wordt gehanteerd voor visuele organisatie:
   - 🔵 Blauw: Logins
   - 🟢 Groen: Offertes
   - 🔴 Rood: Urgent
   - 🟠 Oranje: Wacht op antwoord
   - 🟡 Geel: Opvolging
   - 🟣 Paars: Facturen te betalen
   - ⚪ Grijs: Referentie


## Beperkingen bij multi-account synchronisatie

De agent draait primair op de standaard zakelijke inbox via Apple Mail. Er is een bekende beperking waarbij security-notificaties van gekoppelde persoonlijke accounts (Apple ID, Gmail) wel zichtbaar zijn in de globale 'unread'-lijst, maar niet bereikbaar zijn voor mutatie-tools (`delete_email`, `mark_email_read`) die op het primaire account gericht zijn.

**Aanbevolen workflow:**
1. Identificeer mails die niet verwijderd kunnen worden ('not found' errors).
2. Vermoed een ander bron-account.
3. Schakel over op account-specifieke zoekopdrachten voordat actie wordt ondernomen.
4. Log de uitzondering en sla over om infinite loops te voorkomen.


### Known Limitations & Sync Issues

Er zijn gevallen waargenomen waarbij bepaalde e-mails (zoals Apple security-meldingen) als 'ongelezen' blijven staan in de interface, maar niet gevonden worden door de `delete_email` of `mark_email_read` acties. Dit wijst op een mogelijke desynchronisatie tussen de lokale cache en de server, of dat de mails reeds verwerkt zijn in een andere map.

**Aanbevolen werkwijze:**
- Als een mail niet gevonden wordt maar wel zichtbaar is als ongelezen: controleer eerst of deze niet reeds gearchiveerd of verplaatst is.
- Forceer geen herhaalde delete-pogingen op dezelfde message-ID binnen korte tijd om race-conditions te vermijden.
- Bij hardnekkige 'spook-mails': maak een screenshot en eskaleer naar de beheerder voor handmatige verificatie van de mappenstructuur.
