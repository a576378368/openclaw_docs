---
summary: "Complete API reference for RPC methods and event types"
title: "API Reference"
read_when:
  - Looking up API details
  - Building integrations
---

# API Reference

## Overview

Complete reference for OpenClaw Gateway RPC methods, events, and types.

## RPC Methods

### Core Methods

| Method | Description | Idempotent |
|--------|-------------|------------|
| `connect` | Initial handshake | N/A |
| `disconnect` | Graceful disconnect | N/A |
| `agent` | Run agent | Yes |
| `agent_abort` | Abort agent run | Yes |
| `agent_status` | Get run status | Yes |
| `send` | Send message | Yes |
| `send_reply` | Reply to message | Yes |
| `session_create` | Create session | Yes |
| `session_get` | Get session info | Yes |
| `session_reset` | Reset session | Yes |
| `session_delete` | Delete session | Yes |
| `health` | Health check | Yes |
| `status` | System status | Yes |

## Connect Method

### Connect Request

```typescript
interface ConnectRequest {
  type: "connect";
  params: {
    auth: {
      token?: string;
      password?: string;
    };
    device: {
      id: string;
      name: string;
      platform: string;
      family?: string;
    };
    client: {
      version: string;
      name: string;
    };
    role?: "operator" | "node";
    capabilities?: string[];
  };
}
```

### Connect Response

```typescript
interface ConnectResponse {
  type: "res";
  id: string;
  ok: true;
  payload: {
    serverVersion: string;
    features: {
      methods: string[];
      events: string[];
      streaming: boolean;
    };
    health: HealthStatus;
    presence: PresenceSnapshot;
  };
}
```

## Agent Method

### Agent Request

```typescript
interface AgentRequest {
  type: "req";
  id: string;
  method: "agent";
  params: {
    sessionKey: string;
    agentId: string;
    input: string;
    modelRef?: string;
    temperature?: number;
    maxTokens?: number;
    idemKey?: string;
  };
}
```

### Agent Response

```typescript
interface AgentResponse {
  type: "res";
  id: string;
  ok: true;
  payload: {
    runId: string;
    status: "accepted";
  };
}
```

### Agent Events

```typescript
// Start event
{
  type: "event",
  event: "agent",
  payload: {
    runId: string;
    type: "start";
  }
}

// Delta event
{
  type: "event",
  event: "agent",
  payload: {
    runId: string;
    type: "assistant.delta";
    delta: string;
  }
}

// Tool use event
{
  type: "event",
  event: "agent",
  payload: {
    runId: string;
    type: "tool_use";
    tool: string;
    input: unknown;
  }
}

// Complete event
{
  type: "event",
  event: "agent",
  payload: {
    runId: string;
    type: "complete";
    summary: string;
  }
}
```

## Send Method

### Send Request

```typescript
interface SendRequest {
  type: "req";
  id: string;
  method: "send";
  params: {
    channel: string;
    target: string;
    message: {
      content?: string;
      media?: MediaAttachment;
      buttons?: InlineButton[][];
      format?: "markdown" | "html" | "plain";
    };
    idemKey?: string;
  };
}
```

### Send Response

```typescript
interface SendResponse {
  type: "res";
  id: string;
  ok: true;
  payload: {
    messageId: string;
    timestamp: string;
  };
}
```

## Session Methods

### Session Create

```typescript
interface SessionCreateRequest {
  type: "req";
  id: string;
  method: "session_create";
  params: {
    sessionKey: string;
    agentId?: string;
    metadata?: Record<string, unknown>;
  };
}
```

### Session Get

```typescript
interface SessionGetRequest {
  type: "req";
  id: string;
  method: "session_get";
  params: {
    sessionKey: string;
  };
}

interface SessionGetResponse {
  type: "res";
  id: string;
  ok: true;
  payload: {
    key: string;
    agentId: string;
    createdAt: string;
    messageCount: number;
    lastMessageAt?: string;
    metadata: Record<string, unknown>;
  };
}
```

### Session Reset

```typescript
interface SessionResetRequest {
  type: "req";
  id: string;
  method: "session_reset";
  params: {
    sessionKey: string;
  };
}
```

## Health Methods

### Health Check

```typescript
interface HealthRequest {
  type: "req";
  id: string;
  method: "health";
  params: {};
}

interface HealthResponse {
  type: "res";
  id: string;
  ok: true;
  payload: {
    status: "healthy" | "degraded" | "unhealthy";
    uptime: number;
    memory: {
      used: number;
      total: number;
    };
    channels: ChannelHealth[];
    plugins: PluginHealth[];
  };
}

interface ChannelHealth {
  id: string;
  status: "connected" | "disconnected" | "error";
  latency?: number;
  error?: string;
}

interface PluginHealth {
  id: string;
  status: "active" | "inactive" | "error";
  error?: string;
}
```

## Status Method

```typescript
interface StatusRequest {
  type: "req";
  id: string;
  method: "status";
  params: {};
}

interface StatusResponse {
  type: "res";
  id: string;
  ok: true;
  payload: {
    version: string;
    gateway: {
      status: "running";
      uptime: number;
    };
    channels: {
      id: string;
      name: string;
      connected: boolean;
      users: number;
    }[];
    agents: {
      id: string;
      status: "active" | "idle";
      sessions: number;
    }[];
    plugins: {
      id: string;
      type: string;
      version: string;
    }[];
  };
}
```

## Event Types

### Tick Event

```typescript
{
  type: "event",
  event: "tick",
  payload: {
    timestamp: string;
    uptime: number;
    memory: { used: number; total: number };
    sessions: { active: number; total: number };
    channels: {
      id: string;
      status: string;
      latency: number;
    }[];
    health: "healthy" | "degraded" | "unhealthy";
  }
}
```

### Presence Event

```typescript
{
  type: "event",
  event: "presence",
  payload: {
    channels: {
      id: string;
      status: string;
      users: number;
    }[];
    agents: {
      id: string;
      status: string;
      sessions: number;
      running: number;
    }[];
  }
}
```

### Chat Event

```typescript
{
  type: "event",
  event: "chat",
  payload: {
    channel: string;
    target: string;
    message: {
      id: string;
      from: {
        id: string;
        name: string;
        username?: string;
      };
      content: string;
      timestamp: string;
      media?: MediaAttachment;
    };
    sessionKey: string;
  }
}
```

## Error Codes

### Error Response

```typescript
interface ErrorResponse {
  type: "res";
  id: string;
  ok: false;
  error: {
    code: string;
    message: string;
    details?: unknown;
  };
}
```

### Error Code Reference

| Code | HTTP Equiv | Description |
|------|------------|-------------|
| `AUTH_FAILED` | 401 | Invalid credentials |
| `AUTH_EXPIRED` | 401 | Token expired |
| `AUTH_REQUIRED` | 401 | Auth not provided |
| `DEVICE_NOT_PAIRED` | 403 | Device not paired |
| `DEVICE_REJECTED` | 403 | Pairing rejected |
| `VALIDATION_ERROR` | 400 | Invalid request |
| `SESSION_NOT_FOUND` | 404 | Session doesn't exist |
| `SESSION_EXISTS` | 409 | Session already exists |
| `AGENT_ERROR` | 500 | Agent execution failed |
| `CHANNEL_ERROR` | 500 | Channel operation failed |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Gateway error |

## Types

### Message Types

```typescript
interface Message {
  id: string;
  role: "user" | "assistant" | "system";
  content: string;
  timestamp: string;
  metadata?: Record<string, unknown>;
}

interface MediaAttachment {
  id: string;
  type: "image" | "video" | "audio" | "document";
  url?: string;
  mimeType: string;
  size?: number;
  width?: number;
  height?: number;
  duration?: number;
  filename?: string;
}

interface InlineButton {
  label: string;
  data?: string;
  url?: string;
  style?: "primary" | "secondary";
}
```

### Channel Types

```typescript
interface ChannelTarget {
  channel: string;
  peer: string;
  peerType: "user" | "group" | "channel";
  thread?: string;
}

interface Sender {
  id: string;
  name: string;
  username?: string;
  mention?: string;
  isBot: boolean;
}
```

## Related

- [App SDK](/architecture-book/part-6-sdks-apis/01-app-sdk) - Client SDK
- [Plugin SDK](/architecture-book/part-6-sdks-apis/02-plugin-sdk) - Plugin SDK
- [Protocol Overview](/architecture-book/part-4-gateway-protocol/01-protocol-overview) - Protocol design