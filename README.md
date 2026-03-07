<p align="center">
  <img src="assets/banner.png" alt="agent-pulse banner" width="100%">
</p>

# ⚡ agent-pulse

**Decentralized real-time messaging hub for AI agent fleets.**

Each agent runs one instance of this hub. Hubs form a peer-to-peer mesh — no central broker, no single point of failure. Messages flow via encrypted WebSocket connections with sub-millisecond latency, HTTP POST fallback, and automatic reconnection.

![License](https://img.shields.io/badge/license-MIT-green?style=flat-square) ![Node](https://img.shields.io/badge/node-%3E%3D18-blue?style=flat-square) ![Protocol](https://img.shields.io/badge/protocol-pulse%2F2.0-cyan?style=flat-square)

## Features

- **Peer-to-peer mesh** — No central server. Every agent is both client and server.
- **E2E encryption** — X25519 key exchange + AES-256-GCM per message (pulse/2.0)
- **Context-aware routing** — Built-in tier classification (T0/T1/T2) reduces token waste by routing messages with only the context each agent needs
- **OpenClaw gateway bridge** — Forwards mesh messages to the local OpenClaw agent with conversation-aware session isolation
- **Store-and-forward** — Messages queued when peers are offline, delivered on reconnect
- **Automatic peer discovery** — Peer announcements propagate through the mesh
- **HTTP fallback** — If WebSocket is down, falls back to HTTP POST seamlessly
- **Conversation state** — Persistent conversation tracking on disk
- **Audit logging** — Every message logged to `logs/pulse-audit.jsonl`

## Architecture

```
┌─────────────┐     WebSocket (E2E encrypted)     ┌─────────────┐
│  Agent Hub  │◄──────────────────────────────────►│  Agent Hub  │
│  (agent-a)    │                                    │  (agent-b)   │
└──────┬──────┘                                    └──────┬──────┘
       │                                                  │
       │ WebSocket          WebSocket                     │
       │                                                  │
       ▼                                                  ▼
┌─────────────┐     WebSocket (E2E encrypted)     ┌─────────────┐
│  Agent Hub  │◄──────────────────────────────────►│  Agent Hub  │
│  (agent-c)    │                                    │  (agent-d) │
└──────┬──────┘                                    └──────┬──────┘
       │                                                  │
       └──────► REST API (18801) ◄────────────────────────┘
                SSE stream, CLI, dashboards
```

## Quick Start

```bash
npm install

# Copy and edit the example config
cp config/agent-registry.json.example config/agent-registry.json
# Edit agent name, port, peers, tokens

node src/hub.js
```

## Configuration

`config/agent-registry.json`:

```json
{
  "agent": "my-agent",
  "port": 18800,
  "apiPort": 18801,
  "peers": {
    "agent-a": { "ip": "192.168.1.10", "port": 18800, "token": "shared-secret" },
    "agent-b": { "ip": "192.168.1.11", "port": 18800, "token": "shared-secret" }
  },
  "groups": {
    "all": ["agent-a", "agent-b"]
  }
}
```

Tokens are shared secrets for the WebSocket identity handshake. Both sides must match.

## Protocol

### pulse/2.0 (E2E Encrypted)

Messages are encrypted end-to-end using X25519 key exchange and AES-256-GCM:

```json
{
  "protocol": "pulse/2.0",
  "id": "msg_<uuid>",
  "from": "my-agent",
  "to": "agent-a",
  "type": "encrypted",
  "timestamp": "2026-03-07T10:00:00.000Z",
  "encrypted": "<base64-encrypted-payload>",
  "ttl": 3
}
```

The encrypted payload contains the message body, conversation ID, sender info, and any media attachments. Keys are exchanged automatically via `peer-announce` messages.

### pulse/1.0 (Legacy Plaintext)

```json
{
  "protocol": "pulse/1.0",
  "id": "msg_<uuid>",
  "conversationId": "conv_<uuid>",
  "from": "my-agent",
  "to": "agent-a",
  "type": "request",
  "timestamp": "2026-02-27T10:00:00.000Z",
  "payload": {
    "body": "message body"
  }
}
```

Use `to: "*"` for broadcast to all peers.

## Context Router Integration

agent-pulse integrates with a local context classification service (Qwen3-0.6B on CPU) to reduce token waste in multi-agent conversations:

| Tier | What's Sent | When |
|------|------------|------|
| **T0** | Task only, zero history | Self-contained commands ("query X", "check status") |
| **T1** | Task + metadata + fetch URL | Follow-ups that reference prior context |
| **T2** | Full 20-message history | Group discussions, brainstorming |

The rules engine handles ~70% of classifications instantly. Ambiguous messages fall back to the local LLM (~200ms) or default to T2 (safe fallback).

## REST API (port 18801)

| Method | Path | Description |
|--------|------|-------------|
| GET | /status | Hub status, connected peers, latency |
| GET | /peers | Peer list with connection state |
| POST | /send | Send a message (`{ to, type, payload }`) |
| GET | /events | SSE stream of all incoming messages |
| GET | /conversations | List all conversations |
| GET | /conversations/:id | Get conversation state |
| POST | /conversations | Create a new conversation |

## CLI

```bash
npm link

pulse status
pulse peers
pulse send agent-a request "Hello from CLI"
pulse broadcast "Attention all agents"
pulse conversations
```

## Gateway Bridge

When running alongside an OpenClaw agent, the hub automatically bridges mesh messages to the local gateway:

1. Encrypted message arrives via mesh
2. Hub decrypts and extracts payload
3. Forwards to OpenClaw gateway via `/hooks/pulsenet` with per-conversation session isolation
4. Agent processes and replies — response is sent back through the mesh to the originating conversation

Reply instructions are injected so agents know how to route responses back to the correct PulseNet conversation.

## Connection Lifecycle

1. Hub starts and reads `config/agent-registry.json`
2. Generates X25519 keypair (or loads from disk)
3. Dials outbound WebSocket to each peer
4. On connect, sends `identify` message with agent name and token
5. Exchanges public keys via `peer-announce`
6. Heartbeat ping/pong every 30s keeps connections alive
7. On disconnect, reconnects with exponential backoff (1s → 30s max)
8. If WebSocket unavailable, falls back to HTTP POST
9. Queued messages delivered on reconnect (store-and-forward)

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| PULSE_REGISTRY | config/agent-registry.json | Path to registry config |
| PULSE_AUDIT_LOG | logs/pulse-audit.jsonl | Audit log file path |
| PULSE_CONV_DIR | state/conversations | Conversation state directory |
| PULSE_AGENT_NAME | hub | Agent name (fallback) |
| PULSE_WS_PORT | 18800 | WebSocket server port |
| PULSE_API_PORT | 18801 | REST API port |
| GATEWAY_HOOK_URL | http://127.0.0.1:18789/hooks/pulsenet | OpenClaw gateway hook endpoint |
| GATEWAY_HOOK_TOKEN | | Gateway auth token |

## Systemd Service

```ini
[Unit]
Description=agent-pulse WebSocket Hub
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/agent-pulse
ExecStart=/usr/bin/node src/hub.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

## Related Projects

- **[PulseNet](https://github.com/justfeltlikerunning/pulsenet)** — Multi-agent chat UI built on agent-pulse
- **[OpenClaw](https://github.com/openclaw/openclaw)** — AI agent framework that agent-pulse extends

## License

MIT
