# Cloudflare Cache Purge Action — Agent Instructions

## Project Overview

This repository is a **composite GitHub Action** that purges Cloudflare CDN cache for a zone via the Cloudflare REST API (`POST /client/v4/zones/{zone_id}/purge_cache`). It supports **purge everything** or **purge by URL list** (JSON array of absolute URLs). The implementation is a single bash step using `curl` and `jq`—no Node.js, Docker images, or compiled artifacts.

Consumers reference the action in their workflows with `uses: <owner>/cloudflare-cache-purge-action@<ref>`. End-user documentation lives in [README.md](README.md); this file is the **central reference for AI agents** changing or reviewing this repository.

## Repository Architecture

### Structure Overview

```
cloudflare-cache-purge-action/
├── action.yml      # Composite action: inputs + bash run block
├── AGENTS.md       # Agent instructions (this file)
├── LICENSE         # Apache License 2.0
└── README.md       # User-facing documentation and examples
```

### Action Components

The action is defined entirely in **`action.yml`**:

| Piece | Role |
| ----- | ---- |
| **`inputs`** | `api_token`, `zone_id`, `purge_everything`, `files` — must stay aligned with README and with env mapping in the step |
| **`runs.using: composite`** | Single `steps` entry with `shell: bash` and inline `run` script |
| **Environment** | `CF_API_TOKEN`, `CF_ZONE_ID`, `PURGE_EVERYTHING`, `FILES` — derived from inputs via `${{ inputs.* }}` |

### Execution Flow

1. If `PURGE_EVERYTHING` is exactly `true`, send `{"purge_everything":true}`.
2. Otherwise validate `FILES` as JSON with `jq`; on failure emit `::error::` for invalid `files` and exit `1`.
3. Build payload `{"files":<array>}` with `jq -cn --argjson files`.
4. `curl` POST to `https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/purge_cache` with `Authorization: Bearer ${CF_API_TOKEN}`.
5. If JSON response has `.success == true`, append success text to `GITHUB_STEP_SUMMARY` (and stdout via `tee`).
6. Else extract `.errors[].message`, emit `::error::Cloudflare cache purge failed — …`, append failure to summary, exit `1`.

### Runtime Dependencies

| Tool | Notes |
| ---- | ----- |
| **bash** | Composite step `shell: bash` |
| **curl** | HTTP client; `--silent --show-error` |
| **jq** | JSON validation, payload build, response parsing |

These are present on **GitHub-hosted** `ubuntu-latest` (and typical) runners; do not assume extra packages unless documented.

## Naming Conventions

### YAML and GitHub Actions

- **Input keys:** `snake_case` (`api_token`, `zone_id`, `purge_everything`, `files`) — matches Cloudflare API token naming style and keeps composite `inputs` idiomatic.
- **Step `env` keys:** `SCREAMING_SNAKE_CASE` (`CF_API_TOKEN`, `CF_ZONE_ID`, `PURGE_EVERYTHING`, `FILES`) — avoids clashes and matches common shell env conventions.
- **Action metadata `name`:** Human-readable title string in `action.yml` (e.g. `"Cloudflare Cache Purge"`).

### Shell Script (inside `run: \|`)

- **Shell variables:** Prefer `SCREAMING_SNAKE_CASE` for env-backed values already set from inputs (`CF_*`, `PURGE_EVERYTHING`, `FILES`).
- **Locals in script:** `PAYLOAD`, `RESPONSE`, `ERRORS` — `UPPERCASE` for short-lived script globals is consistent with existing script; if you introduce many locals, `lower_snake_case` is acceptable but **stay consistent within one change**.

### Workflow Examples (README)

- **Secrets:** Document as `CF_API_TOKEN` and `CF_ZONE_ID` unless the project explicitly renames them.
- **`uses:` line:** Pin to a version tag (e.g. `@v1`) in README examples; replace owner if the canonical repo differs.

## Technology Stack

| Layer | Choice |
| ----- | ------ |
| **Action type** | Composite (`runs.using: composite`) |
| **Languages** | YAML (`action.yml`), Bash (inline) |
| **API** | Cloudflare v4 Zones Purge Cache |
| **Auth** | Bearer token (`api_token` input) |

There is no `package.json`, build step, or test runner in-repo unless you add them later—agents should not assume Node or pnpm here.

## Maintainer / Contributor Workflows

### Changing Behavior

1. Edit **`action.yml`** only for action logic unless adding new tracked files (tests, docs).
2. Keep **`inputs`** descriptions accurate; mirror changes in **README.md** (inputs table, examples, error handling).
3. **purge_everything** is a string compared to `"true"` in bash—document that workflows should pass `"true"` / `"false"` strings, not booleans, for predictable behavior.
4. **`files`** must remain a string containing JSON (e.g. YAML single-quoted value) because GitHub passes inputs as strings.

### Validating Locally

- Manually review quoting: `jq --argjson` requires valid JSON in `FILES`.
- Test in a **private fork** or disposable repo with real `CF_API_TOKEN` and `CF_ZONE_ID` secrets (never commit tokens).
- Optional: reproduce the `run` script body in a local shell with exported env vars (no `GITHUB_STEP_SUMMARY` required for curl/jq checks; use a temp file or omit summary lines).

### Releasing

- Tag semantic or moving **`v1`** branch/tag per [GitHub Actions versioning](https://docs.github.com/en/actions/creating-actions/about-custom-actions#using-tags-for-release-management) so consumers can pin `@v1`.

## Inputs Reference (authoritative shape)

Keep this table synchronized with `action.yml` and README:

| Input | Required | Default | Notes |
| ----- | -------- | ------- | ----- |
| `api_token` | Yes | — | Zone Cache Purge permission |
| `zone_id` | Yes | — | Target zone ID |
| `purge_everything` | No | `"false"` | Literal string `"true"` triggers full purge |
| `files` | No | `"[]"` | JSON array string; ignored if purging everything |

## Common Patterns

### GitHub Workflow Commands

- **`::error::`** — User-visible errors in the Actions UI; use for invalid input and API failures.
- **`GITHUB_STEP_SUMMARY`** — Append markdown/text for job summary; success uses `tee`, failure uses append.

### jq Usage

- Validate arbitrary JSON: `echo "${FILES}" | jq -e .`
- Build body: `jq -cn --argjson files "${FILES}" '{"files":$files}'`
- Check success: `echo "${RESPONSE}" | jq -e '.success == true'`
- Collect errors: `[.errors[].message]` with fallback to `"unknown error"` when empty (see current script).

### curl Usage

- Always use the documented endpoint and headers (`Authorization: Bearer`, `Content-Type: application/json`).
- `--silent --show-error` reduces noise while surfacing HTTP errors.

## Security and Secrets

- **Never** echo `CF_API_TOKEN` or log full `Authorization` headers.
- Document minimum token scope: Zone — **Cache Purge** on the specific zone.
- Encourage repository **secrets** for `api_token` and storing `zone_id` as a secret if it should not be public (optional hardening).

## Error Handling Contract

Agents must preserve this behavior unless intentionally breaking for a major version:

| Condition | Behavior |
| --------- | -------- |
| `purge_everything` not `true` and `files` invalid JSON | `::error::` with guidance; exit `1` before API call |
| API returns non-success | Join `.errors[].message`; `::error::Cloudflare cache purge failed — …`; summary failure line; exit `1` |
| API success | Success message to summary and stdout |

## Best Practices

### Architecture

- Keep the action **single-purpose**: cache purge only; do not bundle unrelated Cloudflare operations without a design discussion.
- Prefer **composite + bash** for zero extra runtime if requirements stay simple.

### Changes

- Update **README.md** and **AGENTS.md** when inputs or behavior change.
- Keep examples copy-pasteable (correct YAML quoting for `files`).
- Prefer **backward-compatible** input defaults; use a new major tag if removing or renaming inputs.

### Code Quality (bash in YAML)

- Quote variables where word-splitting or empty expansion matters (`"${VAR}"`).
- Avoid complex bash; if the script grows, consider extracting to a `scripts/` file and `run: bash ${{ github.action_path }}/script.sh` (composite supports `github.action_path`) in a future iteration—document if added.

## Contribution Guidelines

- Follow naming and security conventions above.
- Align `action.yml`, **README.md**, and **AGENTS.md** in the same PR when behavior or inputs change.
- Do not commit Cloudflare tokens, zone IDs, or real URLs that imply private infrastructure unless they are clearly public examples (`example.com`).
- Respect **Apache 2.0** licensing for new files (see [LICENSE](LICENSE)).

---

This documentation is the central reference for AI agents working on **cloudflare-cache-purge-action**. For workflow authors consuming the action, see [README.md](README.md).
