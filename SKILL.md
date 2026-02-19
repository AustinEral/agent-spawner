---
name: agent-spawner
description: Spawn a new OpenClaw agent using Docker. Interactive setup that walks the user through identity, provider, channels, and gateway config. Generates a ready-to-deploy directory with docker-compose.yml, openclaw.json, .env, and workspace files. Use when the user wants to create, spin up, deploy, clone, or provision a new OpenClaw agent.
---

# Agent Spawner

Generate a complete, ready-to-deploy OpenClaw agent. Produces a directory you can `docker compose up` on any machine.

## Output

```
<agent-name>/
├── docker-compose.yml
├── .env
├── config/
│   └── openclaw.json
└── workspace/
    ├── AGENTS.md
    ├── IDENTITY.md
    ├── USER.md
    ├── SOUL.md
    ├── TOOLS.md
    └── HEARTBEAT.md
```

## Interactive Setup

Walk through each step in order. Ask one group at a time.

### Step 1: Agent Identity

Ask:
- **Agent name** (used for container name, config, workspace)
- **One-line description** of what the agent does
- **Emoji** (optional, for personality)

### Step 2: Human Info

Ask:
- **Human's name**
- **Timezone** (e.g. `America/Los_Angeles`)
- **Pronouns** (optional)
- **Brief context** — what they do, communication preferences

### Step 3: AI Provider

Ask:
- **Provider** — Anthropic (recommended), OpenAI, or custom
- **API key** — check if the current agent has one to reuse (look in `~/.openclaw/.env` and config). Offer to carry it over.
- **Default model** — suggest `anthropic/claude-sonnet-4-20250514` for Anthropic, or let user pick

### Step 4: Channels

Ask which channels to enable. For each, collect the required credential:

| Channel | Required | Notes |
|---------|----------|-------|
| Telegram | Bot token from @BotFather | Ready immediately |
| Discord | Bot token | Ready immediately |
| WhatsApp | — | Needs QR login post-deploy |
| Signal | — | Needs signal-cli post-deploy |
| Nostr | Hex private key (generate if needed) | Ready immediately |
| Web/CLI | — | Always available via Control UI |

Generate Nostr keys with: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`

For each channel, ask **DM policy**: `pairing` (default, safe) or `allowlist`.

### Step 5: Tools & Search

Ask:
- **Brave Search API key** — for `web_search` tool. Check if current agent has one to reuse.
- Any other tool config the user wants

### Step 6: Gateway

Ask:
- **Port** — default `18789`
- **Bind** — `lan` (recommended for Docker) or `loopback`
- **Auth token** — auto-generate one: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`

### Step 7: Behavior

Ask:
- **Heartbeat interval** — default `30m`
- **Streaming** — `on` or `off` (off recommended for Telegram)
- **Anything special** for SOUL.md (personality, tone, rules)
- **HEARTBEAT.md instructions** — what should the agent check periodically?

### Step 8: Review & Generate

Show a summary of all choices. Confirm, then generate all files.

## File Generation

### docker-compose.yml

See `references/templates.md` § docker-compose.

### .env

All secrets go here. Never inline secrets in openclaw.json.

```bash
ANTHROPIC_API_KEY=...
OPENCLAW_GATEWAY_TOKEN=...
# TELEGRAM_BOT_TOKEN=...  (if enabled)
# DISCORD_BOT_TOKEN=...   (if enabled)
```

OpenClaw reads standard env vars automatically for channel tokens and API keys.

### config/openclaw.json

See `references/templates.md` § config. Build from gathered answers. Reference env vars for secrets — do not hardcode.

### Workspace files

See `references/templates.md` § workspace. Generate fresh content based on identity/human info. Do NOT copy memory, sessions, or state from the current agent.

## Post-Generation Instructions

Tell the user:

1. **Copy** the directory to the target machine (if remote)
2. **Run** `docker compose up -d`
3. **Open** `http://<host>:<port>/` in a browser
4. **Paste** the gateway token from `.env` to authenticate
5. **Chat** — the agent is ready

If WhatsApp/Signal enabled:
- Run `docker compose exec <name> openclaw channels login` for QR/setup

If any plugins need installing:
- Run `docker compose exec <name> openclaw plugins install <plugin-name>`

## Key Rules

1. **Fresh agent** — no memory, sessions, or personal data carried over
2. **Secrets in .env only** — never in openclaw.json or workspace files  
3. **Complete config** — agent must work on first `docker compose up`, no manual editing needed
4. **Don't generate files the agent makes itself** — no BOOTSTRAP.md, no memory/, no sessions
5. **Ask before reusing** — if carrying over API keys or tokens from current config, confirm with user first
