# ocp

Manage local model providers for [OpenCode](https://opencode.ai) (and optionally
Hermes Agent) from the CLI.

`ocp` talks to your local inference servers, auto-detects what they are, and keeps
your provider config in sync with reality: which models exist, their context window,
and their input modalities. It also keeps Hermes' offline fallback model catalogs
current so its picker still works when a server is down.

Zero dependencies. Requires **Node.js >= 18** (uses the built-in `fetch`).

## Supported servers

| Server | Detection | What ocp reads |
| --- | --- | --- |
| llama.cpp server | `/v1/models` entries with `status.args` / `architecture` | preset `--ctx-size`, trained window (`meta.n_ctx_train`), `input_modalities` (ground truth) |
| LM Studio | `/api/v0/models` | loaded/max context, VLM type; embedding models are skipped |
| Unsloth UI | root `/openapi.json` title | login + persistent API key creation, loaded context |
| any OpenAI-compatible server | fallback for `/v1/models` | `context_length` when present |

## Install (Ubuntu / Linux)

```sh
# 1. Node.js >= 18 if you don't have it (NodeSource, nvm, or your distro's package)
node --version   # must print v18+

# 2. Clone and link the command
git clone <your-repo-url> ocp && cd ocp
npm link          # puts `ocp` on your PATH
```

No `npm install` needed — there are no dependencies. Alternatives to `npm link`:

```sh
node bin/ocp.js help                 # run directly, or
ln -s "$PWD/bin/ocp.js" ~/.local/bin/ocp   # manual symlink (~/.local/bin on PATH)
```

## Quick start

```sh
# status check — nothing is written (safe default)
ocp sync

# register a provider (server type auto-detected, key saved to auth.json)
ocp add 3090 http://10.0.0.5:64980/v1 --key <api-key>

# Unsloth UI: ocp logs in and creates a persistent API key for you
ocp add helen http://10.0.0.6:8888/v1 --username unsloth --password ***

# apply pending changes (new/removed models, ctx, modalities)
ocp sync --apply

# only one provider / only the Hermes catalogs
ocp sync --apply --provider 3090
ocp sync --target hermes
```

## Commands

- `ocp list` — configured providers, model counts, key status.
- `ocp add <id> <baseURL>` — register or update a provider (upsert semantics).
- `ocp sync [--apply] [--target all|opencode|hermes] [--provider ID]` — check and
  optionally apply drift between servers and config.
- `ocp set-ctx <provider>` — interactively re-ask context windows for models whose
  server reports none (needs a terminal).
- `ocp help` — full in-tool documentation, including the resolution rules below.

### Context window resolution (first match wins)

1. **server-reported** — llama.cpp preset `--ctx-size`, LM Studio loaded/max
   context, Unsloth loaded context;
2. `--ctx N` flag (`0` = omit the limit entirely);
3. interactive prompt in a terminal (a guess is shown; Enter accepts it);
4. omitted with a warning when non-interactive and no `--ctx`.

Already-configured values are never clobbered by sync — re-ask them with
`ocp set-ctx <provider>`. Guesses are only ever used as prompt hints for the
OpenCode config; **Hermes only receives server-reported context lengths**.

### Modalities

llama.cpp `architecture.input_modalities` is ground truth and will correct
over/under-declared models. Otherwise heuristics on the model id apply
(`qwen3.8*` → text+image+video, other qwen/gemma → text+image). LM Studio
embedding models are skipped.

## Files touched

| File | What ocp does |
| --- | --- |
| `~/.config/opencode/opencode.jsonc` | provider definitions (models, limits, modalities) |
| `~/.local/share/opencode/auth.json` | API keys per provider id |
| `~/.hermes/config.yaml` | only the per-provider `models:` maps under `custom_providers` (+ `models_discovered`) — surgical line edits, comments and everything else preserved; skipped entirely if Hermes is not installed |

Every write is atomic (tmp file + rename) and preceded by a `.bak-ocp-*` backup
(the last 3 are kept). Run `ocp sync` without `--apply` any time to see what would
change before committing it.

## Notes for other machines

- All paths are derived from `$HOME`, so the tool works as-is on Ubuntu/macOS.
- If a machine has no OpenCode config yet, start with `ocp add <id> <baseURL>`;
  `ocp sync --target hermes` also works standalone (the opencode config is only
  used to borrow API keys for Hermes providers pointing at the same server).
- Offline or auth-failing servers are reported and skipped — they never block a
  sync of the rest.
