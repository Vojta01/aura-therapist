# Aura — AI Coach & Therapist Bot for Telegram

<p align="center">
  <img src="docs/images/aura-banner.png" alt="Aura Banner" width="600">
</p>

<p align="center">
  <b>An autonomous AI coach that runs 24/7 on n8n — empathetic conversations, proactive check-ins, and weekly growth summaries.</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License: MIT">
  <img src="https://img.shields.io/badge/platform-n8n-orange.svg" alt="Platform: n8n">
  <img src="https://img.shields.io/badge/AI-Gemini%202.5%20Flash-brightgreen.svg" alt="AI: Gemini">
  <img src="https://img.shields.io/badge/messaging-Telegram-blue.svg" alt="Messaging: Telegram">
</p>

---

## What is Aura?

Aura is a personal AI coach and therapeutic companion that lives in Telegram. Built entirely as **n8n workflows** (no custom code server needed), she provides:

- 🧠 **Intelligent conversations** — multi-modal chat (text, voice, images) powered by Google Gemini
- 🌅 **Proactive morning check-ins** — notices when you've been away and gently reaches out
- 📊 **Weekly growth summaries** — condenses your week's conversations into an insightful narrative

Aura adapts her tone dynamically: empathetic when you're struggling, challenging when you're making excuses, and strategic when you're planning your next move.

---

## Features

| Feature | Workflow | Trigger | Description |
|---|---|---|---|
| 💬 **Smart Chat** | `aura-chat` | Telegram message | Multi-modal conversations with context from your diary & profile. Handles text, voice memos, and images. |
| ☀️ **Proactive Check-in** | `aura-proactive` | Daily schedule | Checks your last active date. If you haven't messaged in 2+ days, sends a personalized nudge referencing your last conversation. |
| 📝 **Weekly Summary** | `aura-weekly-maintenance` | Weekly schedule | Reads the week's conversation history, creates a deep psychological narrative for long-term memory, archives the raw diary, and sends you a friendly summary. |

---

## Architecture

Aura runs entirely on **n8n** — a self-hosted workflow automation platform. No custom backend, no databases to manage. Just 3 workflows wired together through Google Docs (as a lightweight document store) and Google Sheets (for activity tracking and token caching).

```
┌─────────────────────────────────────────────────────┐
│                     Telegram                        │
│  User sends text / voice / image                    │
└──────────────┬──────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│  aura-chat (main workflow)                          │
│                                                     │
│  Telegram Trigger → Switch (text/voice/image)       │
│     ├─ Text → Merge                                 │
│     ├─ Voice → HTTP Request → Transcribe → Merge    │
│     └─ Image → Gemini Vision → Merge                │
│                         │                           │
│   Live Diary ◄─────────▼──────── User Profile       │
│                         │                           │
│                    AI Agent (Gemini)                 │
│                         │                           │
│                    Cache Log (Sheets)                │
│                         │                           │
│                   Telegram Reply                     │
│                         │                           │
│              Update Diary + Activity Sheet           │
└─────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│  aura-proactive (daily, 10:00)                      │
│                                                     │
│  Schedule → Read Activity Sheet → Check days        │
│     └─ If >2 days → Read Diary → AI drafts nudge    │
│        └─ Send Telegram message                     │
└─────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│  aura-weekly-maintenance (weekly)                   │
│                                                     │
│  Schedule → Read Diary → Copy to full history       │
│     ├─ AI: Deep psychological summary               │
│     ├─ Clear diary (fresh start)                    │
│     └─ AI: User-friendly Telegram summary           │
└─────────────────────────────────────────────────────┘
```

See [ARCHITECTURE.md](docs/ARCHITECTURE.md) for detailed node-by-node breakdown and Mermaid diagrams.

---

## Tech Stack

- **[n8n](https://n8n.io)** — Workflow automation (self-hosted)
- **[Google Gemini](https://ai.google.dev)** — LLM for chat, vision analysis, and speech-to-text
- **[Telegram Bot API](https://core.telegram.org/bots/api)** — Messaging interface
- **[Google Docs](https://docs.google.com)** — Lightweight document store (diary, user profile, full history)
- **[Google Sheets](https://sheets.google.com)** — Activity tracking and API cache logging
- **[Google Drive](https://drive.google.com)** — File management for diary rotation

---

## Getting Started

### Prerequisites

- An **n8n instance** (self-hosted or [n8n Cloud](https://n8n.io/cloud))
- A **Telegram Bot** (create via [@BotFather](https://t.me/BotFather))
- A **Google Cloud project** with these APIs enabled:
  - Google Docs API
  - Google Sheets API
  - Google Drive API
  - Gemini API (Generative Language API)
- **Google OAuth 2.0 credentials** (for Docs/Sheets/Drive)

### Quick Setup

1. **Import the workflows** into n8n:
   - Go to n8n → **Import** → upload the 3 JSON files from `workflows/`

2. **Configure credentials** in n8n for each service:
   - Telegram API
   - Google Gemini (PaLM) API
   - Google Docs OAuth2
   - Google Sheets OAuth2
   - Google Drive OAuth2

3. **Create your Google documents**:
   - Create a Google Doc for the Live Diary → copy its ID → set `YOUR_LIVE_DIARY_DOC_ID`
   - Create a Google Doc for the User Profile → copy its ID → set `YOUR_USER_PROFILE_DOC_ID`
   - Create a Google Doc for Full History → copy its ID → set `YOUR_FULL_HISTORY_DOC_ID`
   - Create a Google Sheet for Activity Tracking → copy its ID → set `YOUR_ACTIVITY_SHEET_ID`
   - Create a Google Sheet for Cache Log → copy its ID → set `YOUR_CACHE_LOG_SHEET_ID`

4. **Find & replace** all `YOUR_` placeholders in the 3 workflow JSONs with your actual IDs.

5. **Set your Telegram chat ID** — replace `YOUR_TELEGRAM_CHAT_ID` with your numeric chat ID.

6. **Activate** all 3 workflows in n8n.

Detailed step-by-step instructions: [docs/SETUP.md](docs/SETUP.md)

---

## Workflow Overview

| Workflow | File | Trigger | Schedule |
|---|---|---|---|
| Chat | `workflows/aura-chat.json` | Telegram webhook | Always on |
| Proactive Check-in | `workflows/aura-proactive.json` | Schedule | Daily at 10:00 |
| Weekly Maintenance | `workflows/aura-weekly-maintenance.json` | Schedule | Weekly |

### Message Routing

The chat workflow handles multiple input types:

- **Photos** → Gemini Vision analysis → merged with caption as context
- **Voice messages** → Downloaded via Telegram API → transcribed via Gemini → merged as text
- **Text** → Directly merged with diary and profile context
- **`pro:` prefix** — If the user starts a message with `pro:`, Gemini 2.5 Pro is used instead of Flash (for deeper reasoning)

### Context Window

Every message includes:
1. The user's latest message
2. The full **Live Diary** (recent conversation history)
3. The **User Profile** (persistent facts, preferences, patterns)

This gives the AI rich context without requiring a vector database.

---

## Customizing the AI Persona

The system prompts are embedded in the workflow JSON files. To customize Aura's personality:

1. Open the workflow in n8n editor
2. Find the **AI Agent** node
3. Edit the `system_instruction` text inside the `jsonBody` parameter

Key customization points:
- The user's name, context, and background
- The three internal states (you can add, remove, or rename them)
- Communication rules and tone
- The `pro:` prefix behavior (you can change the trigger or models used)

---

## Project Structure

```
aura-therapist-bot/
├── README.md
├── LICENSE
├── workflows/
│   ├── aura-chat.json              # Main conversational workflow
│   ├── aura-proactive.json         # Daily proactive check-in
│   └── aura-weekly-maintenance.json # Weekly summary + diary rotation
├── docs/
│   ├── ARCHITECTURE.md             # Detailed architecture & node breakdown
│   ├── SETUP.md                    # Step-by-step setup guide
│   └── images/
│       └── aura-banner.png         # Banner image (add your own)
└── sanitize.py                     # Script used to clean personal data (dev reference)
```

---

## Privacy & Self-Hosting

Aura runs on **your own infrastructure**. No third-party service has access to your conversations or personal data. Everything stays within:

- Your n8n instance
- Your Google Drive (diary, profile)
- Your Google Sheets (activity log, API cache)
- Direct API calls to Gemini (your own API key)

The Telegram bot token, Google credentials, and Gemini API key are stored **encrypted** in n8n's credential store — they never appear in the exported workflow JSONs.

---

## FAQ

**Q: Why n8n and not a custom backend?**
A: n8n provides visual workflow orchestration, built-in credentials management, error handling, retries, and scheduling — all without writing server code. It's perfect for AI agent pipelines.

**Q: Why Google Docs as a database?**
A: For a personal single-user bot, Google Docs is free, accessible from anywhere, and natively understood by Gemini's long-context window. No need for a vector database at this scale.

**Q: Can I use a different LLM?**
A: Yes! Replace the Gemini nodes with OpenAI, Anthropic, or any other LLM supported by n8n. The architecture stays the same.

**Q: How do I change the proactive check-in timing?**
A: Edit the Schedule Trigger node in `aura-proactive.json` — you can change the hour, add multiple times, or switch to a different schedule.

---

## Contributing

This is a portfolio project, but suggestions and improvements are welcome! Open an issue or submit a PR.

## License

MIT — see [LICENSE](LICENSE) for details.
