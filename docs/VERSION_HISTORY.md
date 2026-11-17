# Version History

Newest first. Each entry describes what changed relative to the previous version.
File locations: `JSON files\opencode-v1.jsonc` … `opencode-v2-7.jsonc`.

## v2-7 — 2026-11-17 (current; deployed)

Changes relative to v2-6:

- **Added OpenRouter support** — two new providers, `openrouter-free` and `openrouter-paid` (same `https://openrouter.ai/api/v1` endpoint and key, split to distinguish free from paid models):
  - `openrouter-free`: `nvidia/nemotron-3-ultra-550b-a55b:free` (1M ctx / 65k out), `thinkingmachines/inkling-2-small:free` (1M ctx / 131k out).
  - `openrouter-paid`: `deepseek/deepseek-v4.1-flash-1017` (1M ctx / 943k out).
  - All three declare `capabilities.reasoning` (`supported: true`, effort low/medium/high) — required for custom (non-catalog) models to expose reasoning; limits use the models' real OpenRouter specs with `input` = `context` (matching the local-model pattern).
- **API key protection** — `settings.apiKey` is `"{env:OPENROUTER_API_KEY}"` in both providers. The key lives only in the environment variable, never in any file in this workspace. If the variable is unset, OpenCode substitutes an empty string and OpenRouter requests fail with 401 (local LM Studio models are unaffected).
- **Policies** — added `provider.use` allow rules for `openrouter-free` and `openrouter-paid`; the deny-`*` baseline otherwise hides them from the catalog even with valid credentials.
- **Permissions** — added a `git push origin *` allow rule immediately after the `git push *` deny (last match wins), so pushes to this project's `origin` remote are permitted while pushes to any other remote stay denied.

Deployed to the live global config (`~/.config/opencode/opencode.jsonc`) on 2026-11-17 (byte-identical, SHA256 `FB4D6DC425BB4B8EC4724A22108CCFA4921F8F75ED7D99AC89928E90E788650A`).

**Provenance:** the initial OpenRouter model block was copied from a **Pi** agent `models.json`, whose field names differ from OpenCode's. Conversion applied: Pi `reasoning` (boolean) → OpenCode `capabilities.reasoning` (object); Pi `contextWindow` → `limit.context`; Pi `maxTokens` → `limit.output`; Pi provider `baseUrl` → `settings.baseURL`. Pi's default limit values (128k/16k) were replaced with the models' actual OpenRouter specs. The original `thinkingmachines/inkling:free` ID does not exist on OpenRouter; the `:free` model in that family is `inkling-2-small:free`. The local-model `body.response_format` was not carried over (not a supported parameter on all listed models).

## v2-6 — 2026-11-10 (superseded by v2-7)

Changes relative to v2-5:

- **Restored `websearch: true`** — the v2-5 removal was a regression: after the line was removed, web search stopped working, and re-adding it to the live config restored it. `true` is not a documented form (docs show `false` and `{provider}`) and is a no-op in the current `dev` source — see [docs/WEB_REFERENCES.md](WEB_REFERENCES.md) (Wrong or ambiguous references) for the source-verified behavior and the discrepancy.

The live global config already contained this line (manually re-added by the user after the v2-5 deployment); v2-6 formalizes that edit as a versioned file. It is byte-identical to the live config (verified 2026-11-10, SHA256 `887E5CF98744D6FE7E58ECE2710D5C485CC4ABFFF561F4CC0473694034381063`). v2-6 was not deployed by the assistant — the live config already matched it.

## v2-5 — 2026-11-04 (verified; deployed 2026-11-06, superseded by v2-6)

Changes relative to v2-4, verified against the official V2 docs (see [docs/WEB_REFERENCES.md](WEB_REFERENCES.md)):

- **`body.includeUsage` → `compatibility.streamUsage: true`** — `body` is sent verbatim in the request body, so `includeUsage` there was a no-op. `compatibility.streamUsage` is the documented control for `stream_options.include_usage` (default is already `true`; kept explicit).
- **`headers.Authorization` workaround → `settings.apiKey: "not-needed"`** — the `Authorization: "Bearer not-needed"` header was an undocumented workaround for V1 `options.apiKey`. `settings.apiKey` is the documented credential path; the AI SDK runtime expects a key at construction, so it is kept even though LM Studio auth is disabled.
- **Removed `websearch: true`** — not a documented form (docs show `false` and `{provider}`), and all web-search providers are cloud services, incompatible with the air-gap intent. **Regression:** removing the line stopped web search in practice; restored in v2-6 (see [docs/WEB_REFERENCES.md](WEB_REFERENCES.md)).
- **Removed redundant `read`/`*.env`/`ask` permission rule** — already included in the V2 built-in base policy.
- **Policies: deny `*`, then allow `lmstudio`** — v2-4 denied only `openai`, `anthropic`, `google`, so the "air-gapped" comment overstated it. Docs: last match wins, no specificity rule, so the broad deny goes first and the exception after. A denied provider disappears from the catalog even with valid credentials.
- **`limit.input` raised `120000` → `205555` (all three models) + `buffer` comment corrected** — the v2-3 change from `buffer: 85555` (absolute) to `"40%"` (percentage) had silently moved the auto-compaction trigger from ~120,000 to ~72,000 tokens, because the percentage applies to `inputWindow = min(limit.input, limit.context)`, not the context window. With `input` below `context` (120000 < 205555), the trigger was 72,000 = 35% of the context window. Raising `input` to 205555 makes the input window equal the context window, so the 40% buffer now triggers at 123,333 tokens = 60% of the context window, matching the original intent. Source-verified formula in [docs/WEB_REFERENCES.md](WEB_REFERENCES.md) (Wrong or ambiguous references).

Deployed to the live global config (`~/.config/opencode/opencode.jsonc`) on 2026-11-06; superseded by v2-6 on 2026-11-10 (the live config diverged when the user manually re-added `websearch: true`).

## v2-4 — 2026-11-03 (deployed)

First fully valid version; deployed to the live global config (`~/.config/opencode/opencode.jsonc`, byte-identical as of 2026-11-04).

- Fixed v2-3's syntax error: missing comma after `"update": "auto"`.
- Added commented-out `model` lines for quick model switching.
- Added third model `jackrong/qwopus3.8-27b-coder-fast`.

Known issues: misplaced `body.includeUsage`, `headers.Authorization` workaround, and the policies air-gap overstatement (all fixed in v2-5); `websearch: true` was removed in v2-5, but that removal was a regression (restored in v2-6).

## v2-3 — 2026-11-03

- Added the `aisdk:` prefix to `package` (fixing the v2-1/v2-2 defect).
- Added core settings: `update: "auto"`, `websearch: true`, `snapshots: true`.
- Added the full `permissions` block (shell ask baseline, read-only git/ls exceptions, hard-denies, `external_directory` ask).
- Added `policies` denying `openai`, `anthropic`, `google`.
- `compaction.buffer`: `85555` → `"40%"`.

Defect: missing comma after `"update": "auto"` — invalid JSONC.

## v2-2 — 2026-11-03

- Model `response_format` changed from `json_schema` (generic object) to `text`.
- Still missing the `aisdk:` prefix on `package`.

## v2-1 — 2026-11-02

First V2 conversion of the V1 config:

- `provider` (singular) → `providers` keyed by provider ID.
- `npm: "@ai-sdk/openai-compatible"` → `package` (defect: missing `aisdk:` prefix).
- V1 `options` block → `settings` block (`baseURL`, `timeout`, `chunkTimeout`).
- V1 `options.apiKey` → `headers.Authorization: "Bearer not-needed"` workaround.
- V1 `options.includeUsage` → `body.includeUsage: true` (misplaced; fixed in v2-5).
- Provider-level `limit` moved to per-model `limit`.
- Dropped undocumented V1 model fields (`strictTools`, `experimental_loose_tool_matching`, `extraArgs`) — no-ops in V2.
- Carried over `compaction.buffer: 85555` and the `json_schema` response_format (changed to `text` in v2-2).

## v1 — 2026-11-02

Original V1-format config (migration source):

- Singular `provider` key with `npm: "@ai-sdk/openai-compatible"`.
- `options`: `baseURL` (local LM Studio), `apiKey: "not-needed"`, `includeUsage: true`, `timeout: 12000000`, `chunkTimeout: false`.
- Provider-level `limit` (context 205555 / input 120000 / output 8192).
- Two models (`qwen/qwen3.6-27b`, `qwen/qwen3.8-27b`), each with `strictTools: false`, `experimental_loose_tool_matching: true`, and `extraArgs` forcing `response_format: json_object`.
- `compaction`: `auto: true`, `buffer: 85555`, `keep.tokens: 15000`.
- Root `model: lmstudio/qwen/qwen3.8-27b`.
