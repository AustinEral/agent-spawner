---
name: agent-spawner
description: Spawn a new OpenClaw agent through conversation. Uses the official OpenClaw Docker setup, then patches in API keys and config on the running instance. User just answers a couple questions. Use when the user wants to create, spin up, deploy, or provision a new OpenClaw agent.
---

# Agent Spawner

Deploy a new OpenClaw agent conversationally. Uses OpenClaw's official install, then patches the running agent with keys from the current config. User never edits a file.

## Flow

### 1. Read Current Config (silent)

```bash
cat ~/.openclaw/openclaw.json
cat ~/.openclaw/.env 2>/dev/null
```

Note what's available: provider API keys, model, tool keys (Brave Search, etc.). These will be patched into the new agent after it's running.

### 2. Ask

Two questions, maybe three:

1. **"Where should I deploy it?"** — options:
   - **Docker on this machine** — run it locally
   - **Docker on a remote host** — give SSH credentials
   - **Bare metal / already have a machine** — just need the install commands run
2. **"Want to give it a name?"** — optional, for the container/service name
3. **"Anything special about this agent?"** — purpose, who it's for, constraints. Optional.

Don't ask about API keys, model, ports, or any config — carry it all over from the current agent automatically.

### 3. Deploy a Vanilla Agent

Use the official OpenClaw installation. The goal is a running gateway you can talk to via the Control UI.

#### Docker (local or remote via SSH)

```bash
# Clone the repo
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# Non-interactive onboard with the current agent's API key
# This builds the image, runs onboarding, and starts the gateway
ANTHROPIC_API_KEY="<from current config>" ./docker-setup.sh
```

Or if `docker-setup.sh` requires interaction, use the manual flow:

```bash
docker build -t openclaw:local -f Dockerfile .

# Non-interactive onboard
docker compose run --rm openclaw-cli onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "<from current config>" \
  --gateway-port 18789 \
  --gateway-bind lan \
  --skip-skills

# Start the gateway
docker compose up -d openclaw-gateway
```

If deploying remotely, run all of this over SSH.

#### Bare metal

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard

openclaw onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "<from current config>" \
  --gateway-port 18789 \
  --gateway-bind lan \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

### 4. Patch the Running Agent

Once the gateway is up, use `openclaw config set` (via `docker exec` if Docker) to add everything from the current agent's config.

```bash
# Alias for convenience (Docker)
OC="docker exec <container> openclaw"

# Or bare metal
OC="openclaw"

# Set model (if different from default)
$OC config set agents.defaults.model "<model from current config>"

# Set tool keys
$OC config set tools.web.search.apiKey "<brave key from current config>"

# Set heartbeat
$OC config set agents.defaults.heartbeat.every "30m"

# Restart to pick up changes
# Docker:
docker restart <container>
# Bare metal:
openclaw gateway restart
```

Only patch what the current agent actually has configured. Skip anything that's default or empty.

### 5. Verify & Hand Off

```bash
$OC status
```

Tell the user:
- **Control UI:** `http://<host>:18789/`
- **Gateway token:** (from the onboard output or `.env`)
- "Open the Control UI, paste the token, and send a message. The agent will introduce itself."

The new agent bootstraps on first message — it'll ask its own identity questions and set up its personality.

## What NOT to do

- Don't pre-generate openclaw.json — use `openclaw onboard` + `openclaw config set`
- Don't create workspace files — bootstrapping does this
- Don't configure channels — user adds those later when they have bot tokens
- Don't install plugins during setup — user or agent does this after bootstrapping
- Don't ask more than 3 questions
- Don't hand the user commands to run — do it yourself via exec/SSH
