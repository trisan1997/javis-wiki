---
title: Voice — TTS & STT
type: concept
status: live
updated: 2026-08-17
sources: 0
tags: [voice, tts, stt, edge-tts, whisper, elevenlabs, sara]
---

# Voice — TTS & STT

How agents speak (TTS) and hear (STT) in Javis 2.0.

## TTS (active) — edge-tts

Since **2026-05-15**. `/api/speak` in `src/server.py` uses Microsoft's free
neural `edge-tts` (no key, no quota).

- Voices: `EDGE_VOICES = {jarvis: nl-BE-ArnaudNeural, zoe: nl-BE-DenaNeural, …}`. Extended for `steve_jobs` / `nick` / `siebert` / `tristan` / `valentina` / `interim` (see [[habbo-hq]]).
- Jarvis rate `-10%` (Tristan wanted Jarvis slower). Zoë `+0%`.
- Buffers full clip then returns `Response`. On any failure → `HTTPException(502)` → frontend does ONE consistent browser-TTS fallback.
- Frontend `static/js/voices.js` `playbackRate = 1.0` (was `1.35` for ElevenLabs — chipmunky on neural).
- Requirement: `edge-tts>=6.1.0` (installed 7.2.8). ElevenLabs no longer called.
- Change voices: edit `EDGE_VOICES`. Hard-refresh page/PWA to pick up new `voices.js`.

## STT (active) — faster-whisper

Since **2026-05-15**. `/api/transcribe` uses offline **faster-whisper**.

- Model: `small`, cpu, int8. Lazy singleton `_get_whisper()`. Runs in `run_in_threadpool`. Decodes WAV via bundled PyAV (no system ffmpeg).
- Warm latency ~1s/short clip. First call downloads ~480MB model to `~/.cache/huggingface`.
- Returns `{"text":""}` on failure (no 500).
- Accuracy/speed: swap model size in `_get_whisper()` (`base` faster, `medium` better).
- Requirement: `faster-whisper>=1.0.0`.

### Root cause this replaced

`speech_recognition.recognize_google` shipped an **x86-only `flac-mac`**
binary → "Bad CPU type in executable" on this arm64 Mac → every server
transcription `500`'d. (iPhone uses this server path; desktop uses browser
`webkitSpeechRecognition`.)

Related fix to `BasicAuthMiddleware` in [[jarvis-server]]: 401 on a
POST-with-body desynced uvicorn ("Expected ASGI message 'http.response.body'").
Now drains request body + sends `Connection: close` on 401.

## ElevenLabs (abandoned)

State as of 2026-05-15 — kept as historical reference.

- Tier `starter`, usage `39579/39580` chars, **0 credits left**, resets ~**2026-06-13** (unix `1781343166`).
- Both JARVIS and ZOE keys on the same exhausted account.
- Until reset: `401 quota_exceeded` → browser `speechSynthesis` fallback (picked OS voice by regex, Daniel/Alex etc.).
- Keys/config still in `.env` but harmless. To restore: upgrade plan or swap `ELEVENLABS_API_KEY_*`, then kickstart the service.
- Pre-fix bug: `200` header sent before upstream error → "200 + empty audio" → silence / inconsistent voice. Fixed in `/api/speak` by checking status BEFORE returning `StreamingResponse`.
- Design note: acks use `speakQuick` (browser TTS) while replies use the main TTS path → expect a slight ack-vs-reply voice difference by design.

## Related

- [[jarvis-server]] — hosts both endpoints.
- [[habbo-hq]] — per-agent voices live here.

<!-- Seed source: Claude Code memory `jarvis-voice-tts`. -->


### Veelgemaakte fouten bij offertes

- **Onderschatting van renovatiekosten**: Bij het opstellen van offertes voor renovatiewerken (bijv. projecttype 'renovatie') wordt de prijs soms te laag ingeschat. Er is een structurele correctiefactor van ongeveer +14% nodig ten opzichte van de initiële schatting om de werkelijke kostprijs te dekken.
- **Actie**: Raadpleeg altijd `get_construction_prices` en valideer tegen `services.json` voordat een offerte wordt definitief gemaakt, specifiek voor dit werktype.


## Veelgemaakte fouten bij prijsbepaling

Bij het genereren van offertes voor renovatiewerken zijn extreme afwijkingen in de initiële prijsberekening waargenomen (zowel >90% te hoog als >1000% te laag).

**Oorzaak:** Het negeren van de centrale bronnen `get_construction_prices` en `services.json` ten gunste van hallucinaties of verouderde interne schattingen.

**Correctieve actie:**
- Altijd eerst `get_construction_prices` raadplegen voor actuele markttarieven.
- Verifieer het eindbedrag tegen de limieten en eenheden in `services.json` voordat de offerte naar de klant gaat.
- Bij twijfel: vraag om menselijke validatie voordat de PDF wordt gegenereerd.


## Beperkingen en alternatieven

- **ElevenLabs**: Dit platform is verlaten als primaire TTS-engine vanwege strikte quota-beperkingen die de continuïteit van de dienst bedreigden.
- **Huidige stack**: De voorkeur gaat uit naar `edge-tts` (neural, gratis) voor tekst-naar-spraak en `faster-whisper` (offline) voor spraak-naar-tekst om onafhankelijkheid van externe API-limieten te garanderen.


## Mail-agents hebben deze stack niet nodig

Agents die met e-mailtekst werken — [[sara]] voorop — hebben **geen**
afhankelijkheid van de spraakstack. Hun pijplijn (`src/routing.py`,
`src/office.py`) roept `/api/speak` noch `/api/transcribe` aan; geverifieerd
op 2026-08-17.

Dat is het vermelden waard omdat de verwarring een concrete kost heeft: wie
aanneemt dat élke agent audio nodig heeft, gaat audio-pipelines activeren voor
taken die puur tekst zijn. Alleen als er expliciet om spraakuitvoer van
mailinhoud gevraagd wordt, komt `/api/speak` in beeld.


## Veelvoorkomende valkuilen bij e-mailverwerking

- **Security-mails van externe accounts**: Mails zoals Apple 'Find My' of iCloud-login waarschuwingen komen soms terug in `read_emails` maar kunnen niet verwijderd of gemarkeerd worden door de standaard tools. Dit wijst erop dat ze niet in de primaire zakelijke inbox staan, maar in een gekoppeld persoonlijk account (bijv. Gmail) of een andere map (Spam/Archief).
  - *Oplossing*: Gebruik `search_emails` specifiek per account om de bron te lokaliseren voordat verwijdering geprobeerd wordt. Forceer geen delete-actie als de tool 'niet gevonden' meldt om eindeloze loops te voorkomen.
- **CRM-leads intact laten**: Leads van specifieke bronnen (zoals Bobex) moeten bewust in de INBOX gelaten worden voor ophaling via het CRM-systeem door de gebruiker. Automatische verwerking of archivering hiervan vermijden.


### ### Alternatieven en afwegingen
**ElevenLabs (Verworpen):**
Eerder overwogen voor hoogwaardige stemmen, maar afgezien vanwege strikte quota-beperkingen die onvoldoende schaalbaarheid boden voor continue 'always-on' diensten. De kosten-batenverhouding was ongunstig vergeleken met de gratis, offline beschikbare opties.

**Huidige Standaard:**
- **TTS:** `edge-tts` (Neural voices, gratis, geen quota).
- **STT:** `faster-whisper` (Lokaal draaiend, privacy-vriendelijk).
