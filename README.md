# OpenCode JSON Config

OpenCode configuration files for a fully local, air-gapped setup: [OpenCode](https://opencode.ai) V2 talking to [LM Studio](https://lmstudio.ai) on `localhost:1234`, with no cloud providers reachable.

## Contents

| Path | What it is |
| --- | --- |
| `JSON files/opencode-v1.jsonc` | Original V1 config (migration source) |
| `JSON files/opencode-v2-1.jsonc` … `opencode-v2-5.jsonc` | Successive V2 conversion iterations |
| `opencode.jsonc` | Minimal **local** (project-level) config example — sets only the default model |
| `AGENTS.md` | Context for AI coding agents working in this repo |
| `docs/VERSION_HISTORY.md` | Version history (placeholder) |

**`JSON files/opencode-v2-5.jsonc` is the current, verified config.** Earlier iterations are kept for history: v2-1/v2-2 are missing the `aisdk:` package prefix, v2-3 has a syntax error, v2-4 is the first fully valid version, and v2-5 fixes four issues in it (see `AGENTS.md` → Known issues).

## Requirements

- OpenCode V2
- LM Studio serving an OpenAI-compatible API at `http://localhost:1234/v1`
- Models loaded in LM Studio: `qwen/qwen3.6-27b`, `qwen/qwen3.8-27b`, `jackrong/qwopus3.8-27b-coder-fast`

## Usage

**Global config** — copy the latest version over your global config, then restart OpenCode:

- Linux/macOS: `~/.config/opencode/opencode.jsonc`
- Windows: `%USERPROFILE%\.config\opencode\opencode.jsonc`

**Local (project) config** — the repo-root `opencode.jsonc` shows the minimal project-level form: it sets only the default `model` and inherits everything else (provider definition, permissions, policies) from the global config.

## Notes

- Files are **JSONC** (comments allowed); validate with a JSONC-aware parser, not a plain JSON parser.
- The config is air-gapped by policy: `policies` denies every provider except `lmstudio` (last match wins).
- `apiKey: "not-needed"` is intentional — LM Studio runs unauthenticated locally, but the AI SDK runtime expects a key at construction.
- To switch the default model, uncomment one of the `model` lines in the config (keep only one active).
