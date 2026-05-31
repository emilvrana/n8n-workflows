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

### 4. `ai-cost-monitor.json`
Polls the OpenAI and Anthropic usage APIs daily, compares spending against configured monthly budget thresholds, and sends a Telegram alert if you're approaching or over budget. Sends a quiet green summary if everything is under control.

Useful when you're running multiple AI projects and want one Telegram message instead of logging into two dashboards to check spend.

**Nodes:** Schedule Trigger → Code (configure budgets) → HTTP Request (OpenAI usage) + HTTP Request (Anthropic usage) → Code (aggregate) → IF (any alerts?) → Code (format) → Telegram

**Configuration:**
- Set `monthlyBudget` and `alertThreshold` per provider in the `Configure` node
- Add HTTP Header Auth credentials: `Authorization: Bearer <key>` for OpenAI, `x-api-key: <key>` for Anthropic
- Set `TELEGRAM_CHAT_ID` env var or hardcode in the Telegram nodes
- Default schedule: 9:00 AM daily. Adjust in Schedule Trigger

**Example alert output:**
```
⚠️ AI API Cost Alert — 2026-03-01 to 2026-03-29

🟡 OpenAI: $41.20 / $50 (82.4%)
`████████░░`
🟢 Anthropic: $34.50 / $100 (34.5%)
`███░░░░░░░`
```

### 5. `github-activity-digest.json`
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

### 6. `ssl-expiry-monitor.json`
Checks SSL certificate expiry dates for a configurable list of domains. Runs on a schedule (default: daily). If any certificate is within the warning threshold (default: 14 days) or critical threshold (default: 7 days), sends an alert to Telegram with 🔴/🟡 icons. If everything is fine, sends a quiet 🟢 summary.

Uses `openssl s_client` on the n8n host — so it tests the actual certificate chain your users see, not just the domain's stated expiry. Works for Let's Encrypt, commercial CAs, and self-signed certs alike.

**Nodes:** Schedule Trigger → Code (configure domains) → Execute Command (openssl) → Code (parse + threshold) → IF (alert needed?) → Code (format) → Telegram

**Configuration:**
- Edit the `Configure Domains` node — add your domains and adjust `warningDays` / `criticalDays`
- Set `TELEGRAM_CHAT_ID` env var or hardcode in the Telegram nodes
- Requires `openssl` on the n8n host (available on most Linux/macOS systems)

**Example alert output:**
```
🔴 SSL Certificate Alert

🔴 aiadoption.cz: 3 days left (expires 2026-05-28)
🟡 staging.aiadoption.cz: 12 days left (expires 2026-06-06)
```

### 7. `log-disk-monitor.json`
Checks disk usage of configurable log directories on a schedule (default: every 6 hours). Calculates usage percentage against partition capacity, compares against warning and critical thresholds per directory, and sends a Telegram alert with visual progress bars when thresholds are breached. Sends a quiet 🟢 summary when everything is fine.

Useful for: catching log bloat before it fills a partition and takes down services. Docker containers are notorious for unbounded logs — this catches it early.

**Nodes:** Schedule Trigger → Code (configure directories) → Execute Command (du + df) → Code (parse + threshold) → IF (alert needed?) → Code (format) → Telegram

**Configuration:**
- Edit the `Configure Directories` node — add your log directories and set `warningPercent` / `criticalPercent`
- Set `TELEGRAM_CHAT_ID` env var or hardcode in the Telegram nodes
- Requires `du` and `df` on the n8n host (available on all Linux/macOS systems)

**Example alert output:**
```
⚠️ Log Disk Usage Alert — 2026-05-26

🔴 /var/log: 890.2 MB used (88%) `████████░░`
🟡 /opt/appdata/logs: 340.5 MB used (65%) `█████░░░░░`
```

### 8. `docker-compose-backup.json`
Scheduled backup of Docker Compose configurations with Telegram notifications. Archives `.env`, `docker-compose.yml`, and config files to timestamped tarballs, rotates backups older than 7 days, and sends success/failure alerts.

Useful for: disaster recovery for self-hosted infrastructure without relying on full VM backups. Captures the *configuration* (the valuable, hard-to-rebuild part) separately from data volumes.

**Nodes:** Schedule Trigger → Execute Command (backup) → Execute Command (cleanup) → IF (check success) → Telegram

### 9. `url-monitor.json`
Checks a configurable list of URLs on a schedule (default: every 5 minutes). Measures response time and status code against expected values, and sends a Telegram alert when a URL is down or slow. Sends a quiet 🟢 summary when everything is healthy.

Complements `scheduled-healthcheck.json` — that one checks internal endpoints, this one checks external URLs your users actually visit. Tracks response time, not just status codes, so you catch degradation before it becomes downtime.

**Nodes:** Schedule Trigger → Code (configure URLs) → HTTP Request (check each URL) → Code (evaluate results) → Code (format message) → IF (has alert?) → Telegram Alert / Telegram OK

**Configuration:**
- Edit the `Configure URLs` Code node — add your URLs with `name`, `url`, `expectedStatus`, and `maxResponseTimeMs`
- Set `TELEGRAM_CHAT_ID` env var or hardcode in the Telegram nodes
- Default schedule: every 5 minutes. Adjust in Schedule Trigger
- A URL is flagged as 🟡 slow when response time exceeds 80% of `maxResponseTimeMs`, and 🔴 down when it exceeds `maxResponseTimeMs` or returns an unexpected status

**Example alert output:**
```
⚠️ URL Monitor Alert

🔴 API Health: 503 (4210ms, expected 200)
🟡 Main Site: 200 (4320ms, slow, limit 5000ms)
```

**Example healthy output:**
```
✅ All URLs healthy

🟢 Main Site: 200 (340ms)
🟢 API Health: 200 (120ms)
🟢 RSS Feed: 200 (890ms)
```

### 10. `slack-to-telegram-bridge.json`
Forwards Slack messages from specified channels to Telegram in real time. Filters out bot messages, formats with channel name and sender, and delivers a clean Markdown notification.

Useful when your team uses Slack but you want key messages forwarded to your personal Telegram — or when bridging notifications between two organizations that use different platforms.

**Nodes:** Slack Trigger → IF (filter bots) → Set (format message) → Telegram

**Configuration:**
- Add Slack API credentials (requires a Slack app with `message.channels` event subscription)
- Add Telegram API credentials (bot token from @BotFather)
- Set `TELEGRAM_CHAT_ID` env var or hardcode in the Telegram node
- Slack channel is set dynamically from the webhook payload — no hardcoding needed

**Example output:**
```
💬 *Slack → #deployments*
*@jan*: Production deploy v2.4.1 complete ✅
⏰ 14:05 28.05.2026
```

### 11. `postgres-health-check.json`
Runs a single `psql` query on a schedule (default: every hour) that checks connection usage, replication lag, database sizes, and idle-in-transaction sessions. Evaluates results against configurable thresholds and sends a Telegram alert with 🔴/🟡 icons when something is off — or a quiet 🟢 summary when everything is fine.

Complements `scheduled-healthcheck.json` and `url-monitor.json` — those check HTTP endpoints and external URLs, this checks your actual database health. Catches the things that slow creep up on you: connections leaking, replication falling behind, databases quietly growing.

**Nodes:** Schedule Trigger → Code (configure thresholds) → Execute Command (psql query) → Code (evaluate results) → IF (alert needed?) → Code (format) → Telegram Alert / Telegram OK

**Configuration:**
- Edit the `Configure` node — set `host`, `port`, `database`, `user`, and thresholds
- Store password in `PGPASSWORD` env var or n8n credentials
- Set `TELEGRAM_CHAT_ID` env var or hardcode in the Telegram nodes
- Requires `psql` on the n8n host

**Example alert output:**
```
⚠️ Postgres Health Check

🔴 Connections: 182/200 (91%)
🔴 Idle in transaction: 3 sessions
```

**Example healthy output:**
```
✅ Postgres Health Check

🟢 Connections: 34/200 (17%)
🟢 Replication lag: 0.1s
🟢 Idle in transaction: 0
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

