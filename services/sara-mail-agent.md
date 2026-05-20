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
