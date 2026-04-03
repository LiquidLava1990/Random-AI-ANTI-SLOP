# Claude Code OAuth — CLI Setup, Bearer Headers & Token Renewal

This document covers how to use Claude Code's OAuth tokens with a custom agent system — including the full auth flow, header requirements, token renewal procedure, and fallback provider setup.

---

## How It Works

Claude Code (`claude` CLI) authenticates via Anthropic's OAuth flow and stores tokens in `~/.claude/.credentials.json`. Any system that needs to call the Anthropic API using your Claude account uses these tokens — not a static API key.

### Why OAuth instead of a static API key?

Starting January 2026, Anthropic fingerprints OAuth clients server-side. Opus and Sonnet models reject direct HTTP calls from non-Claude-Code clients when using OAuth tokens. Haiku is allowed direct. All Opus/Sonnet calls must route through `claude -p` (the CLI bridge) to pass fingerprint checks.

### Call flow

```
User message
  -> Agent orchestrator
  -> ModelRouter.route()
     - Try Anthropic (primary)
       - OAuth token detected (sk-ant-oat*) -> CLI bridge for Opus/Sonnet
       - Haiku -> direct HTTP with Bearer header
     - If Anthropic fails -> try fallback provider (e.g. Moonshot/Kimi)
```

---

## Header Reference

### OAuth tokens (sk-ant-oat*) — Primary

Used for all production calls to `/v1/messages`:

```python
headers = {
    "Authorization": f"Bearer {access_token}",
    "anthropic-version": "2023-06-01",
    "anthropic-beta": "oauth-2025-04-20",   # required — do NOT change version
    "content-type": "application/json",
}
```

### Standard API keys (sk-ant-api*) — NOT for OAuth

```python
headers = {
    "x-api-key": api_key,
    "anthropic-version": "2023-06-01",
}
```

Do not mix these. OAuth tokens require `Authorization: Bearer` + the beta header. Standard keys use `x-api-key`. Using `x-api-key` with an OAuth token returns 401.

### Health check exception

The `/v1/models` endpoint accepts both `x-api-key` and `Bearer` for OAuth tokens, so health checks using `x-api-key` will pass even with an OAuth token. The 401 only appears on `/v1/messages`. This is why health checks can look OK while production calls fail.

---

## CLI Bridge (Opus/Sonnet)

Routes through `claude -p` subprocess to pass fingerprint checks:

```bash
claude -p --no-session-persistence --output-format json --model claude-opus-4-6
```

Tools are filtered to primitives only (`read`, `write`, `edit`, `exec`) before being passed to the CLI bridge. All other tools (agent spawning, memory ops, etc.) are intercepted by the orchestrator layer and never reach the bridge.

---

## Token Renewal Procedure

### Step 1 — Check CLI auth status

```bash
claude auth status
```

- `loggedIn: true` → skip to Step 3
- `loggedIn: false` → do Step 2

### Step 2 — Re-authenticate (only if stale)

```bash
claude login
```

Opens Anthropic OAuth in browser. After authorization, CLI writes fresh token to `~/.claude/.credentials.json` automatically.

### Step 3 — Read the fresh token

File: `~/.claude/.credentials.json`

```json
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-oat01-...",
    "refreshToken": "sk-ant-ort01-...",
    "expiresAt": 1773837716063
  }
}
```

### Step 4 — Propagate to all token locations

Your system likely reads the token from multiple places. Update all of them:

| Location | Key |
|----------|-----|
| `~/.claude/.credentials.json` | `claudeAiOauth.accessToken` (CLI writes this — primary) |
| Encrypted vault | `anthropic_api_key` |
| `.env` | `ANTHROPIC_API_KEY`, `TRIDENT_ANTHROPIC_KEY` |
| `os.environ` | Updated at runtime by token sync |
| Agent config / provider config | `agent_auth.access` or equivalent |

**Resolution order at boot:**
```
1. Provider config api_key (vault-resolved)
2. os.environ["ANTHROPIC_API_KEY"]
3. OVERRIDE: ~/.claude/.credentials.json accessToken (if exists and len > 50)
```

Step 3 always wins when CLI credentials exist. This means you typically only need to `claude login` and the system picks it up automatically on next boot.

### Step 5 — Verify the token

Check expiry:

```python
import json, time
from pathlib import Path

creds = json.loads((Path.home() / ".claude" / ".credentials.json").read_text())
oauth = creds["claudeAiOauth"]
remaining = (oauth["expiresAt"] / 1000 - time.time()) / 60
print(f"Token expires in {remaining:.0f} minutes")
print(f"Token prefix: {oauth['accessToken'][:20]}...")
```

Test Opus via CLI bridge:

```bash
echo "Reply with: OPUS_OK" | claude -p --no-session-persistence --output-format json --model claude-opus-4-6
```

---

## Auto-Refresh (TIDE Background Task)

A background task (`oauth_refresh.py`) runs every 15 minutes via your task scheduler (TIDE). It:

1. Reads the current token from `~/.claude/.credentials.json`
2. Calls the Anthropic token refresh endpoint
3. Writes the new token back to `.credentials.json`, vault, and `.env`

If auto-refresh is working, manual renewal is rarely needed. Check whether it's running if you see unexpected 401s.

---

## Fallback Provider (Moonshot/Kimi K2.5)

When Anthropic is unavailable, calls fall back to Moonshot's Kimi K2.5. This requires a separate Moonshot API key.

### Key locations

| Location | Variable |
|----------|----------|
| Encrypted vault | `moonshot_api_key` |
| `.env` | `MOONSHOT_API_KEY` |
| Provider config | `providers.moonshot.api_key` |

Resolution order: vault-resolved config value → `os.environ["MOONSHOT_API_KEY"]`

### Model mapping

Anthropic model names must be mapped to Moonshot equivalents:

| Anthropic | Moonshot |
|-----------|----------|
| `claude-opus-4-6` | `kimi-k2.5` |
| `claude-sonnet-4-6` | `kimi-k2.5` |
| `claude-haiku-4-5` | `kimi-k2.5` |

The router handles this mapping. If the mapping is broken, the raw Anthropic model name gets sent to Moonshot's API and returns 400/404.

### Moonshot headers

```python
headers = {
    "Authorization": f"Bearer {moonshot_api_key}",
    "content-type": "application/json",
}
```

Moonshot uses `Bearer` with its own key — same header format as Anthropic OAuth.

### Endpoint

```
International: https://api.moonshot.ai/v1   ← correct
China:         https://api.moonshot.cn/v1   ← wrong for international keys
```

### Models

```
kimi-k2.5              — 200K context (default)
kimi-k2-turbo-preview  — 1M context
```

Both require `temperature=1.0`.

### Quick test

```python
import httpx, os

key = os.environ.get("MOONSHOT_API_KEY", "")
r = httpx.post(
    "https://api.moonshot.ai/v1/chat/completions",
    headers={"Authorization": f"Bearer {key}"},
    json={"model": "kimi-k2.5", "messages": [{"role": "user", "content": "ping"}], "max_tokens": 5},
)
print(r.status_code, r.json())
```

Expected: `200 {... "content": "Pong" ...}`

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `401 Unauthorized` on `/v1/messages` | Using `x-api-key` instead of `Bearer` for OAuth token | Switch to `Authorization: Bearer <token>` + `anthropic-beta: oauth-2025-04-20` |
| `400` on Opus/Sonnet direct HTTP | Anthropic fingerprint rejection | Route through CLI bridge (`claude -p`) |
| Health check passes but messages 401 | `/v1/models` accepts both header types; `/v1/messages` doesn't | Fix the production request headers — health check is misleading |
| `Failed to decrypt tokens` | Encryption key regenerated, vault stale | Re-seed the encrypted vault from `~/.claude/.credentials.json` |
| `No tokens stored` | Encrypted store empty/corrupt | Re-seed encrypted vault |
| `No refresh token` | CLI credentials missing | Re-run `claude login` |
| `oauth-2025-04-01` returns 401 | Wrong beta header version | Must be `oauth-2025-04-20` not `04-01` |
| Fallback never triggers | Model name lookup fails (bare vs prefixed key mismatch) | Check that your router resolves `claude-opus-4-6` → `kimi-k2.5` correctly |
| Fallback returns 400/404 | Provider prefix not stripped before API call | Strip `moonshot/` prefix — send `kimi-k2.5` not `moonshot/kimi-k2.5` |
| Fallback has no tools | Moonshot in `_NO_TOOL_PROVIDERS` set | Only Ollama/local models should have tools stripped |
| Auto-refresh doesn't update vault | Token sync wrote to `.env` and config but skipped encrypted store | Ensure your sync function writes to ALL token destinations including the vault |

---

## Critical Files (Reference)

| File | Purpose |
|------|---------|
| `~/.claude/.credentials.json` | Fresh OAuth token — Claude CLI writes here |
| `runner/providers/anthropic.py` | OAuth detection, Bearer headers, CLI bridge routing |
| `runner/providers/cli_bridge.py` | `claude -p` subprocess runner for Opus/Sonnet |
| `runner/model_router.py` | Provider dispatch, model mapping, tier gates, fallback chain |
| `runner/providers/moonshot.py` | Kimi K2.5 provider — OpenAI-compatible API |
| `infrastructure/oauth_refresh.py` | Background TIDE task for token auto-refresh |
| `config/providers.yaml` | Provider configs, fallback chain, models, tiers |
| `data/vault.key` | DPAPI-encrypted API key store |
| `.env` | Env var fallbacks for all keys |

---

## DO NOT TOUCH

- `anthropic-beta: oauth-2025-04-20` — required header, do not change the version string
- OAuth token VALUES — update them, never change the key names or structure
- CLI bridge model list — Opus and Sonnet must route through the bridge
- Vault encryption logic — re-keying invalidates all stored tokens
