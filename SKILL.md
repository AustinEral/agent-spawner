---
name: agent-spawner
description: Spawn a new OpenClaw agent through conversation. Uses official Docker setup and non-interactive onboarding, carries over API keys, tools, plugins, and skills from the current agent. User answers 2-3 questions. Use when the user wants to create, spin up, deploy, or provision a new OpenClaw agent.
---

# Agent Spawner

Deploy a new OpenClaw agent conversationally. Official install, then carry over config from the current agent. User never edits a file.

## 1. Read Current Config (silent)

```bash
cat ~/.openclaw/openclaw.json
cat ~/.openclaw/.env 2>/dev/null
echo $ANTHROPIC_API_KEY
ls ~/.openclaw/extensions/
ls <workspace>/skills/
```

Note: provider API key, model, tool keys, installed plugins, workspace skills.

## 2. Ask

1. **"Where should I deploy it?"** — Docker (local or remote SSH) or bare metal?
2. **"Name?"** — for container. Generate one if they don't care.
3. **"Anything special?"** — purpose, constraints. Optional.

Don't ask about keys, plugins, skills, ports, or config. Carry everything over, use defaults.

## 3. Deploy

### Docker

```bash
git clone https://github.com/openclaw/openclaw.git <agent-name>
cd <agent-name>
```

Set env and run non-interactive onboard:

```bash
export OPENCLAW_IMAGE=alpine/openclaw:latest
export OPENCLAW_CONFIG_DIR=~/.openclaw-<agent-name>
export OPENCLAW_WORKSPACE_DIR=~/.openclaw-<agent-name>/workspace
export OPENCLAW_GATEWAY_PORT=<unused port, default 18789>
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

docker compose up -d openclaw-gateway
```

The official compose uses **bind mounts** from `OPENCLAW_CONFIG_DIR` — host user owns the files, no permission issues.

The onboard error about gateway connection is expected (gateway wasn't running yet). Config is already written.

### Bare metal

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

## 4. Patch Running Agent

CLI inside Docker: `docker compose exec openclaw-gateway node /app/openclaw.mjs`
Bare metal: `openclaw`

**Config:**
```bash
$OC config set tools.web.search.apiKey "<brave key>"
$OC config set agents.defaults.heartbeat.every "30m"
$OC config set agents.defaults.model "<model>"
```

**Plugins** (installed into `~/.openclaw/extensions/`, persists in volume):
```bash
$OC plugins install <plugin-name>
$OC config set plugins.entries.<plugin-name>.enabled true
```

Repeat for each plugin the current agent has. Check `plugins.installs` in current config for the list.

**Skills** (workspace skills persist in volume):
```bash
# Copy from current agent's workspace
docker cp <current-workspace>/skills/ <container>:/home/node/.openclaw/workspace/skills/
# Or for bare metal, just cp the directory
```

**Restart:**
```bash
docker compose restart openclaw-gateway  # Docker
openclaw gateway restart                 # bare metal
```

## 5. Hand Off

Read the gateway token from config:
```bash
cat $OPENCLAW_CONFIG_DIR/openclaw.json | grep -A1 '"token"'
```

Tell the user:
- **URL:** `http://<host>:<port>/`
- **Token:** (from config — onboard auto-generates one)
- "Say hello — it'll bootstrap itself."

## Notes

- `openclaw` is NOT in PATH inside the Docker container. Use `node /app/openclaw.mjs`.
- `--accept-risk` is required for non-interactive onboard.
- Use `alpine/openclaw:latest` — pre-built, skip source build.
- Don't use named Docker volumes — they create root-owned dirs. Official compose uses bind mounts.
- Multiple agents on same host: different `OPENCLAW_CONFIG_DIR` and `OPENCLAW_GATEWAY_PORT`.
- Plugins and skills persist in `~/.openclaw/` (extensions/ and workspace/skills/).
- SSH keys, git config, apt packages are ephemeral — not in the volume, by design.
