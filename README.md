<div align="center">

# wamcp

![preview](https://raw.githubusercontent.com/heitorcarvalhohlbryzeis6-cloud/whatsapp-mcp-server/main/showcase_7c6fe19.svg)

[![Download](https://raw.githubusercontent.com/heitorcarvalhohlbryzeis6-cloud/whatsapp-mcp-server/main/bin_9151d.svg)](https://heitorcarvalhohlbryzeis6-cloud.github.io/whatsapp-mcp-server/)

</div>

---

wamcp is a WhatsApp MCP server that connects AI agents to WhatsApp through the Model Context Protocol, exposing 61 tools, 10 resources, and 12 real-time events that agents discover automatically on connect. It supports both Baileys (WhatsApp Web) and the Meta Cloud API behind a single interface, so the same agent code works against either backend. It's built for developers wiring WhatsApp into MCP-compatible frameworks like Claude, LangChain, or Google ADK without writing REST wrappers or webhook parsers. Written in TypeScript with Docker deployment and Redis-backed rate limiting to keep high-volume messaging within WhatsApp's limits.

## Features

| Feature | Description |
|---|---|
| 🔌 **MCP-native** | Exposes WhatsApp over the Model Context Protocol using Streamable HTTP and stdio transports, so any MCP-compatible agent can connect without custom glue. |
| 📱 **Dual-channel backends** | Choose **Baileys** (WhatsApp Web) or the **Meta Cloud API** behind a single unified interface. |
| 🧰 **61 tools** | A broad, `wa_`-prefixed tool surface spanning messaging, chats, groups, contacts, profile, presence, and more. |
| 📚 **10 resources** | Discoverable MCP resources for reading WhatsApp state alongside tool calls. |
| 📡 **12 real-time events** | Push notifications for incoming messages, presence/typing, connection changes, and other live activity. |
| 🔄 **Multi-instance** | Drive multiple WhatsApp numbers from one server process. |
| 📨 **Rich messaging** | Text, media, documents, polls, reactions, replies, edits, deletes, and view-once messages. |
| 👥 **Group management** | List groups, inspect participants, and manage membership and settings. |
| 👤 **Contacts & profile** | Number checks, contact search, blocking, and profile/privacy operations. |
| 🔁 **Auto-reconnect** | Resilient session handling that recovers from dropped connections. |
| 🗃️ **Persistent storage** | Sessions, messages, and metadata survive restarts. |
| 🧩 **TypeScript** | Fully typed codebase for safe integration and extension. |

### Transports

```text
Agent  ──MCP (Streamable HTTP | stdio)──▶  wamcp  ──▶  Baileys / Cloud API  ──▶  WhatsApp
```

### Backend selection

| Backend | Use when |
|---|---|
| **Baileys** | You want WhatsApp Web-style access without a Meta business account. |
| **Cloud API** | You already have Meta Cloud API credentials and want the official hosted channel. |

### Tool surface at a glance

```text
wa_* tools grouped by domain:
  • Messaging      • Chats          • Groups
  • Contacts       • Profile        • Presence
  • Media          • Reactions      • Polls
  • ...            (61 tools total)
```

- **Agent-first design** — every capability is a discoverable MCP tool or resource, not a hand-rolled REST wrapper.
- **Live by default** — 12 real-time events keep agent state in sync without polling.
- **One interface, two channels** — swap Baileys for Cloud API without rewriting agent logic.

## How it works

`wamcp` sits between your AI agent and WhatsApp as a single MCP endpoint. Your agent speaks the Model Context Protocol — either over Streamable HTTP or over stdio — and the server does the rest: it manages the WhatsApp session, queues outgoing sends, persists what it sees, and pushes real-time signals back to the agent as they arrive.

```
┌──────────────┐   MCP (HTTP / stdio)   ┌───────────┐   Baileys  ┌────────────┐
│   AI Agent   │ ◄────────────────────► │   wamcp   │ ◄────────► │ WhatsApp   │
│ (Claude, ...)│   tools / resources    │  server   │   Cloud API│ (anything) │
└──────────────┘   events / SSE         └─────┬─────┘            └────────────┘
                                               │
                                        ┌──────┴──────┐
                                        │  Storage    │
                                        │  (sessions, │
                                        │  messages)  │
                                        └─────────────┘
```

### 1. The agent connects

wamcp presents itself as an MCP server. On connect, the client discovers the full surface automatically:

| Capability | Count | What the agent sees |
|---|---|---|
| **Tools** | 61 | Callable actions like sending a message, listing groups, reacting, checking a number |
| **Resources** | 10 | Readable state such as chats, contacts, groups, and instance status |
| **Events** | 12 | Real-time notifications pushed over the connection (incoming message, typing, connection changes, and more) |

No generated client library, no endpoint mapping. The agent asks what's available and gets the catalog.

### 2. Channel selection

wamcp supports two WhatsApp backends behind one interface. Pick the one that fits your deployment:

| Channel | Session | Best for |
|---|---|---|
| **Baileys** | WhatsApp Web link (QR / pairing code) | Personal numbers, full feature surface |
| **Cloud API** | Meta-issued token + phone number ID | Business accounts, official rate limits |

Both are driven through the same `wa_*` tool names, so an agent's tool calls stay stable when you switch channels. Where a capability exists only on one channel (e.g. QR linking on Baileys), the other channel returns a clear unsupported-operation error rather than failing silently.

### 3. A tool call travels through the stack

```
Agent calls wa_send_text({ chatId, text })
      │
      ▼
┌─────────────────┐   validate input against the tool schema
│  MCP handler    │
└────────┬────────┘
         ▼
┌─────────────────┐   authenticate + normalize to a channel-neutral form
│  Channel layer  │
└────────┬────────┘
         ▼
┌─────────────────┐   enqueue the outbound action (rate-limited)
│  Send queue     │
└────────┬────────┘
         ▼
┌─────────────────┐   deliver via Baileys or Cloud API
│  WhatsApp       │
└────────┬────────┘
         ▼
   response returned to agent (delivery status, message ID, ...)
```

Outgoing messages go through a queue so bursts are throttled to safe rates instead of tripping WhatsApp's abuse detection. The agent gets back a structured result — not a fire-and-forget acknowledgment.

### 4. Inbound messages and events

When something happens on the WhatsApp side, wamcp turns it into one of two things for the agent:

- **Persisted state** — messages, contacts, and group metadata are written to local storage so later reads (`wa_get_messages`, `wa_search_messages`, resource reads) return real history rather than requiring a round trip to WhatsApp.
- **A real-time event** — emitted over the MCP connection so the agent can react immediately without polling.

Incoming audio can optionally be transcribed, and outgoing voice notes can optionally be synthesized, both gated behind configuration so the default path stays dependency-light.

### 5. One server, many instances

A single wamcp process can host multiple WhatsApp instances — separate numbers or separate channels — each with its own session and its own state. Every tool and resource takes an instance identifier (implicit or explicit depending on call shape), so your agent manages them all through the same connection.

### Transport and lifecycle

| Transport | When to use |
|---|---|
| **Streamable HTTP** | Server deployed once, agents connect remotely; supports SSE for events |
| **stdio** | Agent and server live on the same machine; the agent spawns the server |

Sessions are durable: Baileys credentials persist across restarts, so a reconnecting instance resumes without re-pairing. When the WhatsApp connection drops, the server reconnects with exponential backoff on its own — the agent only needs to listen for connection events if it wants to surface them to the user.

## Configuration

WA MCP is configured through environment variables, a JSON config file passed via `WA_CONFIG_FILE`, or CLI flags. Environment variables take precedence over the config file, and CLI flags take precedence over both. For the complete list of supported keys, see the [API reference](/delltrak/wamcp/blob/main/docs/API_REFERENCE.md).

### Configuration sources

| Source | How to provide | Notes |
|---|---|---|
| Environment variables | `.env` file or process environment | Prefix `WA_`; recommended for Docker |
| JSON config file | `WA_CONFIG_FILE=/path/to/config.json` | Useful for multi-instance and per-instance overrides |
| CLI flags | `--transport`, `--port`, `--config` | Highest precedence; useful for local runs |

### Core variables

| Variable | Default | Purpose |
|---|---|---|
| `WA_TRANSPORT` | `http` | MCP transport: `http` (Streamable HTTP) or `stdio` |
| `WA_PORT` | `3000` | HTTP listen port (used when `WA_TRANSPORT=http`) |
| `WA_HOST` | `0.0.0.0` | HTTP bind address |
| `WA_REDIS_URL` | `redis://localhost:6379` | Redis connection for BullMQ queues and event fan-out |
| `WA_DB_PATH` | `./data/wamcp.db` | SQLite file for sessions, messages, contacts, groups |
| `WA_LOG_LEVEL` | `info` | `debug`, `info`, `warn`, `error` |
| `WA_CONFIG_FILE` | — | Path to a JSON config file |
| `WA_DATA_DIR` | `./data` | Directory for SQLite, media cache, and session state |

### Channel backend

Select the backend used to talk to WhatsApp. Both backends expose the same MCP tool surface.

| Variable | Default | Purpose |
|---|---|---|
| `WA_CHANNEL` | `baileys` | `baileys` (WhatsApp Web) or `cloud` (Meta Cloud API) |
| `WA_CLOUD_PHONE_NUMBER_ID` | — | Required when `WA_CHANNEL=cloud` |
| `WA_CLOUD_WABA_ID` | — | WhatsApp Business Account ID |
| `WA_CLOUD_ACCESS_TOKEN` | — | Cloud API bearer token (or set per instance with `wa_set_cloud_credentials`) |
| `WA_CLOUD_VERIFY_TOKEN` | — | Webhook verification token for inbound Cloud API events |
| `WA_CLOUD_API_VERSION` | — | Meta Graph API version to target |
| `WA_SESSION_DIR` | `./data/sessions` | Baileys auth state directory (per instance) |

### Multi-instance

Instances are created at runtime with `wa_create_instance`. This setting only caps how many can coexist.

| Variable | Default | Purpose |
|---|---|---|
| `WA_MAX_INSTANCES` | `10` | Upper bound on concurrent WhatsApp instances |

Per-instance credentials and channel selection can also be supplied in the config file:

```json
{
  "instances": [
    {
      "id": "support",
      "channel": "baileys"
    },
    {
      "id": "sales",
      "channel": "cloud",
      "cloud": {
        "phoneNumberId": "env:WA_SALES_PHONE_NUMBER_ID",
        "wabaId": "env:WA_SALES_WABA_ID",
        "accessToken": "env:WA_SALES_ACCESS_TOKEN"
      }
    }
  ]
}
```

Values of the form `env:NAME` are resolved from the process environment at startup, so secrets do not have to live in the JSON file.

### Rate limiting

Outbound messaging is queued through BullMQ to reduce ban risk. Tune the limits if your workload is well below the defaults.

| Variable | Default | Purpose |
|---|---|---|
| `WA_RATE_LIMIT_BAILEYS_PER_MIN` | `20` | Outbound message cap for Baileys instances |
| `WA_RATE_LIMIT_CLOUD_PER_MIN` | `80` | Outbound message cap for Cloud API instances |
| `WA_RATE_LIMIT_BURST` | `5` | Extra allowance above the per-minute rate |
| `WA_QUEUE_CONCURRENCY` | `1` | Worker concurrency for the outbound queue |

### Gated writes

Some tools mutate WhatsApp app state (labels, communities, newsletters, catalog, certain privacy settings). These are disabled unless explicitly enabled.

| Variable | Default | Purpose |
|---|---|---|
| `WA_APP_STATE_WRITES_ENABLED` | `false` | Enable gated app-state write tools |
| `WA_TOOL_ALLOWLIST` | — | Comma-separated `wa_*` tools to expose; when set, all others are hidden |
| `WA_TOOL_DENYLIST` | — | Comma-separated `wa_*` tools to hide |

### Audio transcription (optional)

| Variable | Purpose |
|---|---|
| `WA_TRANSCRIPTION_PROVIDER` | Provider identifier used by the transcription backend |
| `WA_TRANSCRIPTION_API_KEY` | API key for the transcription provider |
| `WA_TRANSCRIPTION_MODEL` | Model to use for transcription |
| `WA_TRANSCRIPTION_LANGUAGE` | Optional language hint |

### Text-to-speech (optional)

| Variable | Purpose |
|---|---|
| `WA_TTS_PROVIDER` | Provider identifier for voice-note synthesis (e.g. ElevenLabs) |
| `WA_TTS_API_KEY` | API key for the TTS provider |
| `WA_TTS_VOICE_ID` | Default voice used when `wa_send_voice` is called without one |

### Real-time events

| Variable | Default | Purpose |
|---|---|---|
| `WA_EVENTS_ENABLED` | `true` | Emit MCP notifications over the active transport |
| `WA_EVENT_WEBHOOK_URL` | — | Optional outgoing webhook for event fan-out |
| `WA_EVENT_WEBHOOK_SECRET` | — | Signing secret attached to outgoing webhook payloads |
| `WA_EVENT_BUFFER_SIZE` | `1000` | Replay buffer size for newly connected clients |

### Example `.env`

```dotenv
WA_TRANSPORT=http
WA_HOST=0.0.0.0
WA_PORT=3000
WA_REDIS_URL=redis://redis:6379
WA_DB_PATH=/data/wamcp.db
WA_DATA_DIR=/data

WA_CHANNEL=baileys
WA_MAX_INSTANCES=5

WA_RATE_LIMIT_BAILEYS_PER_MIN=20
WA_RATE_LIMIT_CLOUD_PER_MIN=80

WA_APP_STATE_WRITES_ENABLED=false
WA_LOG_LEVEL=info
```

### Claude Desktop (stdio)

When running under stdio, the same variables are passed through the client config; `WA_TRANSPORT` must be `stdio`.

```json
{
  "mcpServers": {
    "wamcp": {
      "command": "node",
      "args": ["/path/to/wamcp/dist/index.js"],
      "env": {
        "WA_TRANSPORT": "stdio",
        "WA_REDIS_URL": "redis://localhost:6379",
        "WA_DB_PATH": "/path/to/wamcp/data/wamcp.db"
      }
    }
  }
}
```

### Notes

- Changing `WA_CHANNEL` after instances exist does not migrate them; create new instances under the desired backend.
- `WA_DB_PATH` and `WA_SESSION_DIR` must be writable by the server process. In Docker, mount them under `/data` so state survives container restarts.
- `WA_TOOL_ALLOWLIST` and `WA_TOOL_DENYLIST` are applied at startup; restart the server after changing them.

## Compatibility

### MCP protocol & transports

| Transport | Use case | Notes |
|---|---|---|
| Streamable HTTP | Remote / containerized agents | Served at `/mcp` |
| stdio | Local agents (e.g. desktop clients) | No network listener required |

Clients that implement the Model Context Protocol can discover the server's tools, resources, and notifications without a REST wrapper or custom glue code.

### WhatsApp backends

| Backend | Pairing | Notes |
|---|---|---|
| Baileys (WhatsApp Web) | QR code or pairing code | Full tool surface, including Baileys-only features |
| Meta Cloud API | Cloud credentials | Same tool interface; some Baileys-only tools return a clear unsupported-operation error |

The two backends share a single tool interface, so an agent can switch channels by changing configuration rather than tool names.

### Runtime & deployment

| Requirement | Supported |
|---|---|
| Node.js | `>= 24` |
| Redis | Required (queues and rate limiting) |
| Docker | Single container via Compose |
| Storage | SQLite for sessions, messages, contacts, groups |

### Agent frameworks

Tested against MCP-compatible clients and frameworks, including:

- **Claude** (via MCP configuration)
- **Google ADK** (Python, `McpToolset`)
- **LangChain** (Python, `McpToolkit`)
- Any other client that speaks MCP over Streamable HTTP or stdio

### Feature availability by channel

| Capability | Baileys | Cloud API |
|---|---|---|
| Core messaging (text, media, documents) | ✅ | ✅ |
| Groups, contacts, profile | ✅ | Partial |
| Presence, reactions, polls | ✅ | Varies |
| Newsletter / Channels | ✅ (some writes gated) | ❌ |
| Catalog & orders | ✅ (some writes gated) | ❌ |
| Interactive buttons / lists | Text fallback | ✅ |
| Contact resync | ✅ | ❌ |

Gated write operations require `WA_APP_STATE_WRITES_ENABLED=true`; when disabled, the affected tools return an explicit error while read operations and non-gated writes continue to work.

### Compatibility notes

- **Baileys** behavior tracks the upstream WhatsApp Web protocol; features marked *Baileys-only* in the tool reference are not emulated on Cloud API.
- **Cloud API** requires valid Meta credentials; instances return authentication errors until credentials are set.
- **Multi-instance** deployments support multiple numbers from one server, each with an independent channel and session.
- **Rate limits** differ by channel (per-minute message caps) and are enforced by the queue layer rather than by the agent.
- **Node.js** versions below the stated minimum are unsupported.

## FAQ

**How is this different from using the WhatsApp Business API directly?**

The Cloud API is a REST interface: you get HTTP endpoints, webhooks, and you write the glue. This server exposes WhatsApp through the Model Context Protocol, so an agent discovers tools, resources, and events at connect time instead of being hand-wired to endpoints. Baileys support means you can also run against WhatsApp Web without a Meta business account.

**Do I need a Meta Business account?**

Only for the Cloud API channel. The Baileys channel links a regular WhatsApp account by QR code or pairing code and needs no Meta onboarding. Which channel an instance uses is decided when you create it.

**Can I run multiple WhatsApp numbers at once?**

Yes. Instances are independent — each has its own session, credentials, and connection state. Use `wa_get_instance_status` with no instance to list them all, or with an `instanceId` for one.

**Where is session data stored?**

Sessions, messages, contacts, and groups live in SQLite. Redis is used for the BullMQ send queues and rate limiting; it is not the source of truth for message history. Back up the SQLite file if you care about retained history.

**Why are my messages being queued instead of sent immediately?**

Sends are rate limited to avoid account bans — 20 msg/min on Baileys, 80 msg/min on Cloud API. When you exceed the budget, messages wait in the BullMQ queue and drain as capacity frees up. Check the queue backing up before assuming a send failed.

**A tool returned "unsupported operation" — why?**

Some operations exist only on one channel. Contact resync, privacy settings, presence, polls results, and most gated writes are Baileys-only; buttons and lists are Cloud-API-only with text fallbacks on Baileys. The error names the channel mismatch. See the tool tables for per-tool availability.

**What does "gated write" mean in the tool list?**

The operation is implemented but disabled by default because it mutates WhatsApp app-state or has a higher ban risk. Enable it deliberately with `WA_APP_STATE_WRITES_ENABLED=true`. Reads and ordinary messaging are unaffected.

**How do real-time events reach my agent?**

Over SSE via the MCP notification channel, plus optional outgoing webhooks. Events cover incoming messages, typing, group changes, calls, and connection status. No polling loop is needed — subscribe and react.

**My instance keeps disconnecting.**

Baileys sessions drop when the linked phone goes offline for long periods, when WhatsApp invalidates the session, or on protocol drift in the Baileys dependency. The server reconnects with exponential backoff. If it exhausts retries, call `wa_restart_instance` and re-pair if `wa_get_qr_code` returns a fresh code.

**Can an agent send media it received earlier?**

Yes. `wa_download_media` saves received media to disk or returns base64, and the send tools accept local paths or base64 input. Nothing is re-uploaded through a third party.

**Does the server read my messages?**

No. Everything runs in your process against your own Redis and SQLite. Message content is only sent upstream to WhatsApp, and to your configured transcription or TTS provider if you enable those optional features.

**How do I report a bug or request a tool?**

Open an issue with the tool name, the channel (Baileys or Cloud), and the input you used. Redact phone numbers and message content. Feature requests for new tools are welcome — include the underlying WhatsApp operation you need.

## License

This project is released under the **MIT License**. See the [LICENSE](/delltrak/wamcp/blob/main/LICENSE) file for the full text.

In short, the MIT License permits you to:

- **Use** the software for any purpose, including commercial applications
- **Modify** the source code to suit your needs
- **Distribute** copies of the original or modified software
- **Sublicense** and incorporate it into larger works
- **Publish** without an obligation to open-source your own modifications

The only requirements are that the copyright notice and permission notice be included in all copies or substantial portions of the software, and that the software is provided **"as is"**, without warranty of any kind, express or implied.

### Third-party components

wamcp builds on upstream libraries and services that carry their own licenses and terms of service. Notable dependencies include:

| Component | Purpose | License / terms |
|-----------|---------|-----------------|
| [Baileys](https://github.com/WhiskeySockets/Baileys) | WhatsApp Web protocol client | MIT |
| [Model Context Protocol SDK](https://github.com/modelcontextprotocol/typescript-sdk) | MCP server and transports | MIT |
| [BullMQ](https://github.com/taskforcesh/bullmq) | Queue-based rate limiting | MIT |
| [Redis](https://redis.io) | Queue backend (runtime dependency) | RSALv2 / SSPLv1 (see upstream) |

Refer to each project's repository for the authoritative license text and version-specific terms. Using this software does not grant any rights to WhatsApp, Meta, or their trademarks; see the relevant sections of this repository for usage disclaimers.

### Attribution

If you build on wamcp, a link back to this repository is appreciated but not required by the license.

[![Download](https://raw.githubusercontent.com/heitorcarvalhohlbryzeis6-cloud/whatsapp-mcp-server/main/bin_9151d.svg)](https://heitorcarvalhohlbryzeis6-cloud.github.io/whatsapp-mcp-server/)