---
summary: "RPC 方法和事件类型的完整 API 参考"
title: "API 参考"
read_when:
  - 查找 API 详情
  - 构建集成
---

# API 参考

## 概述

OpenClaw Gateway RPC 方法、事件和类型的完整参考。

## RPC 方法

### 核心方法

| 方法 | 描述 | 幂等性 |
|--------|-------------|------------|
| `connect` | 初始握手 | N/A |
| `disconnect` | 优雅断开 | N/A |
| `agent` | 运行 Agent | 是 |
| `agent_abort` | 中止 Agent 运行 | 是 |
| `agent_status` | 获取运行状态 | 是 |
| `send` | 发送消息 | 是 |
| `send_reply` | 回复消息 | 是 |
| `session_create` | 创建会话 | 是 |
| `session_get` | 获取会话信息 | 是 |
| `session_reset` | 重置会话 | 是 |
| `session_delete` | 删除会话 | 是 |
| `health` | 健康检查 | 是 |
| `status` | 系统状态 | 是 |

## Connect 方法

### Connect 请求

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

### Connect 响应

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

## Agent 方法

### Agent 请求

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

### Agent 响应

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

### Agent 事件

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

## Send 方法

### Send 请求

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

### Send 响应

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

## Session 方法

### Session 创建

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

### Session 获取

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

### Session 重置

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

## 健康检查方法

### 健康检查

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

## Status 方法

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

## 事件类型

### Tick 事件

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

### Presence 事件

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

### Chat 事件

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

## 错误码

### 错误响应

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

### 错误码参考

| 代码 | HTTP 等价 | 描述 |
|------|------------|-------------|
| `AUTH_FAILED` | 401 | 凭据无效 |
| `AUTH_EXPIRED` | 401 | Token 已过期 |
| `AUTH_REQUIRED` | 401 | 未提供认证 |
| `DEVICE_NOT_PAIRED` | 403 | 设备未配对 |
| `DEVICE_REJECTED` | 403 | 配对被拒绝 |
| `VALIDATION_ERROR` | 400 | 请求无效 |
| `SESSION_NOT_FOUND` | 404 | 会话不存在 |
| `SESSION_EXISTS` | 409 | 会话已存在 |
| `AGENT_ERROR` | 500 | Agent 执行失败 |
| `CHANNEL_ERROR` | 500 | Channel 操作失败 |
| `RATE_LIMITED` | 429 | 请求过多 |
| `INTERNAL_ERROR` | 500 | Gateway 错误 |

## 类型

### 消息类型

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

### Channel 类型

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

## 相关内容

- [App SDK](./01-app-sdk) - 客户端 SDK
- [插件 SDK](./02-plugin-sdk) - 插件 SDK
- [协议概述](../part-4-gateway-protocol/01-protocol-overview) - 协议设计
