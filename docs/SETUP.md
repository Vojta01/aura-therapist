# Setup Guide — Aura AI Coach

Step-by-step instructions to get Aura running on your own infrastructure.

---

## 1. Prerequisites

### 1.1 n8n Instance

You need a running n8n instance. Options:

- **Self-hosted:** [n8n Docker](https://docs.n8n.io/hosting/installation/docker/) (recommended)
- **n8n Cloud:** [n8n.io/cloud](https://n8n.io/cloud/) (paid, easier)

### 1.2 Telegram Bot

1. Open Telegram and message [@BotFather](https://t.me/BotFather)
2. Send `/newbot` and follow the prompts
3. Save the **bot token** (looks like `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`)
4. Message your new bot on Telegram (you need at least one message to get your chat ID)

### 1.3 Get Your Telegram Chat ID

1. Send any message to your bot
2. Visit: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
3. Find `"chat":{"id":` — that's your numeric chat ID

### 1.4 Google Cloud Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project (or use existing)
3. Enable these APIs:
   - **Google Docs API**
   - **Google Sheets API**
   - **Google Drive API**
   - **Generative Language API (Gemini)**
4. Create OAuth 2.0 credentials:
   - Go to **APIs & Services → Credentials**
   - Create **OAuth client ID** (Desktop application type)
   - Download the JSON credentials file
5. Get a Gemini API key:
   - Go to [Google AI Studio](https://aistudio.google.com/apikey)
   - Create API key

---

## 2. n8n Credential Setup

In n8n, go to **Settings → Credentials** and create these:

| Credential Type | Name | What to Enter |
|---|---|---|
| **Telegram API** | `Telegram account` | Bot token from @BotFather |
| **Google Gemini(PaLM) API** | `Google Gemini API account` | Gemini API key from AI Studio |
| **Google Docs OAuth2** | `Google Docs account` | OAuth2 client ID + secret |
| **Google Sheets OAuth2** | `Google Sheets account` | OAuth2 client ID + secret |
| **Google Drive OAuth2** | `Google Drive account` | OAuth2 client ID + secret |

> 💡 **Tip:** You can use the same OAuth2 credentials for Docs, Sheets, and Drive — just create one credential and reuse it. Name them consistently (e.g., `Google account`) so the workflows find them.

---

## 3. Google Document Setup

Create these documents in your Google Drive:

### 3.1 Live Diary (Google Doc)

1. Create a new Google Doc → name it `Aura_live_diary`
2. Add some initial content (e.g., `-`)
3. Copy the **document ID** from the URL:
   - `https://docs.google.com/document/d/1SNg0RX0RkNB.../edit`
   - The ID is: `1SNg0RX0RkNB...`

### 3.2 User Profile (Google Doc)

1. Create a new Google Doc → name it `Aura_user_profile`
2. Write a profile for the AI to reference (example below)
3. Copy the document ID

**Example User Profile:**
```
Name: [Your Name]
Age: [Your Age]
Goals: [Your current goals and aspirations]
Key Relationships: [People important to you]
Current Challenges: [What you're working through]
Preferences: [How you like to be coached — direct, gentle, etc.]
Background: [Relevant life context]
```

### 3.3 Full History (Google Doc)

1. Create a new Google Doc → name it `Aura_full_history`
2. Leave it empty (or add `# Full History`)
3. Copy the document ID

### 3.4 Activity Tracking (Google Sheets)

1. Create a new Google Sheet → name it `Aura_activity`
2. Add headers in row 1: `Last active date` | (leave column B for `row_number`)
3. Add one data row: enter anything in `Last active date`, `1` in `row_number`
4. Copy the sheet ID

### 3.5 Cache Log (Google Sheets)

1. Create a new Google Sheet → name it `Gemini_cache_log`
2. Add headers in row 1: `Timestamp` | `Model` | `Total Tokens` | `Cached tokens` | `HIT / MISS`
3. Copy the sheet ID

---

## 4. Workflow Import & Configuration

### 4.1 Import Workflows

1. In n8n, go to **Workflows → Import from File**
2. Import all 3 JSON files from the `workflows/` directory

### 4.2 Replace Placeholders

Open each workflow in the n8n editor and replace these placeholders:

| Placeholder | Replace With |
|---|---|
| `YOUR_LIVE_DIARY_DOC_ID` | Live Diary Google Doc ID |
| `YOUR_USER_PROFILE_DOC_ID` | User Profile Google Doc ID |
| `YOUR_FULL_HISTORY_DOC_ID` | Full History Google Doc ID |
| `YOUR_ACTIVITY_SHEET_ID` | Activity Tracking Google Sheet ID |
| `YOUR_CACHE_LOG_SHEET_ID` | Cache Log Google Sheet ID |
| `YOUR_TELEGRAM_CHAT_ID` | Your numeric Telegram chat ID |
| `YOUR_BOT_TOKEN` | Your Telegram bot token |

> 💡 **Tip:** Use n8n's find-and-replace (Ctrl+F in the editor) to quickly replace all occurrences.

### 4.3 Assign Credentials

For each node with a credential selector, choose the credential you created in Step 2.

### 4.4 Configure Schedules

- **aura-proactive:** The Schedule Trigger defaults to daily at 10:00. Adjust if needed.
- **aura-weekly-maintenance:** Defaults to weekly. You can set the exact day/hour.

---

## 5. Test & Activate

### 5.1 Test the Chat

1. Activate `aura-chat` workflow
2. Send a message to your Telegram bot
3. Check n8n's **Executions** tab to verify it ran successfully
4. You should receive a reply on Telegram

### 5.2 Test Proactive Check-in

1. Manually set the "Last active date" in your Activity Sheet to 3+ days ago
2. Go to `aura-proactive` workflow → **Execute Workflow** (manual trigger)
3. Verify you receive a nudge message on Telegram
4. Reset the date back to today

### 5.3 Test Weekly Summary

1. Have some conversation history in your Live Diary
2. Go to `aura-weekly-maintenance` → **Execute Workflow** (manual trigger)
3. Verify:
   - The deep summary appears in Full History
   - The Live Diary is cleared (replaced with `-`)
   - You receive a friendly summary on Telegram

### 5.4 Activate All

Once all tests pass, activate all 3 workflows:
- `aura-chat`: Always on (webhook)
- `aura-proactive`: Scheduled (daily)
- `aura-weekly-maintenance`: Scheduled (weekly)

---

## 6. Customization

### Change the AI's Personality

Edit the system prompt in the **AI Agent** node of `aura-chat`:

1. Open the workflow in n8n
2. Click the AI Agent node
3. Find the `jsonBody` parameter → locate the `system_instruction.text` field
4. Modify the prompt to match your preferences

### Change Models

To use OpenAI instead of Gemini:

1. Replace the HTTP Request node (AI Agent) with n8n's native **OpenAI** node
2. Update the proactive and weekly workflows to use the OpenAI Chat Model node
3. Adjust the `pro:` prefix logic

### Add More Features

The modular design makes it easy to extend:
- Add a **Stable Diffusion node** for image generation
- Add a **Notion node** as an alternative document store
- Add a **Webhook node** to trigger from other apps

---

## 7. Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| "Workflow execution failed" | Missing/expired credentials | Re-authenticate OAuth2 credentials in n8n |
| No Telegram reply | Bot token or chat ID wrong | Verify both in Telegram `getUpdates` API |
| Voice messages not transcribed | MIME type issue | Check the Code node is setting `audio/ogg` |
| Proactive check-in never fires | Activity Sheet `gid=0` mismatch | Verify sheet name in the Google Sheets node |
| Weekly summary empty | Diary doc empty | Ensure Live Diary has conversation history |
| Gemini API errors | Quota or billing | Check Google AI Studio for quota limits |

---

## 8. Production Tips

- **Monitor executions:** n8n shows failed/successful runs in the Executions tab
- **API costs:** Gemini Flash is very cheap; Pro costs more. The `pro:` prefix lets you control when to use it
- **Context caching:** Gemini automatically caches system prompts. Token cache logs in the Cache Sheet show hit/miss rates
- **Backups:** Your Google Docs are backed up by Google. For extra safety, periodically export the Full History doc
