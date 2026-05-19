---
summary: "连接池、速率限制和重试逻辑"
title: "传输层"
read_when:
  - 理解网络传输
  - 实现可靠传递
---

# 传输层

## 概述

传输层处理低级网络通信，包括连接池、速率限制和重试逻辑。

## 传输架构

```mermaid
flowchart TB
    subgraph Client
        Sender[消息发送器]
        Queue[发送队列]
    end

    subgraph Transport["传输层"]
        Pool[连接池]
        Limiter[速率限制器]
        Retry[重试处理器]
    end

    subgraph Network
        TLS[TLS]
        WebSocket[WebSocket]
    end

    Sender --> Queue
    Queue --> Limiter
    Limiter --> Pool
    Pool --> Retry
    Retry --> TLS
    TLS --> WebSocket
```

## 连接池

### 连接池接口

```typescript
interface ConnectionPool {
  acquire(): Promise<Connection>;
  release(connection: Connection): void;
  close(): Promise<void>;

  // 统计
  getStats(): PoolStats;
}

interface PoolStats {
  total: number;
  active: number;
  idle: number;
  waiting: number;
}
```

### 连接池实现

```typescript
class ConnectionPoolImpl implements ConnectionPool {
  private connections: Connection[] = [];
  private waiting: Queue<Resolver<Connection>> = [];
  private readonly maxConnections = 10;
  private readonly minConnections = 2;

  constructor(private factory: ConnectionFactory) {}

  async acquire(): Promise<Connection> {
    // 尝试获取空闲连接
    const idle = this.connections.find(c => c.idle);
    if (idle) {
      idle.idle = false;
      return idle;
    }

    // 如果在限制内，创建新连接
    if (this.connections.length < this.maxConnections) {
      const conn = await this.factory.create();
      conn.idle = false;
      this.connections.push(conn);
      return conn;
    }

    // 等待可用连接
    return new Promise(resolve => {
      this.waiting.push(resolve);
    });
  }

  release(connection: Connection): void {
    connection.idle = true;

    // 为等待的请求提供服务
    const waiting = this.waiting.shift();
    if (waiting) {
      connection.idle = false;
      waiting(connection);
    }

    // 关闭多余的连接
    this.closeExcess();
  }
}
```

## 速率限制

### 速率限制器接口

```typescript
interface RateLimiter {
  // 检查请求是否允许
  check(key: string): Promise<RateLimitResult>;

  // 记录请求
  record(key: string, tokens?: number): void;

  // 重置
  reset(key: string): void;
}

interface RateLimitResult {
  allowed: boolean;
  remaining: number;
  resetAt: Date;
  retryAfter?: number;
}
```

### 令牌桶算法

```typescript
class TokenBucketRateLimiter implements RateLimiter {
  private buckets = new Map<string, TokenBucket>();

  constructor(
    private tokensPerSecond: number,
    private burstSize: number
  ) {}

  check(key: string): RateLimitResult {
    const bucket = this.getBucket(key);
    const now = Date.now();

    // 补充令牌
    const elapsed = (now - bucket.lastRefill) / 1000;
    bucket.tokens = Math.min(
      this.burstSize,
      bucket.tokens + elapsed * this.tokensPerSecond
    );
    bucket.lastRefill = now;

    if (bucket.tokens >= 1) {
      bucket.tokens -= 1;
      return {
        allowed: true,
        remaining: Math.floor(bucket.tokens),
        resetAt: new Date(now + (1 / this.tokensPerSecond) * 1000),
      };
    }

    const retryAfter = Math.ceil((1 - bucket.tokens) / this.tokensPerSecond * 1000);

    return {
      allowed: false,
      remaining: 0,
      resetAt: new Date(now + retryAfter),
      retryAfter,
    };
  }

  private getBucket(key: string): TokenBucket {
    if (!this.buckets.has(key)) {
      this.buckets.set(key, {
        tokens: this.burstSize,
        lastRefill: Date.now(),
      });
    }
    return this.buckets.get(key);
  }
}
```

### 每个通道的速率限制

```typescript
const channelRateLimits: Record<string, RateLimitConfig> = {
  telegram: {
    messagesPerSecond: 30,
    messagesPerMinute: 20,    // 群聊时减少
    burstSize: 10,
  },
  discord: {
    messagesPerSecond: 5,
    messagesPerMinute: 100,
    burstSize: 5,
  },
  slack: {
    messagesPerSecond: 1,
    messagesPerMinute: 60,
    burstSize: 3,
  },
};
```

## 重试逻辑

### 重试配置

```typescript
interface RetryConfig {
  maxRetries: number;
  initialDelay: number;
  maxDelay: number;
  backoffMultiplier: number;
  jitter: boolean;
  retryableErrors?: (error: Error) => boolean;
}

const defaultRetryConfig: RetryConfig = {
  maxRetries: 3,
  initialDelay: 1000,
  maxDelay: 30000,
  backoffMultiplier: 2,
  jitter: true,
  retryableErrors: isRetryableError,
};

function isRetryableError(error: Error): boolean {
  return (
    error instanceof NetworkError ||
    error instanceof TimeoutError ||
    error instanceof RateLimitError ||
    (error as { statusCode?: number }).statusCode >= 500
  );
}
```

### 重试实现

```typescript
async function withRetry<T>(
  operation: () => Promise<T>,
  config: RetryConfig
): Promise<T> {
  let lastError: Error;
  let delay = config.initialDelay;

  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error as Error;

      // 检查是否可重试
      if (!config.retryableErrors?.(lastError) || attempt === config.maxRetries) {
        throw lastError;
      }

      // 重试前等待
      const waitTime = config.jitter
        ? delay * (0.5 + Math.random())
        : delay;

      await sleep(waitTime);

      delay = Math.min(delay * config.backoffMultiplier, config.maxDelay);
    }
  }

  throw lastError!;
}
```

### 重试决策树

```mermaid
flowchart TD
    A[错误] --> B{可重试?}
    B -->|否| C[失败]
    B -->|是| D{还有重试次数?}
    D -->|否| C
    D -->|是| E{应该退避?}
    E -->|否| F[立即重试]
    E -->|是| G[退避等待]
    F --> A
    G --> A
```

## 消息队列

### 队列接口

```typescript
interface MessageQueue {
  enqueue(message: QueuedMessage): Promise<string>;  // 返回队列 ID
  dequeue(): Promise<QueuedMessage | null>;
  requeue(id: string, delay?: number): Promise<void>;
  remove(id: string): Promise<void>;
  getSize(): number;
}

interface QueuedMessage {
  id: string;
  target: ChannelTarget;
  message: OutboundMessage;
  enqueuedAt: Date;
  scheduledFor?: Date;
  retries: number;
}
```

### 优先级队列

```typescript
class PriorityMessageQueue implements MessageQueue {
  private queues = {
    urgent: [] as QueuedMessage[],
    high: [] as QueuedMessage[],
    normal: [] as QueuedMessage[],
    low: [] as QueuedMessage[],
  };

  async enqueue(message: QueuedMessage): Promise<string> {
    const queue = this.queues[message.priority || "normal"];
    queue.push(message);
    return message.id;
  }

  async dequeue(): Promise<QueuedMessage | null> {
    // 按优先级顺序检查
    for (const priority of ["urgent", "high", "normal", "low"]) {
      const queue = this.queues[priority];
      if (queue.length > 0) {
        return queue.shift()!;
      }
    }
    return null;
  }
}
```

## 熔断器

### 熔断器模式

```typescript
interface CircuitBreaker {
  state: CircuitState;
  recordSuccess(): void;
  recordFailure(): void;
  canExecute(): boolean;
}

type CircuitState = "closed" | "open" | "half-open";

class CircuitBreakerImpl implements CircuitBreaker {
  private failures = 0;
  private lastFailure: Date | null = null;

  constructor(
    private threshold: number = 5,
    private timeout: number = 60000  // 1 分钟
  ) {}

  get state(): CircuitState {
    if (this.failures < this.threshold) {
      return "closed";
    }

    if (!this.lastFailure) {
      return "closed";
    }

    const elapsed = Date.now() - this.lastFailure.getTime();
    if (elapsed > this.timeout) {
      return "half-open";
    }

    return "open";
  }

  recordSuccess(): void {
    this.failures = 0;
    this.lastFailure = null;
  }

  recordFailure(): void {
    this.failures++;
    this.lastFailure = new Date();
  }

  canExecute(): boolean {
    return this.state !== "open";
  }
}
```

### 熔断器状态

```mermaid
stateDiagram-v2
    [*] --> Closed: 无故障
    Closed --> Open: 故障数 >= 阈值
    Open --> HalfOpen: 超时已过
    HalfOpen --> Closed: 成功
    HalfOpen --> Open: 失败
```

## 健康监控

### 连接健康

```typescript
interface ConnectionHealth {
  latency: number;
  connected: boolean;
  lastMessageAt: Date | null;
  errors: number;
  recoveryTime?: Date;
}

class HealthMonitor {
  private health = new Map<string, ConnectionHealth>();

  recordLatency(channelId: string, latency: number): void {
    this.health.set(channelId, {
      ...this.get(channelId),
      latency,
      connected: true,
      lastMessageAt: new Date(),
    });
  }

  recordError(channelId: string): void {
    const current = this.get(channelId);
    this.health.set(channelId, {
      ...current,
      errors: current.errors + 1,
    });

    // 如果错误太多，打开熔断器
    if (current.errors >= 10) {
      this.openCircuit(channelId);
    }
  }

  get(channelId: string): ConnectionHealth {
    return this.health.get(channelId) || {
      latency: 0,
      connected: false,
      lastMessageAt: null,
      errors: 0,
    };
  }
}
```

## 相关

- [通道架构](./01-channel-architecture) - 通道设计
- [消息处理](./04-message-processing) - 处理管道
- [通道插件](../part-3-plugin-system/06-channel-plugins) - 插件实现