# Aura — AI Coach & Therapist Bot for Telegram

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

## 🔴 The Problem: Why Aura exists

Standard AI chat interfaces (ChatGPT, Gemini Chat, etc.) have a fundamental flaw for ongoing personal work like therapy or coaching:

**Context vanishes.**

Every chat session has a limited context window. After 20–30 exchanges, the AI starts forgetting what you talked about at the beginning of the conversation. Close the tab and the entire session is lost. Open a new one and the AI has no memory of who you are, what you've been through, or what you're working on.

This makes long-term therapeutic work nearly impossible. You can't build on insights from last week. You can't reference a breakthrough from three weeks ago. The AI resets to a blank slate every time.

### How Aura solves it

| Problem | Standard Chat | Aura |
|---|---|---|
| **Conversation history** | Fits in context window (~32K tokens), oldest messages dropped | Written to a **Master Diary** (Google Doc). Every message persists forever. |
| **Cross-session memory** | Each session starts blank | Aura reads the full diary + user profile on every message. You are **never a stranger**. |
| **Multi-modal persistence** | Voice/image context lost after a few messages | Voice transcriptions and image descriptions are stored as text in the diary. |
| **Proactive engagement** | Only responds when you speak | Checks in after inactivity, references your last meaningful topic. |
| **Long-term growth** | No mechanism to condense history | Weekly rotation creates a **structured psychological narrative** — insights compound over time. |

### Additional architectural benefits

- **No vendor lock-in** — all your conversations are plain text in Google Docs, not trapped in a proprietary chat interface. You own your data.
- **Multi-modal memory** — voice messages are transcribed, images are described. Everything becomes searchable text in the diary.
- **Cost-aware persistence** — the weekly diary rotation keeps token usage bounded. Without it, a 6-month diary would cost more to process than the therapy conversation itself.
- **Resilient to outages** — the diary persists in Google Drive independent of any AI service. If Gemini is down, your history isn't lost.
- **Human-readable archive** — the Full History document is a chronological narrative you can read, search, and export. It's not a black box.

---

## Features

| Feature | Workflow | Trigger | Description |
|---|---|---|---|
| 💬 **Smart Chat** | `aura-chat` | Telegram message | Multi-modal conversations with context from your diary & profile. Handles text, voice memos, and images. |
| ☀️ **Proactive Check-in** | `aura-proactive` | Daily schedule | Checks your last active date. If you haven't messaged in 2+ days, sends a personalized nudge referencing your last conversation. |
| 📝 **Weekly Rotation** | `aura-weekly-maintenance` | Weekly schedule | Processes the week's conversations: (1) generates a deep psychological summary for the **Full History** archive, (2) **resets the Master Diary** to keep per-message token costs stable, (3) sends a user-friendly recap via Telegram. Designed primarily as a **token cost control mechanism** — without it, the growing diary would make every message progressively more expensive. |

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

## 💰 Cost Efficiency & Context Caching

Aura is designed with cost efficiency in mind — but real-world LLM usage isn't free. With a daily habit of therapeutic conversations, expect to spend roughly **$8–20/month** (200–500 CZK). Heavy usage months can reach **$40** (1,000 CZK). Three design decisions help keep this in check:

### 1. Gemini Context Caching (best-effort)

Google Gemini automatically caches repeated prompt prefixes, giving a ~90% discount on cached tokens. However, **this cache is short-lived** — typically lasting only **5–10 minutes** between requests (sometimes longer, but not guaranteed). This means:

- The system prompt is cached **only during rapid back-and-forth exchanges** — not on the first message of a session or after a pause
- For most real-world usage (checking in a few times a day), every message starts fresh with **no cache hit**
- A dedicated **Cache Log** Google Sheet tracks every API call with `HIT / MISS`, token counts, and model used — specifically to monitor how often caching actually kicks in

The prompt is still structured with the static system prompt first to maximize cache potential when rapid exchanges do happen, but caching is treated as a bonus, not a guarantee.

### 2. `pro:` Prefix for Model Selection

The user controls which model to use per message:
- **Default** → `gemini-2.5-flash` (faster, almost free)
- **`pro:` prefix** → `gemini-2.5-pro` (deeper reasoning, only when needed)

This avoids paying Pro prices for simple "how are you" exchanges.

### 3. Weekly Diary Rotation

Without intervention, the Live Diary would grow indefinitely — making every API call more expensive over time. The weekly maintenance workflow solves this by:
- Summarizing the week into a condensed psychological narrative (~20-30% of original volume)
- Archiving the full raw diary to a separate **Full History** document
- **Resetting the Live Diary** to a blank state

This keeps the per-message token count stable week after week, while preserving all history in a searchable archive.

---

## 📄 Document System

Aura uses Google Docs as a lightweight, free document store. Each document has a specific role:

| Document | Type | Purpose | Updated |
|---|---|---|---|
| **Master Diary** (`Live Diary`) | Google Doc | The active conversation log for the **current week**. Every user message and Aura response is appended here. Serves as the primary context for the AI. | Every message |
| **User Profile** | Google Doc | Persistent facts about the user — goals, relationships, preferences, patterns. Written once and occasionally updated. Provides stable context across all conversations. | Manual / rarely |
| **Full History** | Google Doc | **Long-term archive.** The weekly maintenance workflow appends a deep psychological summary here, then copies the raw diary before clearing it. Accumulates over time — a searchable record of all past conversations. | Weekly |
| **Activity Sheet** | Google Sheets | Tracks the timestamp of the last user interaction. Checked daily by the proactive workflow to decide whether to send a nudge. | Every message |
| **Cache Log** | Google Sheets | API usage analytics — timestamp, model, total tokens, cached tokens, and cache hit/miss status. Useful for monitoring costs and caching efficiency. | Every message |

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
| Weekly Rotation | `workflows/aura-weekly-maintenance.json` | Schedule | Weekly |

### Message Routing

The chat workflow handles multiple input types:

- **Photos** → Gemini Vision analysis → merged with caption as context
- **Voice messages** → Downloaded via Telegram API → transcribed via Gemini → merged as text
- **Text** → Directly merged with diary and profile context
- **`pro:` prefix** — If the user starts a message with `pro:`, Gemini 2.5 Pro is used instead of Flash (for deeper reasoning)

### Context Window

Every message to the AI includes:

1. **System Prompt** — Aura's persona, rules, and tone (static → cached by Gemini)
2. **User Profile** — Stable facts, goals, and patterns (rarely changes → cached)
3. **Master Diary** — The current week's conversation log (dynamic, grows until reset)
4. **The latest user message** — text, transcribed voice, or image description

See the [Document System](#-document-system) table above for details on each document's lifecycle.

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
│       └── (add your own screenshots here)
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
