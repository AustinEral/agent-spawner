# agent-spawner

Spawn a new OpenClaw agent through conversation. Uses the official Docker setup and non-interactive onboarding, carries over API keys, tools, plugins, and skills from your current agent. You just answer 2-3 questions.

## Install

```bash
clawhub install agent-spawner
```

Or manually:

```bash
cd ~/.openclaw/workspace/skills
git clone https://github.com/AustinEral/agent-spawner.git
```

## What it does

1. Reads your current agent's config (API keys, model, tools, plugins, skills)
2. Asks where to deploy, what to name it, anything special
3. Shows the full plan and waits for confirmation
4. Deploys using the official OpenClaw Docker setup
5. Patches in your config on the running instance
6. Hands you a URL and token — say hello, the new agent bootstraps itself

## Links

- [ClawHub](https://clawhub.ai/AustinEral/agent-spawner)
- [OpenClaw](https://github.com/openclaw/openclaw)
