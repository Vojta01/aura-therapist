# Changelog

All notable changes to the Aura workflow collection are documented here.

## 2026-09-09 — Aura chat: Gemini → DeepSeek V4 Pro

The `Aura` chat workflow switched from Google Gemini REST API to DeepSeek (always Pro model).

### Changed

- **AI Agent** node:
  - URL: `generativelanguage.googleapis.com/...:generateContent` → `https://api.deepseek.com/chat/completions`
  - Model: fixed `deepseek-v4-pro` (previously toggled `gemini-3.1-pro-preview` / `gemini-3-flash-preview` via `pro:` prefix)
  - Request body converted from Gemini format (`system_instruction` + `contents` + `safetySettings` + `generationConfig`) to OpenAI-compatible `messages` (`system` + `user`), `temperature: 0.7`, `max_tokens: 8192`. Aura's system prompt preserved 1:1.
  - Auth: `googlePalmApi` → `httpHeaderAuth` (`Authorization: Bearer <DeepSeek API key>`)
- **Send a text message1**: response parsing `candidates[0].content.parts[0].text` → `choices[0].message.content`
- **Update a document1**: same response-parsing change (diary append)
- **Cache log** columns rewritten for OpenAI response shape:
  - Model: `modelVersion` → `model`
  - Total Tokens: `usageMetadata.totalTokenCount` → `usage.total_tokens`
  - Cached tokens / HIT-MISS: `usageMetadata.cachedContentTokenCount` → `usage.prompt_tokens_details.cached_tokens`

### Rollback (back to Gemini)

- Gemini version that was running until 2026-09-09 is preserved in
  `workflows/aura-chat-gemini-2026-09-09.json` — in n8n: Workflows → ⋯ → Import from File, then activate.
- Note: the previously committed `workflows/aura-chat.json` carried a stale prompt;
  the backup file above matches exactly what ran in production before the switch.

### Setup required after import

- Create an n8n credential of type **Header Auth** (Name `Authorization`, Value `Bearer <DeepSeek API key>`)
  and assign it to the **AI Agent** node (placeholder `YOUR_DEEPSEEK_CREDENTIAL_ID`).
- Old `aura-chat-gemini-2026-09-09.json` still references the Gemini credential placeholders.
