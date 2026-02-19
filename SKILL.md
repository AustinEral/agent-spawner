---
name: agent-spawner
description: Spawn a new OpenClaw agent through conversation. Automatically carries over API keys and config from the current agent, deploys via Docker, and gets a working gateway with Control UI access. User just answers 2-3 questions. Use when the user wants to create, spin up, deploy, or provision a new OpenClaw agent.
---

# Agent Spawner

Deploy a new OpenClaw agent by having a conversation. The agent running this skill does all the work — reads config, generates files, deploys the container, and hands back a URL + token.

## Flow

### 1. Read Current Config (silent)

Read the current agent's config without asking:

```bash
cat ~/.openclaw/openclaw.json
cat ~/.openclaw/.env 2>/dev/null
```

Extract:
- Provider API keys (Anthropic, OpenAI, etc.)
- Model setting
- Tool keys (Brave Search, etc.)
- Installed plugins/extensions list

These carry over automatically. Don't ask.

### 2. Ask Questions

Three questions max:

1. **"Where should I deploy it?"** — local Docker or SSH to a remote host? If remote, get host/credentials.
2. **"Want to give it a name?"** — for the Docker container. Optional, generate one if they don't care.
3. **"Anything special about this agent?"** — purpose, constraints, or skip. This is just context for the user's own reference, not config.

Don't ask about:
- Personality/identity (the agent develops that itself through bootstrapping)
- Channels (they'll set up their own channels later — start with just the Control UI/web chat)
- Ports, bind, auth, heartbeat, streaming (use defaults)
- Which keys to carry over (carry over all of them)

### 3. Generate Config

Create a temp working directory and generate two files:

**`.env`:**
```bash
# Carried over from current agent
ANTHROPIC_API_KEY=<from config>
# OPENAI_API_KEY=<if exists>
OPENCLAW_GATEWAY_TOKEN=<auto-generate>
```

Auto-generate the gateway token:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

**`openclaw.json`:**
```json
{
  "agents": {
    "defaults": {
      "model": "<same as current agent>"
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "lan",
    "auth": {
      "mode": "token"
    }
  },
  "tools": {
    "web": {
      "search": {
        "apiKey": "<if current agent has one>"
      }
    }
  }
}
```

Minimal. No channels (Control UI works out of the box). No plugins. No workspace config. Just provider auth, gateway, and tools.

### 4. Deploy

#### Local Docker

```bash
cd /tmp
git clone https://github.com/openclaw/openclaw.git <agent-name>
cd <agent-name>
cp <generated>/.env .
./docker-setup.sh
```

Then copy the config into the running container:
```bash
docker cp <generated>/openclaw.json <container>:/home/node/.openclaw/openclaw.json
docker exec -u root <container> chown node:node /home/node/.openclaw/openclaw.json
docker restart <container>
```

#### Remote Host (SSH)

Do it all over SSH — clone, copy files, run setup. Don't hand the user commands to run.

```bash
ssh user@host "git clone https://github.com/openclaw/openclaw.git ~/openclaw"
scp .env user@host:~/openclaw/
ssh user@host "cd ~/openclaw && ./docker-setup.sh"
scp openclaw.json user@host:~/.openclaw/
ssh user@host "docker restart <container>"
```

### 5. Hand Off

Tell the user:
- **Control UI URL:** `http://<host>:18789/`
- **Gateway token:** (the one you generated)
- "Send it a message in the Control UI to get started — it'll introduce itself and set up its own identity."

That's it. The new agent bootstraps itself on first message.

### Defaults (never ask)

| Setting | Value |
|---------|-------|
| Port | 18789 |
| Bind | lan |
| Auth | token (auto-generated) |
| Heartbeat | 30m |
| Streaming | off |
| Channels | none (Control UI only) |
| Plugins | none |
| Model | same as current agent |

### Rules

1. **Carry over all keys silently** — API keys, search keys, everything
2. **No channels** — start with just gateway + Control UI. User adds channels later.
3. **No workspace files** — bootstrapping handles identity, personality, everything
4. **No personality config** — the agent figures out who it is on its own
5. **Do the deployment** — user never runs a command or edits a file
6. **3 questions max**
