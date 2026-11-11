# DeepKeep Open WebUI Filter

> **Development moved:** This integration is maintained in the
> [DeepKeep integrations monorepo](https://github.com/Deepkeepai/deepkeep-integrations/tree/main/integrations/openwebui).
> Please open new [issues](https://github.com/Deepkeepai/deepkeep-integrations/issues) and pull requests in the monorepo.
> Existing raw-file installation URLs and the [v0.1.1 release](https://github.com/Deepkeepai/openwebui-deepkeep/releases/tag/v0.1.1) remain available here.
> Historical source and release links are retained; no filter re-import is needed for this repository move.

Native Open WebUI Filter Function for DeepKeep AI Firewall using the DeepKeep v3 OpenAI-compatible moderation API.

## File

- `deepkeep_openwebui_filter.py`

## Public Release Files

- `LICENSE`: MIT license.
- `SECURITY.md`: vulnerability reporting and operational security notes.
- `COMMUNITY_LISTING.md`: historical draft text; the [published community listing](https://openwebui.com/posts/11544def-a732-4f5a-bb04-8df2e9fedd7e) now points to the monorepo.
- `.github/workflows/test.yml`: GitHub Actions test workflow for public repository CI.

## What It Does

- Runs pre-model moderation through `/api/v3/openai/moderations/pre`.
- Runs post-model moderation through `/api/v3/openai/moderations/post`.
- Blocks, redacts, modifies, or allows content based on DeepKeep `guardrail_action`.
- Supports fail-open or fail-closed behavior.
- Runs as an always-on filter when enabled in Open WebUI.

## Install

1. Open Open WebUI as an administrator.
2. Go to `Admin Panel > Functions`.
3. Create a new function.
4. Set the function type to `Filter`.
5. Import or paste the contents of `deepkeep_openwebui_filter.py`.
6. Save and enable the function.

## Configure Valves

Set these admin valves after importing the function:

| Valve | Description |
| --- | --- |
| `DEEPKEEP_API_KEY` | DeepKeep API key. |
| `DEEPKEEP_BASE_URL` | DeepKeep base URL, for example `https://api.example.deepkeep.ai`. |
| `PRE_MODEL` | DeepKeep model/firewall identifier for pre-moderation. Required when `MODE` is `pre` or `both`. |
| `POST_MODEL` | DeepKeep model/firewall identifier for post-moderation. Required when `MODE` is `post` or `both`. |
| `MODE` | `pre`, `post`, or `both`. Defaults to `both`. |
| `FAIL_MODE` | `fail_closed` or `fail_open`. Defaults to `fail_closed`. |
| `BLOCK_MESSAGE` | Message shown when content is blocked. |
| `TIMEOUT` | DeepKeep HTTP timeout in seconds. |
| `priority` | Open WebUI filter execution order. Lower values run first. |

## Enable

For firewall behavior, make the filter active and global:

1. Go to `Admin Panel > Functions`.
2. Enable the function.
3. Toggle global mode for the function.

The filter intentionally does not set `self.toggle = True`, so users cannot disable it from the chat UI when it is active and in scope.

## Smoke Tests

Run these checks after configuring valves:

1. Send an allowed prompt and confirm the model responds normally.
2. Send a prompt that DeepKeep should block and confirm the model is not called.
3. Send a prompt that DeepKeep should redact or modify and confirm the model receives the changed content.
4. Trigger a blocked assistant output policy and confirm the response shown to the user is replaced with `BLOCK_MESSAGE`.
5. Temporarily set an invalid `DEEPKEEP_BASE_URL` and confirm `fail_closed` blocks while `fail_open` allows traffic through.

## Live DeepKeep API Smoke Test

Use this before importing into Open WebUI to verify credentials, base URL, model identifiers, and the DeepKeep v3 contract.

Set environment variables:

```powershell
$env:DEEPKEEP_API_KEY = "your-api-key"
$env:DEEPKEEP_BASE_URL = "https://api.example.deepkeep.ai"
$env:DEEPKEEP_PRE_MODEL = "your-pre-model-or-firewall-id"
$env:DEEPKEEP_POST_MODEL = "your-post-model-or-firewall-id"
```

`.env.example` contains the same variable names as a reference. Keep real credentials out of source control.

Run from this directory:

```powershell
python scripts\smoke_deepkeep_v3.py --phase both --content "Hello from smoke test"
```

The script prints the parsed DeepKeep decision for each phase. It exits before making requests if required environment variables are missing.

## Local Tests

The repository includes mock-based tests for the standalone filter. They do not call DeepKeep or Open WebUI.

Run from this directory:

```powershell
python -m pip install -r requirements-dev.txt
python -m pytest
```

Current coverage checks:

- DeepKeep decision parsing and action priority.
- Pre-moderation redaction and blocking.
- Post-moderation redaction and blocking.
- `fail_open` and `fail_closed` behavior on DeepKeep errors.
- `pre`, `post`, and `both` mode gating.
- DeepKeep v3 endpoint, header, payload, and API error handling.

## Troubleshooting

- If every request is blocked, check `FAIL_MODE`, `DEEPKEEP_BASE_URL`, `DEEPKEEP_API_KEY`, and the required model valve for the active `MODE`.
- If pre-moderation works but post-moderation does not run, confirm the filter is active for the selected model and the Open WebUI request path invokes `outlet()` for your version/API usage.
- For direct `/api/chat/completions` callers on some Open WebUI releases, `outlet()` may not rewrite the immediate HTTP response. WebUI chat requests and `/api/chat/completed` are the reliable paths for post-response filtering.
- If Open WebUI fails to import the function due to `httpx`, install the function requirement or ensure `httpx` is available in the Open WebUI Python environment.
- If users can disable the filter from chat, verify the plugin file was not modified to set `self.toggle = True`.

## API Contract

The filter calls DeepKeep with:

```http
POST {DEEPKEEP_BASE_URL}/api/v3/openai/moderations/pre
X-API-Key: {DEEPKEEP_API_KEY}
Accept: application/json
```

Pre-moderation body:

```json
{
  "model": "PRE_MODEL",
  "input": "user message",
  "title": "optional title",
  "chat": "optional chat id"
}
```

Post-moderation body:

```json
{
  "model": "POST_MODEL",
  "output": "assistant message",
  "title": "optional title",
  "chat": "optional chat id"
}
```

Decision parsing expects DeepKeep actions at `verbosity[*].details.guardrail_action` and replacement content at `verbosity[*].details.modified[*].content`.
