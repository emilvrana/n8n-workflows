# n8n Workflows

A collection of practical n8n workflow templates I use in production. These pair with my post [n8n for Developers: When Workflow Automation Beats Writing Code](https://emil.aiadoption.cz/blog/n8n-for-developers/).

Self-hosted n8n, running on a VPS. No n8n Cloud required.

## Workflows

### 1. `webhook-to-telegram.json`
Receives an incoming webhook, sends the payload to an LLM for a brief summary, then delivers the result to a Telegram chat. 

Useful for: alerting pipelines, CI/CD notifications, anything that POSTs a JSON payload and you want a human-readable message in Telegram instead of a wall of JSON.

**Nodes:** Webhook → Code (extract fields) → HTTP Request (OpenAI/local LLM) → Telegram

**Example Webhook Payload:**

The workflow is generic, but here's an example of what it might expect for a CI/CD notification:

```json
{
  "repository": "emilvrana/n8n-workflows",
  "status": "success",
  "commit": "a1b2c3d",
  "author": "Emil Vrána",
  "message": "feat: Add example payload to README"
}
```

### 2. `scheduled-healthcheck.json`
Pings a list of HTTP endpoints on a schedule. If any return a non-2xx status or time out, sends a Telegram alert with the failing service name and status code.

Better than uptime services for internal/private endpoints that aren't reachable from the outside. Runs inside your VPS, so it tests real internal connectivity too.

**Nodes:** Schedule Trigger → HTTP Request (each endpoint) → IF (check status) → Telegram

### 3. `rss-ai-digest.json`
Fetches a set of RSS feeds, deduplicates against previously seen items, runs each new item title through a summarization prompt, then compiles a daily digest and sends it via email or Telegram.

I use this for AI/tech news. Gets me the signal without the scroll.

**Nodes:** Schedule Trigger → RSS Feed Read → Code (deduplicate) → HTTP Request (LLM summarize) → Merge → Send Email / Telegram

## Requirements

- A running [n8n](https://n8n.io/) instance (self-hosted recommended).
- An LLM API endpoint (e.g., local Ollama, OpenAI, Anthropic).
- A Telegram Bot Token and Chat ID.
- Basic familiarity with n8n concepts like credentials and nodes.

## How to import

1. In your n8n instance: **Workflows → Import from File**
2. Select the `.json` file
3. Update credentials (Telegram bot token, OpenAI/LLM API key, email SMTP)
4. Activate

## Setup notes

- **LLM calls**: workflows use `http://localhost:11434` (Ollama) by default. Swap the URL and auth header for OpenAI, Anthropic, or any OpenAI-compatible endpoint.
- **Telegram**: you'll need a bot token from [@BotFather](https://t.me/botfather) and your chat ID
- **Credentials**: n8n stores creds separately from workflow JSON — you won't accidentally export secrets when sharing workflows

## My stack

```
n8n             self-hosted, Docker, VPS
LLM backend     llama.cpp server (local) or OpenAI API
Telegram        notifications and quick I/O
Traefik         reverse proxy + SSL
```

More on the infrastructure: [Running a Local LLM on Your Own Server](https://emil.aiadoption.cz/blog/local-llm-server/)

---

Issues and PRs welcome. These are production configs, so I keep them lean.

## License

[MIT](./LICENSE)

