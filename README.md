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

### 4. `github-activity-digest.json`
Fetches commits and updated issues from a configurable list of GitHub repos over the last 24 hours, runs the raw activity through a local LLM to produce a short prose summary, and sends it to Telegram.

Useful when you maintain several repos and want one morning message instead of checking each one. Works with private repos (add a GitHub token). No GitHub webhooks needed — pull-based, runs on a schedule.

**Nodes:** Schedule Trigger → Code (configure repos) → HTTP Request (commits) + HTTP Request (issues) → Code (merge) → IF (any activity?) → HTTP Request (LLM) → Code (format) → Telegram

**Configuration:**
- Edit the `Configure Repos` node — replace the repo list with your own
- Set `lookbackHours` to match your schedule interval (default: 24)
- Add a GitHub token to the HTTP Request headers for private repos / higher rate limits
- Swap `http://localhost:11434` in `LLM Summarize` for OpenAI or any OpenAI-compatible endpoint

**Example output:**
```
📊 GitHub Digest — Fri, 27 Mar

Three commits landed in n8n-workflows today: README updates and a new github-activity-digest workflow. No new issues were opened. local-rag-stack had one issue updated with a fix for the Docker Compose volume mapping.
```

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

