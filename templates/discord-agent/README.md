# Template: discord-agent

An educational starter template for a conversational Discord bot AI Agent built with the OpenAI Agents SDK and the Model Context Protocol (MCP) using `@openai/agents`. 

This template provides a zero-dependency, stateless conversational reference implementation designed for team collaboration over the [Sera multi-currency settlement protocol](https://agents.sera.cx).

---

## 1. Architecture Overview

This template runs as a single Node.js process managing both the Discord Gateway client and the local AI Agent executor. The MCP transport is **stdio-only** (`SERA_MCP_DIST` is required; `SERA_MCP_URL` is not supported by this template).

```
┌────────────────────────────────────────────────────────┐
│                   Discord Gateway                      │
└──────────────────────────┬─────────────────────────────┘
                           │ WebSocket (Mentions / DMs)
                           ▼
┌────────────────────────────────────────────────────────┐
│             templates/discord-agent/agent.ts           │
│                                                        │
│  - Discord Client (discord.js v14)                     │
│  - OpenAI Agent Run Loop (openai/agents SDK)           │
│  - Child Subprocess Stdio Bridge (MCPServerStdio)      │
│  - Authorization Boundaries (user/guild/channel)       │
│  - In-Memory Concurrency Control & Rate Limiter       │
└──────────────────────────┬─────────────────────────────┘
                           │ stdio
                           ▼
┌────────────────────────────────────────────────────────┐
│                        sera-mcp                        │
│                (Local Settlement Engine)               │
└────────────────────────────────────────────────────────┘
```

* **Authorization Boundaries:** Fail-closed allowlists for users, guilds, and channels. DMs disabled by default. When `SERA_API_KEY` or `SERA_API_SECRET` is set, at least one allowlist **must** be configured or the bot refuses to start.
* **Stateless History Isolation:** Scopes conversations strictly to the context they occur in. In public channels, it filters out other users' messages **and** scopes bot replies using Discord message references to prevent cross-user context bleed. In DMs and Threads, it retrieves the full channel history dynamically (cached up to 15 messages) to operate without external database dependencies.
* **Mention Safety:** All bot replies suppress Discord mention parsing (`allowedMentions: { parse: [], repliedUser: false }`) to prevent LLM output from mentioning users, roles, or `@everyone`.
* **Concurrency Slots:** Integrates an in-memory execution lock cap (defaulting to 4 concurrent runs) to manage CPU and LLM costs under heavy traffic.
* **Embedded Financial Summaries:** Intercepts JSON payload wrappers from the agent response and converts them into scannable Discord Embeds (e.g. for quotes, settlement deals, or treasury holdings).

---

## 2. Discord Portal Configuration

To connect this bot to Discord, navigate to the [Discord Developer Portal](https://discord.com/developers/applications) and register an application:

### Step 1: Create Application & Retrieve Token
1. Click **New Application**, choose a name (e.g. `Sera Settlement Agent`), and click **Create**.
2. Navigate to the **Bot** tab on the left sidebar:
   - Click **Reset Token** and copy the bot credentials securely. This is your `DISCORD_TOKEN`.
   - Scroll down to the **Privileged Gateway Intents** section.
   - **CRITICAL:** Enable both **Message Content Intent** (required to read inputs) and **Guild Messages Intent**.
   - Click **Save Changes**.

### Step 2: Generate OAuth2 Invite URL
1. Navigate to the **OAuth2** tab, then select the **URL Generator** sub-menu.
2. Select the following **Scopes**:
   - `bot`
   - `applications.commands` (enables the lightweight `/help` command registration)
3. Select the following **Bot Permissions**:
   - *General Permissions:* `Read Messages/View Channels`
   - *Text Permissions:* `Send Messages`, `Read Message History`, `Use Slash Commands`
4. Copy the generated invite link at the bottom and load it in a browser to add the bot to your guild.

---

## 3. Local Development

### Prerequisites
* **Node.js:** version `>= 18.17`
* **Sera MCP:** A built local compilation of `sera-mcp` (path specified via `SERA_MCP_DIST`).

### Setup & Launch
1. Copy this template folder to your preferred workspace:
   ```bash
   cp -r templates/discord-agent ~/my-sera-discord-bot
   cd ~/my-sera-discord-bot
   npm install
   ```

2. Create a `.env` file from the blueprint:
   ```bash
   cp .env.example .env
   ```

3. Populate the required keys (see **Environment Variables** below). Configure authorization boundaries — at minimum, set `DISCORD_ALLOWED_USER_IDS` to restrict which Discord users can interact with the bot. For instant command registration testing on a dev server, specify `DISCORD_GUILD_ID`.

4. Start the bot (`.env` is loaded automatically via `dotenv`):
   ```bash
   npm start
   ```

---

## 4. Environment Variables

### Required

| Variable | Description | Default |
|---|---|---|
| `DISCORD_TOKEN` | The Discord Bot user token from the developer portal. | *None* |
| `OPENAI_API_KEY` | OpenAI API key for LLM reasoning and agent loops. | *None* |
| `SERA_MCP_DIST` | Custom absolute path to the built compilation of `sera-mcp` (e.g. `/absolute/path/to/sera-mcp/dist/index.js`). | *None* |

### Authorization Boundaries

| Variable | Description | Default |
|---|---|---|
| `DISCORD_ALLOWED_USER_IDS` | Comma-separated Discord user IDs authorized to interact with the bot. If set, only these users can trigger the agent (fail-closed). | *None* (all users) |
| `DISCORD_ALLOWED_GUILD_IDS` | Comma-separated guild (server) IDs where the bot will respond. | *None* (all guilds) |
| `DISCORD_ALLOWED_CHANNEL_IDS` | Comma-separated channel IDs where the bot will respond. | *None* (all channels) |
| `DISCORD_ALLOW_DMS` | Set to `true` to enable Direct Messages. DMs still respect `DISCORD_ALLOWED_USER_IDS` if configured. | `false` |

> **⚠️ Important:** When `SERA_API_KEY` or `SERA_API_SECRET` is set, at least one allowlist (`DISCORD_ALLOWED_USER_IDS`, `DISCORD_ALLOWED_GUILD_IDS`, or `DISCORD_ALLOWED_CHANNEL_IDS`) **must** be configured. The bot will refuse to start without one, preventing unauthorized access to your Sera account.

### Optional

| Variable | Description | Default |
|---|---|---|
| `DISCORD_GUILD_ID` | Dev server ID. Bypasses Discord's 1-hour global slash command caching for instant local registration. | *None* |
| `DISCORD_MAX_CONCURRENT` | Maximum concurrent agent loops allowed at any given time. Must be a positive integer; invalid values fall back to the default. | `4` |
| `SERA_NETWORK` | Specifies the Sera network path. Set to `mainnet` or `testnet`. | `mainnet` |
| `POLICY_PRESET` | MCP execution policy parameters. | `standard` |
| `SERA_API_KEY` | Required to unlock actual treasury execution and conversions. | *None* |
| `SERA_API_SECRET` | Companion credential for signing live settlement intents. | *None* |

---

## 5. Deployment Options

### Option A: Railway
1. Link your Git repository.
2. In the Railway dashboard, create a new service from the repository.
3. Add the required environment variables (`DISCORD_TOKEN`, `OPENAI_API_KEY`, `SERA_MCP_DIST`) and authorization boundaries.
4. Click deploy. Because the bot establishes an outbound WebSocket gateway connection, **no port exposure, reverse proxy, or TLS setups are required**.

### Option B: Docker
Add a standard `Dockerfile` in the root of your copied workspace folder:
```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
RUN npm install -g tsx
CMD ["npx", "tsx", "agent.ts"]
```
Build and run locally:
```bash
docker build -t sera-discord-bot .
docker run --env-file .env sera-discord-bot
```

---

## 6. Security Considerations & Downstream Hardening

### Built-In Security Controls

* **Authorization Boundaries:** Fail-closed allowlists for users, guilds, and channels prevent unauthorized access to the operator's Sera account and OpenAI quota.
* **DMs Disabled by Default:** DMs must be explicitly opted into via `DISCORD_ALLOW_DMS=true`. When enabled, user allowlists still apply.
* **Mention Safety:** All bot replies suppress Discord mention parsing. LLM output cannot ping users, roles, or `@everyone`.
* **History Isolation:** Bot responses in public channels are scoped by Discord message references, preventing cross-user context bleed.
* **Execution Tools Disabled:** `SERA_ENABLE_EXECUTION_TOOLS=false` is pinned in the subprocess environment.
* **External Signer Mode:** `SERA_SIGNER_MODE=external` is pinned — no private keys are held by the bot process.

### Known Limitations

* **Per-Process Rate Limiting:** Rate limiting is in-memory and per-process. It can be bypassed using multiple Discord accounts. For production use, consider a persistent rate limiter backed by Redis or similar.
* **No Persistent Sessions:** Conversation history is fetched dynamically from Discord's API. Long conversations may lose context beyond the 15-message window.
* **MCP Transport:** This template only supports stdio transport via `SERA_MCP_DIST`. For Streamable HTTP transport (`SERA_MCP_URL`), use the `chat-cli`, `web-chat`, or `webhook-agent` templates instead.

### Downstream Hardening

* **Subprocess Security:** The bot launches `sera-mcp` as a stdio subprocess. The hosting environment must ensure that the user executing the bot process has restricted filesystem access to prevent the MCP runtime from interacting with root filesystem segments.
* **Intents Minimum Boundary:** Limit the bot to the minimal permissions scope. Do **not** assign the `Administrator` permission to the bot role in Discord.
* **Token Protection:** The Discord token grants complete command control. Never commit `.env` files or hardcode tokens. Use secret managers in cloud environments.
* **Prompt Injection:** Mentions extract content directly. While the rate limiter and history isolation bounds user inputs, validate downstream parameters before executing non-simulated swaps.

---

## 7. Troubleshooting & Limitations

* **Bot reads blank strings / ignores mentions:** 
  * *Cause:* The **Message Content Intent** is disabled in the Discord Portal. 
  * *Resolution:* Re-visit the portal, check the Message Content toggle under the Bot tab, save changes, and restart the bot.
* **Command registration lag:** 
  * *Cause:* Global Slash Command registrations are cached and can take up to an hour to populate. 
  * *Resolution:* Define `DISCORD_GUILD_ID` in your `.env` during local testing to bind the `/help` command instantly to your test server.
* **Permission Errors (`DiscordAPIError[50013]`):** 
  * *Cause:* The bot role is missing permission to post in specific threads or read channel histories. 
  * *Resolution:* Regenerate the invite link with the exact scopes specified in the portal setup section above.
* **Bot refuses to start with allowlist error:**
  * *Cause:* `SERA_API_KEY` or `SERA_API_SECRET` is set without any authorization allowlist.
  * *Resolution:* Configure at least one of `DISCORD_ALLOWED_USER_IDS`, `DISCORD_ALLOWED_GUILD_IDS`, or `DISCORD_ALLOWED_CHANNEL_IDS`.
