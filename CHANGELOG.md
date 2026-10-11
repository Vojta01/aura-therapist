# Changelog

All notable changes to the Aura workflow collection are documented here.

## 2026-10-11 — Aura chat: empty-response retry + plain-text reply fixes

### Added

- **Empty response check → Attempt count → Attempt limit → Fallback notice** — the chat workflow
  now validates every model reply. On an empty response it retries the model call (up to 3 attempts
  total); after the third empty reply it sends a Telegram warning instead of failing silently.
- System prompt instruction: *"Always write a full reply. Never send an empty message."*

### Changed

- **Send a text message1**: HTML `parse_mode` removed — replies are sent as plain text, so stray
  formatting characters can no longer make the Telegram call fail.
- All three workflow exports were regenerated from the live instance and fully re-sanitized
  (`aura-chat.json`, `aura-proactive.json`, `aura-weekly-maintenance.json`). The previous
  proactive and weekly exports were not valid JSON (unescaped multi-line prompts).

### Why

- DeepSeek occasionally returns **HTTP 200 with empty `content`** (reasoning-only or filtered
  output). The Telegram node then failed with *"message text is empty"* while the workflow still
  recorded a success — no reply reached the user.
- Observed in 2 of the last 25 runs (latest: 2026-10-11 15:11 CEST). Re-sending the same message
  usually passes; the retry loop now does that automatically.
- Verified against a mock API: 4/4 scenarios (empty-then-recovered, always-empty, no-error,
  recovery on the third attempt).

### Rollback

- The retry is additive — the model call itself is unchanged. To drop it, restore `aura-chat.json`
  from Git (`b67ea7b`) and re-import in n8n.

### Repo hygiene

- Gemini rollback export (`aura-chat-gemini-2026-09-09.json`): the personal prompt was replaced
  with the generic English prompt; personal IDs (documents, sheets, chat id, bot token) replaced
  with `YOUR_*` placeholders.

## 2026-09-10 (3) — Aura chat: model switch DeepSeek Flash → DeepSeek V4 Pro

- **AI Agent** node: request body `"model"`: `deepseek-flash` → `deepseek-v4-pro`
- `max_tokens` stays 16384; URL, `messages`, `temperature: 0.7` and credentials untouched.

### Why / naming

- Requested as "DeepSeek 4.0 Pro". **There is no 4.0 variant in the API** — `GET /models` returns exactly
  `deepseek-flash` and `deepseek-v4-pro`, and calls with `deepseek-4.0-pro` / `deepseek-v4.0-pro` fail with
  HTTP 400: *"The supported API model names are deepseek-flash, deepseek-v4-pro"* (verified 2026-09-10).
- `deepseek-v4-pro` is the official stable API ID of **DeepSeek V4 Pro GA** (released 2026-08-13, hosted
  checkpoint `DeepSeek-V4-Pro-0813`, 1.6T total / 49B active params, 1M context, up to 384K output).

### Rollback

- **Back to Flash:** set `"model"` to `deepseek-flash` in the **AI Agent** node
  (Git: previous state = `6d138b1`).

## 2026-09-10 (2) — Aura chat: max_tokens 8192 → 16384

- **AI Agent** node: request body `"max_tokens"`: 8192 → 16384
- Nothing else touched (model stays `deepseek-flash`, URL, `messages`, `temperature: 0.7`).

### Why

- `max_tokens` caps **generated (output) tokens only** — the prompt is *not* counted against it
  (input + output together are bounded by the model's context window). But DeepSeek V4 Flash runs
  with thinking enabled by default, and **reasoning tokens are drawn from that same output budget**.
- Measured on the native API (2026-09-10): a trivial prompt generated 200 tokens of which **163 were
  reasoning** (82 %); with `max_tokens: 64` the model returned `finish_reason: "length"` and an
  **empty reply**. The API accepts large limits without complaint (16K / 32K / 64K all tested OK).

### Rollback

- Set `"max_tokens"` back to 8192 in the **AI Agent** node (Git: previous state = `9614052`).

## 2026-09-10 — Aura chat: model switch DeepSeek V4 Pro → DeepSeek Flash

The `Aura` chat workflow's **AI Agent** node now calls model `deepseek-flash`
(DeepSeek V4 Flash family — the only Flash model ID served by the DeepSeek API,
alongside `deepseek-v4-pro`) instead of `deepseek-v4-pro`.

### Changed

- **AI Agent** node: request body `"model"`: `deepseek-v4-pro` → `deepseek-flash`
- **AI Agent** credentials: type switched from Header Auth to **Bearer Auth**
  (`httpBearerAuth`, value = bare DeepSeek API key). The stale `httpHeaderAuth`
  credential reference was removed.
- Nothing else changed — URL (`https://api.deepseek.com/chat/completions`),
  `messages` payload, `temperature: 0.7`, `max_tokens: 8192`, response parsing
  (`choices[0].message.content`) and the Cache log columns all stay the same.

### Why

- Flash output is ~3× cheaper than Pro ($0.28 vs $0.87 per 1M tokens) and it scores
  better than Pro Preview on agent benchmarks. Pro stays available.

### Rollback

- **Back to Pro:** in n8n open **AI Agent** → change `"model"` to `deepseek-v4-pro` → save.
  (Git: the Pro state is commit `98e9c32`.)
- **Back to Gemini:** `workflows/aura-chat-gemini-2026-09-09.json` → n8n: Workflows → ⋯ → Import from File, then activate.

### Setup required after import

- Create an n8n credential of type **Bearer Auth** (value = DeepSeek API key **without**
  the `Bearer ` prefix) and assign it to the **AI Agent** node (placeholder
  `YOUR_DEEPSEEK_CREDENTIAL_ID`).

### Note

- `max_tokens` was left at 8192 in this change; it was raised to 16384 the same day — see the entry above.

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
