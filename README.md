# agent-pulse

Real-time WebSocket messaging hub for AI agent fleets.

Each agent runs one instance of this hub. Hubs maintain persistent bidirectional WebSocket connections to all peer agents. Messages flow in sub-millisecond time instead of HTTP round-trips.

## Architecture

- WebSocket server on port 18800 (configurable) - accepts inbound connections from peers
- WebSocket clients dialing all known peers in the registry
- HTTP POST fallback when WebSocket connection is down
- REST API on port 18801 for CLI tools and local integrations
- SSE stream of all incoming messages (for dashboards and bridges)
- Conversation state stored on disk in `state/conversations/`
- Audit log written to `logs/pulse-audit.jsonl`

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
    "agent-a": { "ip": "10.0.0.10", "port": 18800, "token": "secret-a" },
    "agent-b": { "ip": "10.0.0.11", "port": 18800, "token": "secret-b" }
  },
  "groups": {
    "all": ["agent-a", "agent-b"]
  }
}
```

The `token` for each peer is the shared secret used during the WebSocket identity handshake. Both sides must have matching tokens for the connection to authenticate.

## Message Envelope

All messages use this JSON format:

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
    "subject": "optional subject",
    "body": "message body"
  }
}
```

Use `to: "*"` for broadcast to all peers.

## REST API (port 18801)

| Method | Path | Description |
|--------|------|-------------|
| GET | /status | Hub status, connected peers, latency |
| GET | /peers | Peer list with connection state |
| POST | /send | Send a message (JSON body: `{ to, type, payload }`) |
| GET | /events | SSE stream of all incoming messages |
| GET | /conversations | List all conversations |
| GET | /conversations/:id | Get conversation state |
| POST | /conversations | Create a new conversation |

## CLI

```bash
# Install globally (optional)
npm link

pulse status
pulse peers
pulse send agent-a request "Hello from CLI"
pulse broadcast "Attention all agents"
pulse conversations
```

## Connection Lifecycle

1. Hub starts and reads `config/agent-registry.json`
2. Dials outbound WebSocket to each peer
3. On connect, sends identity message: `{ type: "identify", agent, token }`
4. Peer authenticates and acknowledges
5. Heartbeat ping/pong every 30s keeps connections alive
6. On disconnect, reconnects with exponential backoff (1s, 2s, 4s ... 30s max)
7. If WebSocket unavailable, falls back to HTTP POST

## Conversation State

Conversations are stored as JSON files in `state/conversations/<id>.json`:

```json
{
  "conversationId": "conv_abc123",
  "type": "request",
  "participants": ["my-agent", "agent-a"],
  "status": "active",
  "createdAt": "2026-02-27T10:00:00.000Z",
  "rounds": [
    {
      "round": 1,
      "from": "my-agent",
      "to": "agent-a",
      "type": "request",
      "timestamp": "2026-02-27T10:00:00.000Z",
      "payload": { "body": "Hello" }
    }
  ]
}
```

Status values: `active`, `parked`, `complete`, `timeout`

## Audit Log

All messages are appended to `logs/pulse-audit.jsonl`, one JSON object per line:

```json
{"timestamp":"2026-02-27T10:00:00.000Z","id":"msg_xxx","from":"my-agent","to":"agent-a","type":"request","conversationId":"conv_xxx","status":"delivered","latencyMs":2}
```

## Running Tests

```bash
npm test
# or
node test/test-hub.js
```

The test suite starts two in-process hub instances on ephemeral ports and verifies end-to-end messaging, broadcast, reconnect, audit logging, and the REST API.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| PULSE_REGISTRY | config/agent-registry.json | Path to registry config |
| PULSE_AUDIT_LOG | logs/pulse-audit.jsonl | Audit log file path |
| PULSE_CONV_DIR | state/conversations | Conversation state directory |
| PULSE_AGENT_NAME | hub | Agent name (fallback if no config) |
| PULSE_WS_PORT | 18800 | WebSocket server port (fallback) |
| PULSE_API_PORT | 18801 | REST API port (fallback) |

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
