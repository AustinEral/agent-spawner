---
name: agent-spawner
description: Spawn a new OpenClaw agent through conversation. Uses the official OpenClaw Docker setup and non-interactive onboarding, then patches in extra config on the running instance. User just answers a couple questions. Use when the user wants to create, spin up, deploy, or provision a new OpenClaw agent.
---

# Agent Spawner

Deploy a new OpenClaw agent conversationally. Uses OpenClaw's official install flow, patches the running instance with keys from the current agent. User never edits a file.

## Flow

### 1. Read Current Config (silent)

```bash
cat ~/.openclaw/openclaw.json
cat ~/.openclaw/.env 2>/dev/null
echo $ANTHROPIC_API_KEY  # may be in env
```

Note what's available: provider API key, model, tool keys (Brave Search, etc.).

### 2. Ask

1. **"Where should I deploy it?"** — Docker (this machine or remote via SSH) or bare metal
2. **"Want to give it a name?"** — optional, for the container name. Generate one if they don't care.
3. **"Anything special about this agent?"** — purpose, who it's for. Optional.

That's it. Don't ask about API keys, channels, ports, or config.

### 3. Deploy

Pick the right method based on the user's answer.

#### Method A: Docker (recommended)

Works on the current machine or remotely over SSH.

**Step 1: Clone the repo**
```bash
git clone https://github.com/openclaw/openclaw.git /path/to/<agent-name>
cd /path/to/<agent-name>
```

**Step 2: Non-interactive onboard**

The official compose uses bind mounts from `OPENCLAW_CONFIG_DIR` and `OPENCLAW_WORKSPACE_DIR` on the host — NOT named Docker volumes. This means the host user owns the files and there are no permission issues.

```bash
export OPENCLAW_IMAGE=alpine/openclaw:latest   # use pre-built image, skip build
export OPENCLAW_CONFIG_DIR=~/.openclaw-<agent-name>
export OPENCLAW_WORKSPACE_DIR=~/.openclaw-<agent-name>/workspace
export OPENCLAW_GATEWAY_PORT=<pick unused port, default 18789>
export OPENCLAW_GATEWAY_BIND=lan
export ANTHROPIC_API_KEY=<from current agent>

mkdir -p $OPENCLAW_CONFIG_DIR/workspace

docker compose run --rm openclaw-cli onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind lan \
  --skip-skills
```

This creates:
- `openclaw.json` with auth, gateway, workspace config
- Workspace files (AGENTS.md, SOUL.md, BOOTSTRAP.md, etc.)
- Gateway token (auto-generated, written into openclaw.json)
- Sessions directory

The onboard will show an error about connecting to the gateway — ignore it, the gateway isn't running yet.

**Step 3: Start the gateway**
```bash
docker compose up -d openclaw-gateway
```

**Step 4: Verify**
```bash
docker compose logs --tail 10 openclaw-gateway
curl -s -o /dev/null -w '%{http_code}' http://localhost:$OPENCLAW_GATEWAY_PORT/
```

Should see `listening on ws://0.0.0.0:18789` and HTTP 200.

#### Method B: Bare metal

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard

openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "<from current agent>" \
  --gateway-port 18789 \
  --gateway-bind lan \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

### 4. Patch the Running Agent

After the gateway is up, use `openclaw config set` to add extras from the current agent.

For Docker, exec into the compose service:
```bash
docker compose exec openclaw-gateway node /app/openclaw.mjs config set <key> <value>
```

For bare metal:
```bash
openclaw config set <key> <value>
```

Patch what the current agent has:
```bash
# Search API key
config set tools.web.search.apiKey "<brave key>"

# Heartbeat interval
config set agents.defaults.heartbeat.every "30m"

# Model (if different from default)
config set agents.defaults.model "<model>"
```

Then restart:
```bash
# Docker
docker compose restart openclaw-gateway

# Bare metal
openclaw gateway restart
```

### 5. Hand Off

Read the gateway token from the config:
```bash
cat $OPENCLAW_CONFIG_DIR/openclaw.json | grep -A1 '"token"'
# Or for bare metal: cat ~/.openclaw/openclaw.json | grep -A1 '"token"'
```

Tell the user:
- **Control UI:** `http://<host>:<port>/`
- **Gateway token:** (from config)
- "Open the Control UI, paste the token, and say hello. The agent will bootstrap itself."

The new agent has BOOTSTRAP.md in its workspace — on first message it will ask identity questions and set up its personality.

## Gotchas Discovered During Testing

### Use bind mounts, not named volumes
The official docker-compose uses `OPENCLAW_CONFIG_DIR` and `OPENCLAW_WORKSPACE_DIR` as bind mounts. Do NOT use named Docker volumes — they create directories owned by root, and the container runs as `node` (uid 1000), causing permission errors on canvas, cron, and config writes.

### Use `alpine/openclaw:latest` to skip building
Set `OPENCLAW_IMAGE=alpine/openclaw:latest` to use the pre-built image. Otherwise `docker-setup.sh` builds from source which is slow.

### CLI path inside Docker
`openclaw` is NOT in PATH inside the official container. Use:
```bash
node /app/openclaw.mjs <command>
```

### Onboard gateway connection error is expected
`openclaw onboard` tries to connect to the gateway at the end. Since the gateway isn't running yet during `docker compose run`, it errors with "gateway closed (1006)". This is fine — config is already written.

### `--accept-risk` is required
Non-interactive onboarding requires `--accept-risk` flag.

### CLAUDE_* env var warnings
Docker compose warns about unset `CLAUDE_AI_SESSION_KEY`, `CLAUDE_WEB_SESSION_KEY`, `CLAUDE_WEB_COOKIE` vars. These are harmless — they're for Claude.ai web auth which most users don't use.

### Multiple agents on the same host
Use different `OPENCLAW_GATEWAY_PORT` values and different `OPENCLAW_CONFIG_DIR` paths. Each agent gets its own config directory and port.

## What NOT to do

- Don't pre-generate openclaw.json — use `openclaw onboard --non-interactive`
- Don't create workspace files — onboarding does this
- Don't configure channels — user adds those later
- Don't use named Docker volumes — use bind mounts
- Don't ask more than 3 questions
- Don't hand the user commands to run — do it yourself
