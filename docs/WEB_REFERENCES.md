# Web References

Docs pages this config and its verification depend on (last checked 2026-11-04). Re-check when OpenCode updates them; each entry lists the claims that would change if the page changes.

This file has two parts:

- **Verified references** — pages whose claims the config relies on, with the specific claim noted.
- **Wrong or ambiguous references** — pages that turned out to be wrong or ambiguous, with the actual behavior recorded in detail (verified against source). If a flagged page is later updated to clarify the problem, move its link back to *Verified references* and remove the detailed note here.

## Verified references

- https://opencode.ai/v2/docs/policies
  - "When several statements match, the last one wins. There is no specificity rule, so put broad statements first and exceptions after them." — basis for the deny-`*`/allow-`lmstudio` order in v2-5.
  - `provider.use` resource is the provider ID; "A denied provider disappears from the catalog and model selection even when it has valid credentials."
  - Wildcards: `*` matches zero or more characters.
  - Precedence: broader configuration (managed/system/console/global) is evaluated later and overrides narrower config; within one file, written order is kept.
- https://opencode.ai/v2/docs/providers
  - `settings` = "Typed OpenCode controls and JSON options passed to the runtime package"; `headers` = HTTP headers; `body` = "JSON fields merged into request bodies" — basis for moving `includeUsage` out of `body`.
  - `settings.apiKey` is the documented credential path — basis for restoring `apiKey` instead of the `Authorization` header.
  - `settings.baseURL`, `settings.timeout`, `settings.chunkTimeout` (Timeouts section).
  - LM Studio built-in discovery at `http://127.0.0.1:1234/v1`.
- https://opencode.ai/v2/docs/models
  - `compatibility.streamUsage` — "Requests token usage in the stream with `stream_options.include_usage`"; defaults to `true` — basis for the v2-5 includeUsage fix.
  - Options merge order: provider first, model second, variant last.
  - LM Studio section: "Omit `apiKey` when LM Studio authentication is disabled" — we keep `apiKey: "not-needed"` because the AI SDK runtime expects a key at construction.
- https://opencode.ai/v2/docs/permissions
  - "The last matching rule wins, so the specific exceptions follow the broad rule." — matches the config comment.
  - Built-in base policy already includes `read`/`*.env`/`ask` — basis for removing the redundant rule in v2-5.
  - No default rule asks for `external_directory` — the config's `external_directory`/`ask` rule is not redundant and stays.
- https://opencode.ai/v2/docs/websearch
  - Documented forms are `websearch: false` and `websearch: {provider}`; `true` is not documented — basis for removing it in v2-5.
  - All five search providers (Exa, Firecrawl, Parallel, Tavily, TinyFish) are cloud services — incompatible with the air-gap intent.
- https://opencode.ai/v2/docs/migrate-v1
  - V1→V2 field conversions used for this config (V1 `options.apiKey` → `settings.apiKey`; V1 `npm` package key → `package` with `aisdk:` prefix).
- https://opencode.ai/v2/docs/config
  - V2 core config reference (locations, precedence, `update`, `snapshots`, `compaction`).
  - "Automatic compaction uses the selected model's usable input budget." — this is the statement that matches the source (see *Wrong or ambiguous references* below for the conflict with the compaction guide).
- https://opencode.ai/docs/config (V1)
  - V1 `options` mechanism — migration source for `apiKey: "not-needed"`.
- https://opencode.ai/docs/models (V1)
  - V1 model `options` — confirms the dropped V1 model fields (`strictTools`, `experimental_loose_tool_matching`, `extraArgs`) were undocumented no-ops.

## Wrong or ambiguous references

### `compaction.buffer` — "context window" vs "input window"

**Pages:**

- https://opencode.ai/v2/docs/compaction — describes `buffer` as a share of the model's **context window** kept free, with an example: a 128k window with a 20% buffer compacts at ~102k. That example applies the buffer to the full context window.
- https://opencode.ai/v2/docs/config — says "Automatic compaction uses the selected model's **usable input budget**."

**The problem:** The two pages disagree with each other. The compaction guide's example applies the buffer to the full context window, but the source code applies it to `min(limit.input, limit.context)` (the "input window"). When `limit.input` is below `limit.context`, the two interpretations give very different trigger points — and the source follows the config page, not the compaction guide. The compaction guide's wording is the misleading one.

**How the buffer is actually calculated (verified against source):**

Source of truth: `packages/core/src/session/compaction.ts` in [`anomalyco/opencode`](https://github.com/anomalyco/opencode) (the repo moved from `sst/opencode`), functions `calculateCeiling` (~line 1160) and `inputWindow` (~line 1182):

```
inputWindow  = min(limit.input, limit.context)        // when both are set
bufferTokens = round(inputWindow × buffer%)           // for a percentage buffer
ceiling      = inputWindow − bufferTokens             // the auto-compaction trigger
```

Notes:

- A percentage buffer is a string like `"40%"`; an absolute token count is also accepted by the schema (`buffer: Schema.Union([Percent, NonNegativeInt])`).
- The TUI's utilization percentage is measured against the **context window** (`limit.context`), not the input window. That is why the observed trigger percentage can look much lower than the buffer percentage suggests.
- The schema's own description says "Room kept free below the model's context window … as a share of the window" — "the window" here is the input window, which is the source of the ambiguity.

**Worked example (this config, after the v2-5 fix):**

- `limit.input = 205555`, `limit.context = 205555` → `inputWindow = 205555`
- `buffer = "40%"` → `bufferTokens = round(205555 × 0.40) = 82222`
- `ceiling = 205555 − 82222 = 123333` tokens → auto-compaction triggers at ~123,333 tokens
- As a share of the context window: `123333 / 205555 = 60.0%` — matches the "60% utilization" intent in the config comment.

**Why the earlier config triggered at ~35%:**

With `limit.input = 120000` (below the 205555 context window), `inputWindow = 120000`, so `ceiling = 120000 − round(120000 × 0.40) = 120000 − 48000 = 72000` tokens. Measured against the 205555 context window, that is `72000 / 205555 = 35.0%` — exactly the ~35–36% observed in the TUI. The v2-5 fix raises `limit.input` to 205555 so the input window equals the context window and the 40% buffer behaves as the comment intends.

**Resolution:** The config comment in `opencode-v2-5.jsonc` now states the formula explicitly, and `limit.input` is set equal to `limit.context` so the percentage applies to the full context window. If the compaction guide is later updated to say the buffer applies to `min(input, context)` (or to the "input window"), move https://opencode.ai/v2/docs/compaction back to *Verified references* above and remove this note.
