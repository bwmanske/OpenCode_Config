# Version History

Newest first. Each entry describes what changed relative to the previous version.
File locations: `JSON files\opencode-v1.jsonc` … `opencode-v2-5.jsonc`.

## v2-5 — 2026-11-04 (current, verified; deployed 2026-11-06)

Changes relative to v2-4, verified against the official V2 docs (see [docs/WEB_REFERENCES.md](WEB_REFERENCES.md)):

- **`body.includeUsage` → `compatibility.streamUsage: true`** — `body` is sent verbatim in the request body, so `includeUsage` there was a no-op. `compatibility.streamUsage` is the documented control for `stream_options.include_usage` (default is already `true`; kept explicit).
- **`headers.Authorization` workaround → `settings.apiKey: "not-needed"`** — the `Authorization: "Bearer not-needed"` header was an undocumented workaround for V1 `options.apiKey`. `settings.apiKey` is the documented credential path; the AI SDK runtime expects a key at construction, so it is kept even though LM Studio auth is disabled.
- **Removed `websearch: true`** — not a documented form (docs show `false` and `{provider}`), and all web-search providers are cloud services, incompatible with the air-gap intent.
- **Removed redundant `read`/`*.env`/`ask` permission rule** — already included in the V2 built-in base policy.
- **Policies: deny `*`, then allow `lmstudio`** — v2-4 denied only `openai`, `anthropic`, `google`, so the "air-gapped" comment overstated it. Docs: last match wins, no specificity rule, so the broad deny goes first and the exception after. A denied provider disappears from the catalog even with valid credentials.
- **`limit.input` raised `120000` → `205555` (all three models) + `buffer` comment corrected** — the v2-3 change from `buffer: 85555` (absolute) to `"40%"` (percentage) had silently moved the auto-compaction trigger from ~120,000 to ~72,000 tokens, because the percentage applies to `inputWindow = min(limit.input, limit.context)`, not the context window. With `input` below `context` (120000 < 205555), the trigger was 72,000 = 35% of the context window. Raising `input` to 205555 makes the input window equal the context window, so the 40% buffer now triggers at 123,333 tokens = 60% of the context window, matching the original intent. Source-verified formula in [docs/WEB_REFERENCES.md](WEB_REFERENCES.md) (Wrong or ambiguous references).

Deployed to the live global config (`~/.config/opencode/opencode.jsonc`) on 2026-11-06.

## v2-4 — 2026-11-03 (deployed)

First fully valid version; deployed to the live global config (`~/.config/opencode/opencode.jsonc`, byte-identical as of 2026-11-04).

- Fixed v2-3's syntax error: missing comma after `"update": "auto"`.
- Added commented-out `model` lines for quick model switching.
- Added third model `jackrong/qwopus3.8-27b-coder-fast`.

Known issues (all fixed in v2-5): misplaced `body.includeUsage`, `headers.Authorization` workaround, undocumented `websearch: true`, policies air-gap overstatement.

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
