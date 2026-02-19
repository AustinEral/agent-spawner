---
name: agent-spawner
description: Spawn a new OpenClaw agent through conversation. Automatically reads the current agent's config, asks a few questions, then generates and optionally deploys a complete agent — no manual file editing required. Use when the user wants to create, spin up, deploy, or provision a new OpenClaw agent.
---

# Agent Spawner

Deploy a new OpenClaw agent by having a conversation. The agent running this skill does everything — reads config, generates files, copies them to the target, and starts the container. The user just answers questions.

## Flow

### 1. Read Current Config (silent — don't ask)

Automatically read and inventory the current agent's secrets and config:

```bash
# Find all transferable config
cat ~/.openclaw/openclaw.json
cat ~/.openclaw/.env 2>/dev/null
ls ~/.openclaw/extensions/
```

Build an internal list:
- Provider API keys (Anthropic, OpenAI, etc.)
- Channel configs and tokens (Telegram, Discord, Nostr, etc.)
- Tool keys (Brave Search, etc.)
- Installed plugins/extensions
- Gateway settings

### 2. Ask Questions (conversational)

Keep it brief. Ask naturally, not like a form:

1. **"What do you want to call the new agent?"**
2. **"Who's it for?"** — name, timezone
3. **"Want me to use the same API key / channels / search key?"** — present what you found as a checklist. Default to yes for everything unless they say otherwise.
4. **"Where should I deploy it?"** — local Docker, or SSH to a remote host? If remote, get the host/credentials.
5. **"Anything special about this agent?"** — personality, purpose, or skip

That's it. 5 questions max. Don't ask about ports, bind addresses, auth modes, heartbeat intervals, or any technical config — use sensible defaults (port 18789, bind lan, token auth, 30m heartbeat). Only ask if there's a conflict (e.g. same port on same host).

### 3. Generate Everything

Create a temp directory and generate:

**`.env`:**
```bash
# Pull from gathered config
ANTHROPIC_API_KEY=<from current config>
OPENCLAW_GATEWAY_TOKEN=<auto-generate>
# ... any channel tokens selected
```

**`openclaw.json`:**
- Only include non-default settings
- Channels the user selected (with tokens/keys)
- Plugin entries for enabled channels
- Tool config (search keys, etc.)
- Model setting
- Gateway auth mode token

Generate the gateway token automatically:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### 4. Deploy

#### Local Docker
```bash
# Clone OpenClaw repo
git clone https://github.com/openclaw/openclaw.git /tmp/<agent-name>-deploy
cd /tmp/<agent-name>-deploy

# Copy generated config
cp .env /tmp/<agent-name>-deploy/
mkdir -p ~/.openclaw-<agent-name>
cp openclaw.json ~/.openclaw-<agent-name>/openclaw.json

# Run official setup
./docker-setup.sh
```

Adjust as needed — the official `docker-setup.sh` handles building, compose, and startup.

#### Remote Host (SSH)
If the user gave SSH access:
```bash
# SSH in, clone repo, copy files, run setup
ssh user@host "git clone https://github.com/openclaw/openclaw.git ~/openclaw && cd ~/openclaw && ./docker-setup.sh"
scp .env user@host:~/openclaw/
scp openclaw.json user@host:~/.openclaw/
ssh user@host "cd ~/openclaw && docker compose restart"
```

Do all of this automatically. Don't ask the user to run commands.

### 5. Post-Deploy

After the container is running:

1. Install any selected plugins:
   ```bash
   docker exec <container> openclaw plugins install <plugin-name>
   ```
2. Restart if plugins were installed
3. Tell the user:
   - Control UI URL and gateway token
   - "Send it a message to start bootstrapping"
   - Any channels that need interactive login (WhatsApp, Signal)

### Defaults (don't ask unless conflict)

| Setting | Default |
|---------|---------|
| Port | 18789 |
| Bind | lan |
| Auth | token (auto-generated) |
| Heartbeat | 30m |
| Streaming | off |
| DM policy | pairing |
| Model | same as current agent |

### Key Rules

1. **Do the work** — user should never edit a file or run a command
2. **Reuse everything by default** — carry over all keys/tokens unless told not to
3. **Minimal questions** — 5 or fewer
4. **No workspace files** — bootstrapping generates those on first message
5. **Auto-generate secrets** — gateway tokens, Nostr keys if needed
6. **Handle deployment** — clone, copy, start. Don't hand off instructions.
