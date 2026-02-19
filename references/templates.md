# Templates

## docker-compose

```yaml
services:
  ${AGENT_NAME}:
    image: openclaw/openclaw:latest
    container_name: ${AGENT_NAME}
    restart: unless-stopped
    ports:
      - "${PORT}:18789"
    volumes:
      - ${AGENT_NAME}-data:/home/node/.openclaw
    env_file:
      - ./.env
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun

volumes:
  ${AGENT_NAME}-data:
```

### First-run seeding

After `docker compose up -d`, copy config and workspace into the volume:

```bash
docker cp ./config/openclaw.json ${AGENT_NAME}:/home/node/.openclaw/openclaw.json
docker cp ./workspace/. ${AGENT_NAME}:/home/node/.openclaw/workspace/
docker exec -u root ${AGENT_NAME} chown -R node:node /home/node/.openclaw
docker restart ${AGENT_NAME}
```

### Notes

- Official image runs as `node` (uid 1000), data at `/home/node/.openclaw/`
- `NET_ADMIN` + `/dev/net/tun` for optional Tailscale — harmless if unused
- Control UI at `http://<host>:${PORT}/`

---

## config

Minimal complete `openclaw.json`:

```json
{
  "agents": {
    "defaults": {
      "model": "${MODEL}",
      "workspace": "/home/node/.openclaw/workspace",
      "heartbeat": {
        "every": "${HEARTBEAT}"
      },
      "blockStreamingDefault": "${STREAM_MODE}",
      "maxConcurrent": 4,
      "subagents": {
        "maxConcurrent": 8
      }
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "${BIND}",
    "auth": {
      "mode": "token"
    }
  },
  "channels": {
    ${CHANNEL_CONFIGS}
  },
  "plugins": {
    "entries": {
      ${PLUGIN_ENTRIES}
    }
  },
  "tools": {
    ${TOOLS_CONFIG}
  }
}
```

### Channel config blocks

**Telegram:**
```json
"telegram": {
  "enabled": true,
  "dmPolicy": "${DM_POLICY}",
  "streamMode": "${STREAM_MODE}"
}
```

**Discord:**
```json
"discord": {
  "enabled": true,
  "dmPolicy": "${DM_POLICY}"
}
```

**Nostr:**
```json
"nostr": {
  "enabled": true,
  "privateKey": "${NOSTR_KEY}",
  "dmPolicy": "${DM_POLICY}",
  "relays": ["wss://relay.damus.io", "wss://nos.lol", "wss://relay.nostr.band"],
  "profile": {
    "name": "${AGENT_NAME}",
    "about": "${DESCRIPTION}"
  }
}
```

**WhatsApp:**
```json
"whatsapp": {
  "enabled": true,
  "dmPolicy": "${DM_POLICY}"
}
```

### Plugin entries

Each enabled channel needs a plugin entry:
```json
"telegram": { "enabled": true },
"nostr": { "enabled": true }
```

### Tools config

**Brave Search:**
```json
"tools": {
  "web": {
    "search": {
      "apiKey": "${BRAVE_KEY}"
    }
  }
}
```

---

## workspace

### AGENTS.md

Use the standard OpenClaw AGENTS.md — read from the current agent's workspace and copy as-is. This file is generic project guidance, not personal.

### IDENTITY.md

```markdown
# IDENTITY.md - Who Am I?

- **Name:** ${AGENT_NAME}
- **Creature:** AI assistant
- **Vibe:** ${VIBE}
- **Emoji:** ${EMOJI}
```

### USER.md

```markdown
# USER.md - About Your Human

- **Name:** ${HUMAN_NAME}
- **Pronouns:** ${PRONOUNS}
- **Timezone:** ${TIMEZONE}

## Context

${HUMAN_CONTEXT}
```

### SOUL.md

Use the standard OpenClaw SOUL.md — read from the current agent's workspace and copy as-is. This is generic personality guidance.

### TOOLS.md

```markdown
# TOOLS.md - Local Notes

Add environment-specific notes here as you learn your setup.
```

### HEARTBEAT.md

```markdown
# HEARTBEAT.md

${HEARTBEAT_INSTRUCTIONS}

If nothing needs attention, reply HEARTBEAT_OK.
```
