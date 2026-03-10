# OpenClaw WhatsApp Cloud API Channel

WhatsApp channel for [OpenClaw](https://github.com/openclaw/openclaw) using Meta's **official Cloud API** — production-safe, no Baileys, no ban risk.

## Why this plugin?

OpenClaw's built-in WhatsApp channel uses [Baileys](https://github.com/WhiskeySockets/Baileys), a reverse-engineered WhatsApp Web protocol. It works great for personal use, but Meta can ban accounts at any time — making it unsuitable for business bots.

This plugin uses the **official WhatsApp Cloud API** (`graph.facebook.com`) instead:

| | Built-in (Baileys) | This plugin (Cloud API) |
|---|---|---|
| Auth | QR code scan | OAuth access token |
| Ban risk | High (unofficial) | None (official) |
| 24-hour window | No restriction | Required (templates after 24h) |
| Cost | Free | Pay-per-conversation |
| Sending first | Anytime | Templates only |
| Best for | Personal assistant | Customer-facing bots |

## Prerequisites

- **OpenClaw** >= 2026.2.x installed and configured (`openclaw configure`)
- **Node.js** >= 22
- A **Meta Business App** with WhatsApp product enabled
- A domain with **HTTPS** for the webhook (ngrok for dev, reverse proxy for prod)

## Quick start (development)

### 1. Install the plugin

```bash
git clone https://github.com/baiadigitale/openclaw-channel-whatsapp-cloud.git
cd openclaw-channel-whatsapp-cloud
npm install && npm run build
openclaw plugins install -l .
```

> **Note:** Due to a known OpenClaw bug with symlinks, if the plugin isn't discovered after `install -l .`, add this to `~/.openclaw/openclaw.json`:
> ```json
> {
>   "plugins": {
>     "load": {
>       "paths": ["/absolute/path/to/openclaw-channel-whatsapp-cloud"]
>     }
>   }
> }
> ```

### 2. Get Meta credentials

1. Go to [developers.facebook.com](https://developers.facebook.com/) and create an app (type: **Business**)
2. Add the **WhatsApp** product
3. In **WhatsApp > API Setup**, note your **Phone Number ID**
4. Create a permanent access token:
   - **Business Settings > System Users** > create one with **Admin** role
   - Generate a token with permissions: `whatsapp_business_messaging`, `whatsapp_business_management`
5. Note your **App Secret** from **App Settings > Basic** (for webhook signature verification)

### 3. Run the setup wizard

```bash
openclaw whatsapp-cloud setup
```

This will prompt for all credentials and save them to `~/.openclaw/openclaw.json`.

Alternatively, set them manually:

```bash
openclaw config set channels.whatsapp-cloud.phoneNumberId "YOUR_PHONE_NUMBER_ID"
openclaw config set channels.whatsapp-cloud.accessToken "YOUR_ACCESS_TOKEN"
openclaw config set channels.whatsapp-cloud.appSecret "YOUR_APP_SECRET"
openclaw config set channels.whatsapp-cloud.verifyToken "a-random-string-you-choose"
```

### 4. Expose the webhook (dev)

```bash
ngrok http 3100
```

Copy the `https://xxxx.ngrok-free.app` URL.

### 5. Register the webhook on Meta

1. Go to **WhatsApp > Configuration** in your Meta app
2. Click **Edit** on the Webhook section
3. **Callback URL**: `https://your-ngrok-url/webhook/whatsapp-cloud`
4. **Verify Token**: the string you chose in step 3
5. Click **Verify and Save**
6. Subscribe to the **messages** webhook field

### 6. Start the gateway

```bash
openclaw gateway restart
```

Send a WhatsApp message to your business number — the bot will respond.

---

## Production deployment

### Architecture

```
User on WhatsApp
      |
      v
Meta Cloud API (graph.facebook.com)
      |
      v  HTTPS POST (webhook)
+--------------------------------------------------+
|  n8n Cloud (optional relay)                      |
|    - receives Meta webhooks                      |
|    - forwards to VPS with bearer token auth      |
|    - retries on failure, Telegram alerting       |
+--------------------------------------------------+
      |
      v  HTTPS POST (relay or direct)
+--------------------------------------------------+
|  Your server (VPS / Docker / Cloud)              |
|                                                  |
|  nginx/Caddy (TLS termination, port 443)         |
|      |                                           |
|      v  proxy_pass :3100                         |
|  OpenClaw Gateway (systemd service)              |
|    +-- whatsapp-cloud plugin                     |
|    |     webhook.ts  -> receives messages         |
|    |     crypto.ts   -> verifies HMAC signature   |
|    |     index.ts    -> dispatches to agent        |
|    |     api.ts      -> sends replies              |
|    +-- agent (Claude / GPT / ...)                |
+--------------------------------------------------+
```

The plugin supports two inbound paths:
- **Direct**: Meta sends webhooks straight to your server (verified via HMAC-SHA256)
- **n8n relay**: Meta sends webhooks to n8n Cloud, which forwards them to `/webhook/whatsapp-relay` with bearer token auth — adds retry, logging, and alerting

### Recommended repo structure

Create a **deployment repository** separate from this plugin:

```
my-openclaw-bot/
  openclaw.json          # OpenClaw config (env var refs for secrets)
  .env.example           # Documents all required env vars
  .env                   # Actual secrets (NEVER commit this)
  .gitignore
  workspace/
    AGENTS.md            # Agent instructions, persona, behavior rules
    SOUL.md              # Personality, tone, boundaries
    IDENTITY.md          # Agent name, emoji
    USER.md              # Info about the user/company
    TOOLS.md             # Tool-specific notes
  scripts/
    deploy.sh            # Deployment automation
    backup.sh            # State backup
  docker-compose.yml     # Optional: containerized deployment
  Caddyfile              # Or nginx.conf — reverse proxy config
```

**`.gitignore`:**
```
.env
*.bak
sessions/
credentials/
```

**`openclaw.json`** (with env var references):
```json5
{
  "gateway": {
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "${OPENCLAW_GATEWAY_TOKEN}"
    }
  },

  // LLM provider
  "auth": {
    "profiles": {
      "anthropic:default": {
        "provider": "anthropic",
        "mode": "token"
      }
    }
  },
  "agents": {
    "defaults": {
      "workspace": "./workspace",
      "models": {
        "anthropic/claude-sonnet-4-5": {}
      }
    }
  },

  // WhatsApp Cloud channel
  "channels": {
    "whatsapp-cloud": {
      "phoneNumberId": "${WHATSAPP_PHONE_NUMBER_ID}",
      "accessToken": "${WHATSAPP_ACCESS_TOKEN}",
      "appSecret": "${WHATSAPP_APP_SECRET}",
      "verifyToken": "${WHATSAPP_VERIFY_TOKEN}",
      "webhookPort": 3100,
      "dmPolicy": "open",
      "sendReadReceipts": true
    }
  },

  // Plugin
  "plugins": {
    "entries": {
      "whatsapp-cloud": { "enabled": true }
    }
  }
}
```

**`.env.example`:**
```bash
# Anthropic API key (get from https://console.anthropic.com/settings/keys)
ANTHROPIC_API_KEY=sk-ant-...

# OpenClaw gateway auth token (generate: openssl rand -hex 24)
OPENCLAW_GATEWAY_TOKEN=

# WhatsApp Cloud API (from https://developers.facebook.com/)
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_APP_SECRET=
WHATSAPP_VERIFY_TOKEN=
```

### Install on a production server

```bash
# 1. Install OpenClaw
npm install -g openclaw

# 2. Clone the plugin (private repo — no npm publish needed)
git clone git@github.com:baiadigitale/openclaw-channel-whatsapp-cloud.git ~/extensions/whatsapp-cloud
cd ~/extensions/whatsapp-cloud && npm install && npm run build

# 3. Clone your deployment repo
git clone https://github.com/yourorg/my-openclaw-bot.git ~/openclaw-bot
cd ~/openclaw-bot

# 4. Copy env file and fill in secrets
cp .env.example .env
nano .env

# 5. Point OpenClaw to your config
export OPENCLAW_CONFIG_PATH=~/openclaw-bot/openclaw.json
export OPENCLAW_STATE_DIR=~/openclaw-bot/.state

# 6. Install and start the gateway
openclaw gateway install
systemctl --user start openclaw-gateway.service
systemctl --user enable openclaw-gateway.service
```

Make sure your `openclaw.json` loads the plugin from the cloned path:

```json
{
  "plugins": {
    "load": {
      "paths": ["~/extensions/whatsapp-cloud"]
    },
    "entries": {
      "whatsapp-cloud": { "enabled": true }
    }
  }
}
```

### Update the plugin

After pushing changes to the repo, run this on the server:

```bash
cd ~/extensions/whatsapp-cloud && git pull && npm install && npm run build && systemctl --user restart openclaw-gateway.service
```

### Reverse proxy (Caddy)

**`Caddyfile`:**
```
yourdomain.com {
    reverse_proxy /webhook/whatsapp-cloud localhost:3100
    reverse_proxy /webhook/whatsapp-relay localhost:3100
}
```

```bash
sudo caddy start --config Caddyfile
```

Caddy handles TLS automatically via Let's Encrypt.

**nginx alternative:**
```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location /webhook/whatsapp-cloud {
        proxy_pass http://127.0.0.1:3100;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /webhook/whatsapp-relay {
        proxy_pass http://127.0.0.1:3100;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Docker deployment (optional)

**`docker-compose.yml`:**
```yaml
services:
  openclaw:
    image: node:22-slim
    working_dir: /app
    command: ["npx", "openclaw", "gateway", "--bind", "lan", "--port", "3100"]
    env_file: .env
    environment:
      OPENCLAW_CONFIG_PATH: /app/openclaw.json
      OPENCLAW_STATE_DIR: /data
      NODE_ENV: production
    volumes:
      - ./openclaw.json:/app/openclaw.json:ro
      - ./workspace:/app/workspace:ro
      - openclaw-data:/data
    ports:
      - "3100:3100"
    restart: unless-stopped

volumes:
  openclaw-data:
```

### Monitoring

```bash
# Live logs
journalctl --user -u openclaw-gateway.service -f

# Channel status
openclaw whatsapp-cloud status

# Gateway health
openclaw gateway status

# Send a test message
openclaw whatsapp-cloud test +39XXXXXXXXXX

# Audit log (structured JSONL of all webhook events)
tail -f ~/.openclaw/whatsapp-cloud-audit.jsonl | jq .
```

### Security checklist

- [ ] `appSecret` is set (enables webhook HMAC signature verification)
- [ ] Access token is a **System User token** (permanent, not a temporary test token)
- [ ] `dmPolicy` is set to `"allowlist"` if the bot should only serve specific numbers
- [ ] Webhook endpoint is HTTPS-only
- [ ] `.env` file has `chmod 600` and is not committed to git
- [ ] Gateway auth token is set (`gateway.auth.mode: "token"`)
- [ ] Gateway binds to loopback only (reverse proxy handles external traffic)
- [ ] n8n relay token (if using relay) stored in `~/.secrets/n8n/whatsapp-relay-token` with `chmod 600`

---

## Configuration reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | boolean | `true` | Enable/disable the channel |
| `phoneNumberId` | string | *required* | WhatsApp Phone Number ID |
| `businessAccountId` | string | — | WhatsApp Business Account ID |
| `accessToken` | string | *required* | Meta API access token (system user token) |
| `appSecret` | string | — | Meta App Secret for webhook signature verification |
| `verifyToken` | string | `"openclaw-wa-cloud-verify"` | Custom token for webhook endpoint verification |
| `webhookPort` | number | `3100` | HTTP server port for webhooks |
| `webhookPath` | string | `"/webhook/whatsapp-cloud"` | URL path for the webhook endpoint |
| `relayPath` | string | `"/webhook/whatsapp-relay"` | URL path for the n8n relay endpoint |
| `apiVersion` | string | `"v21.0"` | Meta Graph API version |
| `dmPolicy` | string | `"open"` | `"open"` (anyone) or `"allowlist"` (restricted) |
| `allowFrom` | string[] | `[]` | E.164 numbers allowed when dmPolicy=allowlist |
| `sendReadReceipts` | boolean | `true` | Auto-mark incoming messages as read |

## Features

### Inbound message types

- Text messages
- Images (with/without captions)
- Audio, video, documents, stickers
- Location sharing
- Contact cards
- Interactive replies (button and list selections)
- Quoted messages (reply context)

### Outbound capabilities

- Text messages (auto-split at 4096 chars)
- Interactive buttons (up to 3 quick reply buttons)
- Interactive lists (section-based menus)
- Media messages (image, audio, video, document)
- Template messages (for messages outside the 24h window)
- Read receipts

### Agent tools

The plugin registers two agent tools available to any bound agent:

- **`whatsapp_send`** — Send a text message directly via the API. Works from any session context (Telegram, WhatsApp, cron) without cross-context errors or session fragmentation. Handles 24h window expiry automatically (falls back to template).
- **`whatsapp_send_template`** — Send a specific pre-approved template message. Use for first-ever outbound contact or when a particular template is required.

### n8n relay

An optional relay path (`/webhook/whatsapp-relay`) accepts webhooks forwarded by n8n Cloud with bearer token authentication. This adds:

- **Retry logic**: n8n retries delivery if the VPS is temporarily unreachable
- **Execution logging**: full webhook history in n8n's execution log
- **Alerting**: configurable Telegram/email alerts on forwarding failure

To configure: store the relay bearer token at `~/.secrets/n8n/whatsapp-relay-token`. The token is loaded on gateway startup. If no token file exists, the relay path returns 503.

### Audit logging

All webhook events (inbound messages, signature failures, relay auth failures, parse errors, blocked messages) are written as structured JSONL to `~/.openclaw/whatsapp-cloud-audit.jsonl`. Non-critical — logging failures never block message processing.

### Auto session cleanup

On every gateway startup, the plugin scans the agent's `sessions.json` and removes any `whatsapp-cloud:group:` session artifacts and stale Baileys sessions. These artifacts are created when OpenClaw's built-in `message send` delivery system is used instead of the `whatsapp_send` tool, and would otherwise fragment conversation history across duplicate sessions.

### Security

- HMAC-SHA256 webhook signature verification (via App Secret)
- Timing-safe comparison to prevent timing attacks
- Bearer token authentication for n8n relay path
- DM policy (open / allowlist)
- Phone number normalization for allowlist matching

## Multi-agent routing

This plugin uses OpenClaw's `resolveAgentRoute` API to honor `bindings` in your config. If you run multiple agents, bind the WhatsApp Cloud channel to a specific agent:

```json5
{
  bindings: [
    { agentId: "monica", match: { channel: "whatsapp-cloud" } },
  ],
}
```

All inbound WhatsApp messages will be routed to the matched agent with a properly scoped session key (`agent:<agentId>:whatsapp-cloud:direct:<phone>`). See [Multi-Agent Routing](https://docs.openclaw.ai/concepts/multi-agent) for details.

## The 24-hour messaging window

WhatsApp Cloud API enforces a **24-hour customer service window**:

- When a customer messages you, you have **24 hours** to respond with free-form text
- After the window closes, you can only send **pre-approved template messages**
- Each template must be submitted to Meta for review
- A valid **payment method** must be linked to your WhatsApp Business Account for business-initiated (template) messages to deliver

### Automatic window detection

The plugin tracks the last inbound message timestamp per phone number (persisted to `~/.openclaw/whatsapp-cloud-windows.json`). When an outbound free-form message is attempted:

- **Window open** (< 23h since last inbound) → sends as normal free-form text
- **Window expired** (≥ 23h) → automatically falls back to a configurable template

The fallback template defaults to `monica_followup`. This means your agent doesn't need to manually decide between free-form and template messages — the plugin handles it transparently.

### Sending messages (agent tools)

**`whatsapp_send`** — the recommended way to send all outbound WhatsApp messages:

```
whatsapp_send({ to: "38631651745", text: "Your message here" })
```

Works from any session context. If the 24h window is open, sends free-form text. If expired, automatically falls back to a template. No session artifacts are created.

**`whatsapp_send_template`** — for first-ever outbound contact or when a specific template is required:

```
whatsapp_send_template({
  to: "38651313135",
  template: "your_template_name",
  language: "en",
  components: [
    {
      type: "body",
      parameters: [
        { type: "text", text: "Name" },
        { type: "text", text: "Your message here" }
      ]
    }
  ]
})
```

Templates must be pre-approved in the [Meta WhatsApp Business Manager](https://business.facebook.com/). Both tools are available to all agents that have the WhatsApp Cloud channel bound to them.

## Development

```bash
git clone https://github.com/baiadigitale/openclaw-channel-whatsapp-cloud.git
cd openclaw-channel-whatsapp-cloud
npm install

npm run type-check    # TypeScript strict mode
npm test              # 32 tests
npm run dev           # Watch mode (auto-rebuild)

# Link to OpenClaw for development
openclaw plugins install -l .
```

### Project structure

```
src/
  index.ts        — Plugin entry point, channel definition, agent tools, auto-cleanup
  types.ts        — TypeScript interfaces
  api.ts          — Meta Cloud API client (outbound)
  webhook.ts      — HTTP server (inbound: direct + n8n relay, audit logging)
  crypto.ts       — HMAC-SHA256 signature verification
  setup.ts        — Interactive setup wizard
  runtime.ts      — OpenClaw runtime accessor
  onboarding.ts   — Onboarding adapter
  __tests__/      — Vitest test suites
```

## Rate limits

New WhatsApp Business accounts start at **250 unique recipients per 24 hours**. As quality improves:

250 > 1,000 > 10,000 > 100,000 > unlimited

## Troubleshooting

**Webhook verification fails:**
- Ensure `verifyToken` in OpenClaw config matches what you entered in Meta dashboard
- The webhook URL must be reachable over HTTPS

**Messages not arriving:**
- Check that you subscribed to the `messages` webhook field in Meta dashboard
- Check logs: `journalctl --user -u openclaw-gateway.service -f`
- Verify `appSecret` is correct (wrong secret = messages silently dropped)

**Template messages accepted by API but never delivered:**
- Verify a **payment method** is linked to the correct WhatsApp Business Account in Meta Business Manager. Without one, the API returns `accepted` but silently drops business-initiated messages.
- Check that the template status is `APPROVED` (not just `PENDING`)
- Newly approved templates may take up to a few hours to fully propagate

**"phoneNumberId?.trim is not a function":**
- The `phoneNumberId` was saved as a number instead of a string. Fix it in `~/.openclaw/openclaw.json` by wrapping the value in quotes: `"phoneNumberId": "878388375365101"`

**Plugin not found after `install -l .`:**
- OpenClaw has a symlink discovery bug. Add `plugins.load.paths` to your config pointing to the plugin directory (see install instructions above)

**n8n relay returns 503:**
- No relay token file found at `~/.secrets/n8n/whatsapp-relay-token`. Create it with a random token and restart the gateway.

**n8n relay returns 401:**
- The bearer token in the n8n HTTP Request node doesn't match the token in `~/.secrets/n8n/whatsapp-relay-token`.

**Duplicate "group" sessions appearing:**
- Someone used `message send --channel whatsapp-cloud` or `message action=send channel=whatsapp-cloud` instead of the `whatsapp_send` tool. These artifacts are auto-cleaned on next gateway restart. Update agent instructions to use `whatsapp_send` exclusively.

**Gateway won't start:**
- Set `gateway.mode`: `openclaw config set gateway.mode local`
- Check logs: `journalctl --user -u openclaw-gateway.service -n 50`

## License

MIT — [Baia Digitale SRL](https://baiadigitale.com)
