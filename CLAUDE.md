# CLAUDE.md

This file provides context for Claude Code and other AI agents working with this repository.

## Project Overview

**Moltbot on Cloudflare Workers** - Runs [Moltbot](https://molt.bot/) (OpenClaw) personal AI assistant in a [Cloudflare Sandbox](https://developers.cloudflare.com/sandbox/) container.

- **Docs**: https://docs.openclaw.ai
- **CLI Name**: `clawdbot` (legacy name, being renamed to `openclaw`)
- **Config Path**: `/root/.clawdbot/clawdbot.json` (inside container)

## Architecture

```
Browser/Client
     │
     ▼
Cloudflare Worker (src/index.ts)
     │
     ├── Public routes (no auth)
     ├── Protected routes (Cloudflare Access)
     └── Proxy to container port 18789
            │
            ▼
Cloudflare Sandbox Container
     │
     └── Moltbot Gateway (clawdbot gateway)
            ├── Web UI (Control Dashboard)
            ├── WebSocket for real-time chat
            ├── Telegram/Discord/Slack channels
            └── Skills in /root/clawd/skills/
```

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Main worker entry point, routing, cron handler |
| `src/gateway/process.ts` | Container process management (start, find, health) |
| `src/gateway/env.ts` | Maps worker env vars to container env vars |
| `src/gateway/sync.ts` | R2 backup/restore logic |
| `src/types.ts` | TypeScript types for environment bindings |
| `start-moltbot.sh` | Container startup script, configures clawdbot.json |
| `wrangler.jsonc` | Cloudflare Worker configuration |
| `Dockerfile` | Container image definition |

## Environment Variables

Worker secrets are set via `npx wrangler secret put <NAME>`:

**Required:**
- `ANTHROPIC_API_KEY` or `AI_GATEWAY_API_KEY` - AI provider access
- `MOLTBOT_GATEWAY_TOKEN` - Gateway authentication token
- `CF_ACCESS_TEAM_DOMAIN` / `CF_ACCESS_AUD` - Cloudflare Access for admin UI

**Optional:**
- `BRAVE_API_KEY` - Enables web search tool
- `FIRECRAWL_API_KEY` - Enhanced web content extraction (fallback for web_fetch)
- `OPENROUTER_API_KEY` - Multi-model access (DeepSeek, Qwen, Llama, Gemini free models)
- `CDP_SECRET` / `WORKER_URL` - Enables browser automation
- `TELEGRAM_BOT_TOKEN`, `DISCORD_BOT_TOKEN`, `SLACK_*` - Chat channels
- `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `CF_ACCOUNT_ID` - Persistent storage
- `SANDBOX_SLEEP_AFTER` - Container sleep timeout (default: `never`)

## Adding New Features

### Adding a New Environment Variable

1. **Type definition** - Add to `MoltbotEnv` in `src/types.ts`
2. **Pass to container** - Add mapping in `src/gateway/env.ts` `buildEnvVars()`
3. **Configure in container** - Update `start-moltbot.sh` Node.js config section
4. **Document** - Update `.dev.vars.example`, `wrangler.jsonc`, `README.md`

### Container Configuration

The `start-moltbot.sh` script has an inline Node.js section (lines ~136-290) that:
- Reads `/root/.clawdbot/clawdbot.json`
- Updates config based on environment variables
- Writes updated config back

Config structure follows OpenClaw schema. Check https://docs.openclaw.ai for valid keys.

**Common config sections:**
- `gateway` - Port, auth, mode settings
- `channels` - Telegram, Discord, Slack
- `browser.profiles` - CDP browser connections
- `tools.web.search` - Web search provider config
- `agents.defaults.model` - Default AI model settings
- `models.providers` - Custom provider configurations

### Adding Skills

Skills go in `skills/<skill-name>/`:
- `SKILL.md` - Frontmatter metadata + documentation
- `scripts/` - Executable scripts

Skills are copied to `/root/clawd/skills/` in the container via Dockerfile.

## Common Tasks

### Deploy
```bash
npm run deploy
```

### View Logs
```bash
npx wrangler tail
```

### List Secrets
```bash
npx wrangler secret list
```

### Local Development
```bash
# Create .dev.vars from .dev.vars.example
npm run dev
```

Note: WebSocket proxying has limitations in local dev. Deploy for full testing.

## Cron Jobs

The worker has a cron trigger (`*/5 * * * *`) that syncs container data to R2.

**Important**: The cron handler must call `ensureMoltbotGateway()` before accessing the container, otherwise it will fail if the container is asleep.

## Container Lifecycle

- Default: Container stays alive indefinitely (`keepAlive: true`)
- With `SANDBOX_SLEEP_AFTER=10m`: Container sleeps after 10 minutes of inactivity
- Cold starts take 1-2 minutes
- R2 storage enables persistence across container restarts

## Troubleshooting

**"Unknown config keys" warning**: The key isn't in OpenClaw's config schema. Check docs.

**Cron not firing**: Ensure `ensureMoltbotGateway()` is called before container operations.

**Container not starting**: Check `npx wrangler tail` for errors. Verify secrets are set.

**WebSocket issues locally**: Known limitation of `wrangler dev`. Deploy to test WebSocket.

## Code Patterns

### Reading config in startup script
```javascript
// In start-moltbot.sh Node.js section
if (process.env.MY_VAR) {
    config.section = config.section || {};
    config.section.key = process.env.MY_VAR;
    console.log('Configured my feature');
}
```

### Passing env to container
```typescript
// In src/gateway/env.ts
if (env.MY_VAR) envVars.MY_VAR = env.MY_VAR;
```

### Container port
Always use port `18789` (defined in `src/config.ts` as `MOLTBOT_PORT`).
