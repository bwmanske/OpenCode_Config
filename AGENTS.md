# AGENTS.md

This is an OpenCode **config workspace**, not a code repo. There is no build, test, lint, CI, or git history — the versioned filenames are the only history.

## Layout

- `JSON files\opencode-v1.jsonc` — original V1 config (migration source).
- `JSON files\opencode-v2-1.jsonc` … `opencode-v2-7.jsonc` — successive V2 conversion iterations.
- **Live config (what actually runs):** `C:\Users\brad\.config\opencode\opencode.jsonc` (global). As of 2026-11-17 it is byte-identical to `opencode-v2-7.jsonc`. Verify which version is deployed before assuming:

  ```powershell
  Get-FileHash "$env:USERPROFILE\.config\opencode\opencode.jsonc", "JSON files\opencode-v2-7.jsonc" -Algorithm SHA256
  ```

## Version lineage (verified)

- v2-1, v2-2: provider `package` missing the `aisdk:` prefix.
- v2-3: syntax error — missing comma after `"update": "auto"`.
- v2-4: first fully valid version.
- v2-5: fixed version — resolves all Known issues below; deployed 2026-11-06.
- v2-6: restores `websearch: true` (the v2-5 removal was a regression).
- v2-7: current — adds OpenRouter providers (`openrouter-free`, `openrouter-paid`) with the API key via `{env:OPENROUTER_API_KEY}`; deployed 2026-11-17.

## Working with these files

- Files are **JSONC** (comments allowed). Plain `ConvertFrom-Json` fails on them; strip comments or use a JSONC-aware parser to validate.
- Target schema is **V2** (`"$schema": "https://opencode.ai/config.json"`). Check the V2 docs (`opencode.ai/docs/` — config, providers, models, permissions, policies; the old `/v2/docs/` URLs now redirect there, and the V1 docs are gone — see [docs/WEB_REFERENCES.md](docs/WEB_REFERENCES.md)).
- Environment: local LM Studio at `http://localhost:1234/v1`; models `qwen/qwen3.6-27b`, `qwen/qwen3.8-27b`, `jackrong/qwopus3.8-27b-coder-fast`. Plus OpenRouter via the `openrouter-free` / `openrouter-paid` providers (v2-7+), which require the `OPENROUTER_API_KEY` environment variable. **Never write the real OpenRouter key into any file in this workspace** — the config references it only as `"{env:OPENROUTER_API_KEY}"`, and this repo is public.

## Known issues (in v2-4; fixed in v2-5, except websearch — see below)

These were present in v2-4 (and in the live config while it was v2-4). v2-5 fixes all of them — except the `websearch` item, whose "fix" was a regression:

- `providers.lmstudio.body.includeUsage: true` is misplaced — `body` is sent verbatim in the request body. Fixed in v2-5 as `compatibility.streamUsage: true` (the documented control; default is already `true`).
- `headers.Authorization: "Bearer not-needed"` was an undocumented workaround for V1 `options.apiKey`. Fixed in v2-5 as `settings.apiKey: "not-needed"`.
- `websearch: true` is not a documented form (docs show `false` and `{provider}`), and all web-search providers are cloud services. v2-5 removed it — **a regression**: web search stopped working, and the line was restored in v2-6 (kept in the live config because it is observed to be required; see [docs/WEB_REFERENCES.md](docs/WEB_REFERENCES.md) for the source-verified behavior).
- `policies` denied only `openai`, `anthropic`, `google`, so the "air-gapped" comment overstated it. Fixed in v2-5: deny `*`, then allow `lmstudio` (last match wins).

## External Web References

Docs pages this config and its verification depend on (last checked 2026-11-10) are catalogued in [docs/WEB_REFERENCES.md](docs/WEB_REFERENCES.md). That file lists each page and the specific claims the config relies on, and it also records pages that turned out to be **wrong or ambiguous** (e.g. the `compaction.buffer` "context window" wording) together with how the affected behavior is actually calculated, verified against source. Re-check the pages when OpenCode updates them; if a flagged page is later corrected, move its link back to the verified list and drop the detailed note.
