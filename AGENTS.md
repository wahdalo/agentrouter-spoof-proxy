# AgentRouter Spoof Proxy — AI Agent Guide

## What This Is
A Node.js reverse proxy that sits between 9Router and AgentRouter. It injects Claude Code spoof headers and maintains WAF cookies to bypass AgentRouter's restrictions.

## Quick Reference

| Endpoint | Port | Purpose |
|----------|------|---------|
| `agentrouter-proxy` (internal) | `:8318` | This proxy |
| `9router` (internal) | `:20128` | 9Router service |
| `agentrouter.org` | `:443` | Upstream |

## How to Set Up

### 1. Deploy
```bash
docker compose up -d --build
```

### 2. Verify
```bash
curl http://localhost:8318/health
# look for: "wafCookie": true, "circuitOpen": false
```

### 3. Configure 9Router
Add to 9Router's config as an `anthropic-compatible` provider:
```yaml
providers:
  - name: agentrouter
    type: anthropic-compatible
    base_url: http://agentrouter-proxy:8318
    api_key: sk_9router_test
    models:
      - AG/claude-opus-4-6
      - AG/claude-opus-4-7
      - AG/claude-opus-4-8
      - AG/glm-5.2
```

### 4. Test via 9Router
```bash
curl http://localhost:20128/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk_9router_test" \
  -d '{"model": "AG/claude-opus-4-8", "messages": [{"role": "user", "content": "hi"}], "stream": true}'
```

### 5. Configure opencode
Add provider to `~/.config/opencode/opencode.jsonc`:
- npm: `@ai-sdk/openai-compatible`
- baseURL: `http://<SERVER_LAN_IP>:20128/v1` (use `localhost` if same machine)
- models: 3 Claude Opus variants + GLM-5.2, all with `AG/` prefix
- Set API key via `/connect 9router` → `sk_9router_test`

## Architecture

```
LLM client → 9Router → agentrouter-proxy:8318 → agentrouter.org
```

The proxy:
- Rewrites `/messages` → `/v1/messages`
- Injects spoof headers (User-Agent, X-Stainless-*, Anthropic-Beta, etc.)
- Maintains `acw_tc` WAF cookie via periodic warmup (every 3 min)
- Pipes SSE streaming without buffering
- Retries timeouts/5xx with exponential backoff (2 retries)
- Circuit breaker after 5 consecutive failures (up to 10 min backoff)
- Optional model auto-discovery from upstream via `AR_API_KEY`

## Env Vars

| Variable | Default | Description |
|----------|---------|-------------|
| `LISTEN_PORT` | `8318` | Listen port |
| `TARGET_HOST` | `agentrouter.org` | Upstream host |
| `TARGET_PORT` | `443` | Upstream port |
| `MODELS_CSV` | `claude-opus-4-6,...` | Static model fallback list |
| `WARMUP_INTERVAL_MS` | `180000` | WAF cookie refresh interval |
| `AR_API_KEY` | `""` | API key for model auto-discovery |
| `DISCOVERY_INTERVAL_MS` | `600000` | Model list refresh interval (10 min) |

## Important Notes

- Model IDs use `AG/` prefix through 9Router
- The proxy does NOT validate model IDs — unknown ones are forwarded as-is
- Without `AR_API_KEY`, model list is static from `MODELS_CSV`
- Container must be on `9router-net` network for 9Router DNS resolution
- `agentrouter-proxy` is the service name on Docker network (not the folder name)
- `gpt-5.5` always returns 403 (upstream quota) — exclude from config
- `glm-5.2` may hit 429 TPM rate limit
- `NoChannelError` (503) is normal — channels fluctuate, retry or switch models

## Key Files

- `proxy.mjs` — main proxy source (483 lines, single file)
- `Dockerfile` — `FROM node:22-alpine`, HEALTHCHECK on `/health`
- `docker-compose.yml` — service networking, env vars
- `.env` — do NOT commit (gitignored)
