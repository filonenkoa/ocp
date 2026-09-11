# ocp

CLI to manage local LLM providers for [OpenCode](https://opencode.ai) and Hermes Agent. Auto-detects the server type and keeps your config in sync with reality: which models exist, their context window, input modalities — plus keeps Hermes' offline model catalogs current so its picker works when a server is down.

Zero dependencies · Node ≥ 18 (built-in `fetch`) · Linux/macOS (all paths via `$HOME`).

## Install

```sh
git clone https://github.com/filonenkoa/ocp.git && cd ocp
npm link    # or: ln -s "$PWD/bin/ocp.js" ~/.local/bin/ocp
```

No `npm install` — there are no dependencies.

## Quick start

```sh
ocp sync            # status check, writes nothing (safe default)
ocp add 3090 http://10.0.0.5:64980/v1 --key <api-key>    # register a provider
ocp sync --apply    # apply drift: new/removed models, ctx, modalities
```

## Commands

| Command | What it does |
| --- | --- |
| `ocp list` | Providers, model counts, key status |
| `ocp add <id> <baseURL>` | Register/update a provider (upsert). Options: `--key`, `--username/--password` (Unsloth auto-login + API-key creation), `--ctx N`, `--output N`, `--dry-run` |
| `ocp sync [--apply] [--target all\|opencode\|hermes] [--provider ID]` | Check/apply drift between servers and config |
| `ocp set-ctx <provider>` | Interactively re-ask context for models whose server reports none (needs a TTY) |
| `ocp help` | Full in-tool documentation |

## How it works

**Server detection:** llama.cpp (`status.args`/`architecture` in `/v1/models`) → LM Studio (`/api/v0/models`, embeddings skipped) → Unsloth UI (root `/openapi.json`) → any OpenAI-compatible server.

**Context window, first match wins:**
1. server value — llama.cpp preset `--ctx-size`, LM Studio loaded/max context, Unsloth loaded context;
2. `--ctx N` flag (`0` = no limit);
3. interactive prompt in a TTY (a guess is shown, Enter accepts it);
4. omitted with a warning when non-interactive and no `--ctx`.

Sync never clobbers already-configured values — re-ask them with `ocp set-ctx <provider>`. Guesses are only ever used as hints for OpenCode; **Hermes receives server-reported context lengths only**.

**Modalities:** llama.cpp `architecture.input_modalities` is ground truth (corrects mis-declarations); otherwise heuristics on the model id (`qwen3.8*` → text+image+video, qwen/gemma → text+image).

## Files touched

| File | What ocp does |
| --- | --- |
| OpenCode config — first existing of `opencode.jsonc` / `opencode.json` / `config.json` in `$OPENCODE_CONFIG_DIR` or `~/.config/opencode` (override: `OCP_CONFIG`) | provider definitions (models, limits, modalities) |
| `~/.local/share/opencode/auth.json` | API keys per provider id |
| `~/.hermes/config.yaml` | only the `models:` maps under `custom_providers` — surgical line edits, comments preserved; skipped if Hermes is not installed |

Every write is atomic (tmp + rename) and preceded by a `.bak-ocp-*` backup (last 3 kept). Offline or auth-failing servers are reported and skipped without blocking the rest. `ocp sync --target hermes` works standalone — the opencode config is only used to borrow API keys for Hermes providers pointing at the same server.
