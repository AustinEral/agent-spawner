---
name: agent-spawner
description: Spawn a new OpenClaw agent fast by carrying over API keys, channel tokens, skills, and search keys from the current agent's config. Eliminates the tedious re-entry of secrets and credentials. Use when the user wants to create, spin up, deploy, or provision a new OpenClaw agent.
---

# Agent Spawner

Speed up new agent setup by selectively exporting secrets and config from the current agent. OpenClaw's official Docker setup and bootstrapping handle everything else.

## What This Skill Does

1. Reads the current agent's config and credentials
2. Asks the user what to carry over
3. Generates a `.env` and `openclaw.json` with just the selected secrets/config
4. Provides copy-paste deploy instructions using OpenClaw's official Docker flow

## What This Skill Does NOT Do

- Generate workspace files (bootstrapping does this)
- Generate docker-compose.yml (OpenClaw ships one)
- Replace the onboarding wizard (it complements it)

## Setup Flow

### Step 1: Inventory

Read the current agent's config and find what's transferable:

```
~/.openclaw/openclaw.json    — channels, plugins, tools, provider config
~/.openclaw/.env             — API keys, tokens
```

Build a checklist of what exists:
- [ ] AI provider key (Anthropic, OpenAI, etc.)
- [ ] Channel tokens (Telegram bot, Discord bot, Nostr key, etc.)
- [ ] Search API key (Brave)
- [ ] Installed plugins/skills
- [ ] Any custom provider config

Present this list to the user.

### Step 2: Select

Ask the user which items to carry over. For each:
- **API keys** — "Carry over your Anthropic key?" (yes/no)
- **Channels** — "Which channels? Telegram, Discord, Nostr..." (pick)
- **Channel tokens** — for each selected channel, carry over the existing token or enter a new one
- **Search keys** — "Carry over Brave Search key?" (yes/no)  
- **Skills/plugins** — "Which plugins? openclaw-agent-reach, ..." (pick)

Also ask:
- **New agent name** — for the container
- **Gateway port** — default 18789, but if running on same host need a different port
- **Gateway token** — auto-generate one

### Step 3: Generate

Produce two files:

**`.env`** — all selected secrets:
```bash
ANTHROPIC_API_KEY=...
OPENCLAW_GATEWAY_TOKEN=...
TELEGRAM_BOT_TOKEN=...
```

**`openclaw.json`** — minimal config with selected channels, plugins, and tools. Only include sections the user selected. Let OpenClaw defaults handle everything else.

Important:
- Do NOT inline secrets in openclaw.json — reference env vars where possible
- Do NOT include workspace paths (Docker default is correct)
- Do NOT include agent identity/personality (bootstrapping handles this)
- Keep the config minimal — only what differs from OpenClaw defaults

### Step 4: Deploy Instructions

Give the user these steps:

```bash
# 1. Clone the OpenClaw repo (has docker-compose + Dockerfile)
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 2. Copy your generated files
cp /path/to/.env .
cp /path/to/openclaw.json ~/.openclaw/openclaw.json
# (or mount them — adjust docker-compose.yml volumes as needed)

# 3. Run the official Docker setup
./docker-setup.sh

# 4. Install selected plugins (post-start)
docker compose exec openclaw-gateway openclaw plugins install <plugin-name>
```

Or if deploying to a remote host, tell them to scp the `.env` and `openclaw.json` first.

Adjust instructions based on whether they're deploying locally or remotely. The official `docker-setup.sh` handles image building, compose, and startup.

### Step 5: Post-Deploy Checklist

Remind the user:
- Open the Control UI and paste the gateway token
- The agent will bootstrap on first message (identity Q&A)
- If WhatsApp/Signal: run `openclaw channels login` for interactive setup
- If plugins were selected: install them and restart
- Verify with `openclaw status` inside the container
