# Claude Code OAuth — CLI Setup, Bearer Headers & Token Renewal

This document covers how to use Claude Code's OAuth tokens with Trident — including the full auth flow, header requirements, token renewal procedure, and fallback provider setup.

---

## How It Works

Claude Code (`claude` CLI) authenticates via Anthropic's OAuth flow and stores tokens in `~/.claude/.credentials.json`. Trident reads these tokens at boot and uses them for all Anthropic API calls.

### Why OAuth instead of a static API key?

Claude Pro/Max subscriptions authenticate via OAuth. Static API keys are pay-per-token via console.anthropic.com. Either works — Trident detects which you have by the token prefix (`sk-ant-oat*` = OAuth, `sk-ant-api*` = standard key).

### How Trident routes OAuth calls (Agent SDK)

As of April 2026, Anthropic clarified that personal local tools wrapping Claude Code and the Agent SDK are explicitly allowed to use OAuth subscriptions. Trident uses `claude-agent-sdk` as the sole OAuth path for Opus and Sonnet — no subprocess bridge.

### Call flow

```
User message
  -> NexusOrchestrator
  -> ModelRouter.route()
     - Try Anthropic (primary)
       - OAuth token (sk-ant-oat*) -> Agent SDK for ALL models
       - Standard API key (sk-ant-api*) -> direct HTTP for ALL models
     - Anthropic fails -> Kimi K2.5 (moonshot)
     - Kimi fails -> Ollama local
```

---

## Header Reference

### OAuth tokens (sk-ant-oat*) — Primary

Used for health checks and any direct HTTP calls to `/v1/messages` (standard key path):

```python
headers = {
    "Authorization": f"Bearer {access_token}",
    "anthropic-version": "2023-06-01",
    "anthropic-beta": "oauth-2025-04-20",   # required — do NOT change version string
    "content-type": "application/json",
}
```

### Standard API keys (sk-ant-api*) — Direct HTTP path

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

## Agent SDK (Opus/Sonnet via OAuth)

Trident routes all OAuth Opus/Sonnet calls through the `claude-agent-sdk`, which handles OAuth authentication internally using `~/.claude/.credentials.json`. No subprocess, no CLI bridge.

```python
from claude_agent_sdk import query, ClaudeAgentOptions

async for msg in query(
    messages,
    ClaudeAgentOptions(model="claude-opus-4-6", max_tokens=4096),
):
    if isinstance(msg, ResultMessage):
        # done
```

Tools are filtered to primitives only (`read`, `write`, `edit`) before being passed to the Agent SDK. All other tools (agent spawning, memory ops, web, etc.) are intercepted by NexusOrchestrator and never reach the SDK.

**Requirement:** `pip install claude-agent-sdk` and `claude login` must be active on the machine.

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

| Location | Key |
|----------|-----|
| `~/.claude/.credentials.json` | `claudeAiOauth.accessToken` (CLI writes this — Agent SDK reads here) |
| Encrypted vault | `anthropic_api_key` |
| `.env` | `ANTHROPIC_API_KEY`, `TRIDENT_ANTHROPIC_KEY` |
| `os.environ` | Updated at runtime by token sync |

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

Test Opus via Agent SDK:

```python
import anyio
from claude_agent_sdk import query, ClaudeAgentOptions
from claude_agent_sdk.types import ResultMessage

async def test():
    async for msg in query(
        [{"role": "user", "content": "Reply with exactly: OPUS_OK"}],
        ClaudeAgentOptions(model="claude-opus-4-6", max_tokens=20),
    ):
        if isinstance(msg, ResultMessage):
            print(msg.result)

anyio.run(test)
```

---

## Auto-Refresh (TIDE Background Task)

A background task (`oauth_refresh.py`) runs every 15 minutes via TIDE. It:

1. Reads the current token from `~/.claude/.credentials.json`
2. Calls the Anthropic token refresh endpoint
3. Writes the new token back to `.credentials.json`, vault, and `.env`

The Agent SDK reads OAuth credentials directly from `~/.claude/.credentials.json` at call time, so TIDE refresh keeps both paths current. If auto-refresh is working, manual renewal is rarely needed. Check whether it's running if you see unexpected auth failures.

---

## Fallback Provider Chain

When Anthropic is unavailable, Trident cascades:

```
Anthropic (Agent SDK) -> Kimi K2.5 (moonshot) -> Ollama (local)
```

### Kimi K2.5 (moonshot)

| Location | Variable |
|----------|----------|
| Encrypted vault | `moonshot_api_key` |
| `.env` | `MOONSHOT_API_KEY` |
| Provider config | `providers.moonshot.api_key` |

**Model mapping** (router handles automatically):

| Anthropic | Moonshot |
|-----------|----------|
| `claude-opus-4-6` | `kimi-k2.5` |
| `claude-sonnet-4-6` | `kimi-k2.5` |
| `claude-haiku-4-5` | `kimi-k2.5` |

**Headers:**
```python
headers = {
    "Authorization": f"Bearer {moonshot_api_key}",
    "content-type": "application/json",
}
```

**Endpoint:** `https://api.moonshot.ai/v1` (international)

**Models:**
```
kimi-k2.5              — 200K context (default)
kimi-k2-turbo-preview  — 1M context
```

Both require `temperature=1.0`.

### Ollama (local — final fallback)

Ollama serves as last resort for all tiers. Configured in `config/providers.yaml`. Models depend on what's pulled locally (e.g., `qwen3:8b`, `llama3.1:8b`).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `401 Unauthorized` on `/v1/messages` | Using `x-api-key` instead of `Bearer` for OAuth token | Switch to `Authorization: Bearer <token>` + `anthropic-beta: oauth-2025-04-20` |
| Agent SDK raises auth error | `~/.claude/.credentials.json` missing or token expired | Run `claude login` to refresh |
| Agent SDK import error at boot | `claude-agent-sdk` not installed | `pip install claude-agent-sdk` |
| Health check passes but messages fail | `/v1/models` accepts both header types; `/v1/messages` doesn't | Fix the production request headers — health check is misleading |
| `Failed to decrypt tokens` | Encryption key regenerated, vault stale | Re-seed the encrypted vault from `~/.claude/.credentials.json` |
| `No tokens stored` | Encrypted store empty/corrupt | Re-seed encrypted vault |
| `No refresh token` | CLI credentials missing | Re-run `claude login` |
| `oauth-2025-04-01` returns 401 | Wrong beta header version | Must be `oauth-2025-04-20` not `04-01` |
| Fallback never triggers | Model name lookup fails | Check router resolves `claude-opus-4-6` → `kimi-k2.5` |
| Fallback returns 400/404 | Provider prefix not stripped | Send `kimi-k2.5` not `moonshot/kimi-k2.5` |
| Fallback has no tools | Moonshot in `_NO_TOOL_PROVIDERS` | Only Ollama/local should strip tools |
| Auto-refresh doesn't update vault | Token sync skipped encrypted store | Ensure sync writes to ALL destinations including vault |
| TIDE refresh ran but Agent SDK still fails | SDK read credentials before refresh completed | Next call will pick up fresh token automatically |

---

## Critical Files (Reference)

| File | Purpose |
|------|---------|
| `~/.claude/.credentials.json` | Fresh OAuth token — Claude CLI writes, Agent SDK reads |
| `runner/providers/anthropic.py` | OAuth detection, Agent SDK routing, direct HTTP for API keys |
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
- Vault encryption logic — re-keying invalidates all stored tokens
- `~/.claude/.credentials.json` key structure — Agent SDK and oauth_refresh.py both depend on it
