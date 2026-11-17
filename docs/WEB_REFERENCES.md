# Web References

Docs pages this config and its verification depend on (last checked 2026-11-10). Re-check when OpenCode updates them; each entry lists the claims that would change if the page changes.

> **Note (2026-11-10): the docs were consolidated.** Every `/v2/docs/X` URL now redirects to `/docs/X` with byte-identical content — there is a single set of docs at `/docs/` (the V2 docs), and the `/v2/` prefix is just a redirect. All references below still resolve. The two entries labelled **(V1)** are now **historical**: the old V1 pages were replaced by the consolidated V2 content, so the V1-specific claims can no longer be verified from those pages (see the notes on each).

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
- https://opencode.ai/v2/docs/migrate-v1
  - V1→V2 field conversions used for this config (V1 `options.apiKey` → `settings.apiKey`; V1 `npm` package key → `package` with `aisdk:` prefix).
- https://opencode.ai/v2/docs/config
  - V2 core config reference (locations, precedence, `update`, `snapshots`, `compaction`).
  - "Automatic compaction uses the selected model's usable input budget." — this is the statement that matches the source (see *Wrong or ambiguous references* below for the conflict with the compaction guide).
- https://opencode.ai/docs/config (V1) — **HISTORICAL (2026-11-10)**
  - Was: V1 `options` mechanism — migration source for `apiKey: "not-needed"`.
  - No longer verifiable: after the consolidation this URL serves the V2 config page. The V1 `options` block is no longer documented there (`options` now appears only in generic prose; `apiKey` only in a V2 `settings` example). Kept for provenance of the `apiKey: "not-needed"` migration.
- https://opencode.ai/docs/models (V1) — **HISTORICAL (2026-11-10)**
  - Was: V1 model `options` — confirms the dropped V1 model fields (`strictTools`, `experimental_loose_tool_matching`, `extraArgs`) were undocumented no-ops.
  - No longer verifiable: after the consolidation this URL serves the V2 models page, and all four V1 fields (`strictTools`, `experimental_loose_tool_matching`, `extraArgs`, `response_format`) are absent from it. Kept for provenance of the dropped-fields decision.

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

### `websearch` — `true` is undocumented but appears required in practice

**Pages:**

- https://opencode.ai/docs/websearch — documents only two forms: `websearch: false` and `websearch: { provider: ... }` (provider is `"random"` or a specific provider ID). `true` is not documented.

**The problem:** The docs — and the v2-5 "fix" that removed `websearch: true` on the basis that it is undocumented — treat `true` as an invalid form. But in practice, removing the line stopped web search, and re-adding `"websearch": true` to the live config restored it. So `true` appears to be required even though it is undocumented: a discrepancy between the docs and observed behavior.

**How it actually works (verified against source, `dev` branch):**

Source of truth (in [`anomalyco/opencode`](https://github.com/anomalyco/opencode)):

- **Schema** — `packages/schema/src/config/websearch.ts`:
  ```ts
  export const Selection = Schema.Union([Schema.Literal(false), Info])
  // Info = { provider: Union([Literal("random"), WebSearch.ID]) }
  ```
  The only schema-valid values are `false` and `{ provider }`. **`true` is not schema-valid.**

- **Config plugin** — `packages/core/src/config/plugin/websearch.ts`:
  ```ts
  const selection = Config.latest(loaded.entries, "websearch")
  if (selection === false) websearch.default.set(false)
  if (selection) websearch.default.set(selection.provider)
  ```
  - `websearch: false` → selection = `false` (web search disabled).
  - `websearch: { provider: X }` → selection = `X`.
  - `websearch: true` → `if (selection)` is true, but `true.provider` is `undefined`, so selection = `undefined`. **In the current `dev` code, `true` is a no-op — equivalent to omitting the key.**
  - No `websearch` key → selection stays `undefined`.

- **Service** — `packages/core/src/websearch.ts`: `selection()` returns the config selection when it is `false` / `"random"` / a valid provider ID; otherwise it falls back to a KV-stored value (`websearch:provider`); otherwise `undefined`. `query()` fails with `ProviderRequiredError` when no provider resolves.

- **Tool** — `packages/core/src/tool/plugin/websearch.ts`: the `websearch` tool is always registered, but a session hook removes it from the model's tool list when the selection is `false` (a `DisabledError`) or when no providers are available. When the tool is invoked with no resolvable provider, it shows a form to pick a provider, and the choice is stored in KV via `websearch.select()`.

- **Providers** — `packages/core/src/plugin/websearch/index.ts` registers five providers (Exa, Firecrawl, Parallel, Tavily, TinyFish). All are cloud services requiring API keys — which is why the original v2-5 note called them "incompatible with the air-gap intent."

**So web search is actually enabled by a resolvable provider** — either `websearch: { provider: X }` in config, or a KV-stored selection (from the provider-pick form or a prior `select()`). It is disabled by `websearch: false` or by having no providers. In the current `dev` code, `websearch: true` does not itself select a provider.

**The discrepancy:** The current `dev` code treats `websearch: true` as a no-op (same as omitting it), which contradicts the observed behavior that removing the line broke web search. Likely explanations:

1. **Version difference** — the installed OpenCode build may handle `true` differently (e.g. as "enable with a random/default provider"); the `dev` branch may have changed since that build.
2. **The real enabler is the KV/form selection** — web search works because a provider was selected (stored in KV), and the `websearch: true` line is correlated rather than causal.

**Resolution / recommendation:** The documented, schema-valid way to enable web search is `websearch: { "provider": "..." }` (or `"random"`). `websearch: true` is kept in the live config because it is observed to be required, but it is not a documented form and is a no-op in the current `dev` source. If the docs add `true` as a valid form, or the source changes so `true` selects a default provider, move this reference back to *Verified references* and update the note.
