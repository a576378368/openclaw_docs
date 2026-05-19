---
summary: "Message flow from inbound to outbound, session routing"
title: "Message Flow"
read_when:
  - Tracing message paths
  - Understanding routing logic
---

# Message Flow

## Overview

Messages flow through OpenClaw in a well-defined pipeline, from inbound channel events through session resolution to agent execution and outbound delivery.

## Message Flow Overview

```mermaid
flowchart TB
    subgraph Inbound["Inbound Path"]
        Channel[Channel Event]
        Normalize[Normalize]
        Classify[Classify]
        Resolve[Resolve Session]
    end

    subgraph Execution["Execution Path"]
        Route[Route to Agent]
        Context[Build Context]
        Infer[Model Inference]
        Format[Format Response]
    end

    subgraph Outbound["Outbound Path"]
        Send[Send Message]
        Deliver[Deliver to Channel]
    end

    Channel --> Normalize
    Normalize --> Classify
    Classify --> Resolve
    Resolve --> Route
    Route --> Context
    Context --> Infer
    Infer --> Format
    Format --> Send
    Send --> Deliver
```

## Inbound Path

### Channel Event Reception

```mermaid
sequenceDiagram
    participant Platform
    participant ChannelPlugin
    participant Gateway
    participant Normalizer

    Platform->>ChannelPlugin: Platform event
    ChannelPlugin->>Normalizer: Transform
    Normalizer-->>ChannelPlugin: Normalized event
    ChannelPlugin->>Gateway: InboundMessage
```

### Message Normalization

```typescript
interface InboundMessage {
  id: string;
  channel: string;
  peer: string;
  peerType: "user" | "group" | "channel";
  sender: Sender;
  content: string;
  media?: MediaAttachment;
  timestamp: Date;
  replyTo?: string;
  metadata: Record<string, unknown>;
}

interface Sender {
  id: string;
  name: string;
  username?: string;
  mention?: string;
}
```

### Message Classification

```typescript
interface MessageClassification {
  type: "user" | "command" | "mention" | "callback" | "system";
  intent?: string;
  entities?: Entity[];
}

function classifyMessage(msg: InboundMessage): MessageClassification {
  // Check for command prefix
  if (msg.content.startsWith("/")) {
    return { type: "command", intent: extractCommand(msg.content) };
  }

  // Check for bot mention
  if (msg.content.includes("@bot")) {
    return { type: "mention", intent: extractIntent(msg.content) };
  }

  // Check for callback query
  if (msg.metadata.callbackQuery) {
    return { type: "callback", intent: msg.metadata.callbackQuery.data };
  }

  return { type: "user", intent: extractIntent(msg.content) };
}
```

## Session Resolution

### Session Key Derivation

```typescript
interface SessionResolution {
  sessionKey: string;
  agentId: string;
  scope: SessionScope;
}

function resolveSession(msg: InboundMessage, config: Config): SessionResolution {
  const { channel, peer, peerType } = msg;

  let scope: SessionScope;

  switch (config.session.dmScope) {
    case "per-channel-peer":
      scope = peerType === "user" ? "dm" : "group";
      break;
    case "per-channel":
      scope = "channel";
      break;
    case "global":
      scope = "global";
      break;
  }

  // Derive session key
  const sessionKey = scope === "global"
    ? "main"
    : `${channel}:${scope}:${peer}`;

  // Resolve agent
  const agentId = resolveAgent(sessionKey, config);

  return { sessionKey, agentId, scope };
}
```

### Session Lookup

```mermaid
flowchart TB
    A[Message] --> B[Extract channel + peer]
    B --> C{Lookup Session}
    C -->|Found| D[Use Existing Session]
    C -->|Not Found| E[Create New Session]
    D --> F[Add to History]
    E --> F
    F --> G[Route to Agent]
```

## Execution Path

### Agent Routing

```typescript
interface AgentRouting {
  agentId: string;
  modelRef?: string;
  tools?: string[];
}

function routeToAgent(
  msg: InboundMessage,
  session: Session,
  config: Config
): AgentRouting {
  // Check for explicit agent selection
  if (msg.metadata.agentId) {
    return { agentId: msg.metadata.agentId };
  }

  // Check session binding
  if (session.agentId) {
    return { agentId: session.agentId };
  }

  // Use default agent
  return { agentId: config.agents.default };
}
```

### Context Building

```mermaid
sequenceDiagram
    participant Agent
    participant ContextBuilder
    participant Memory
    participant Config

    Agent->>ContextBuilder: buildContext(message, session)
    ContextBuilder->>Memory: get recent history
    Memory-->>ContextBuilder: messages
    ContextBuilder->>Memory: search relevant memories
    Memory-->>ContextBuilder: memories
    ContextBuilder->>Config: get system prompt
    Config-->>ContextBuilder: system prompt
    ContextBuilder-->>Agent: assembled context
```

### Inference Pipeline

```typescript
async function runInference(
  params: InferenceParams
): Promise<AsyncIterable<InferenceEvent>> {
  const context = await buildContext(params);

  // Stream response
  const stream = await provider.createCompletion({
    model: params.modelRef,
    messages: context.messages,
    system: context.systemPrompt,
    temperature: params.temperature,
    stream: true,
  });

  for await (const chunk of stream) {
    yield {
      type: "delta",
      delta: chunk.delta,
      usage: chunk.usage,
    };

    // Handle tool calls
    if (chunk.toolCalls) {
      yield { type: "tool_calls", calls: chunk.toolCalls };
      // Execute tools and continue...
    }
  }

  yield { type: "complete", summary: await generateSummary(context) };
}
```

## Outbound Path

### Response Formatting

```typescript
interface OutboundFormatter {
  format(
    response: AgentResponse,
    format: "markdown" | "html" | "plain"
  ): FormattedMessage;
}

class ResponseFormatter implements OutboundFormatter {
  format(response: AgentResponse, format: Format): FormattedMessage {
    return {
      content: this.transformContent(response.content, format),
      media: response.media,
      buttons: response.buttons,
      replyTo: response.replyTo,
    };
  }

  private transformContent(content: string, format: Format): string {
    switch (format) {
      case "html":
        return this.markdownToHtml(content);
      case "plain":
        return this.stripMarkdown(content);
      default:
        return content;
    }
  }
}
```

### Channel Delivery

```mermaid
sequenceDiagram
    participant Gateway
    participant Formatter
    participant ChannelPlugin
    participant Platform

    Gateway->>Formatter: format response
    Formatter-->>Gateway: formatted message
    Gateway->>ChannelPlugin: send(target, message)
    ChannelPlugin->>Platform: platform API call
    Platform-->>ChannelPlugin: sent confirmation
    ChannelPlugin-->>Gateway: messageId
```

### Send Pipeline

```typescript
async function sendMessage(
  target: ChannelTarget,
  message: OutboundMessage,
  options: SendOptions = {}
): Promise<SendResult> {
  // Validate target
  validateTarget(target);

  // Format message
  const formatted = formatter.format(message, options.format);

  // Check rate limits
  await rateLimiter.check(target.channel);

  // Send with retry
  const result = await withRetry(
    () => channel.send(target, formatted),
    {
      maxRetries: 3,
      backoff: { initial: 1000, multiplier: 2 },
    }
  );

  return {
    messageId: result.messageId,
    timestamp: new Date(),
  };
}
```

## Error Flow

### Error Handling Pipeline

```mermaid
flowchart TB
    A[Error Occurs] --> B{Error Type}
    B -->|Retryable| C[Retry with backoff]
    B -->|Permanent| D[Return error]
    B -->|Partial| E[Partial success]

    C --> F{Retry count < max?}
    F -->|Yes| G[Execute again]
    F -->|No| H[Give up]

    D --> I[Format error response]
    E --> J[Return partial + error]
    H --> I
```

### Error Recovery

```typescript
async function withErrorRecovery<T>(
  operation: () => Promise<T>
): Promise<T> {
  try {
    return await operation();
  } catch (error) {
    if (isRetryableError(error)) {
      return handleRetryableError(error);
    }

    if (isPartialSuccess(error)) {
      return handlePartialSuccess(error);
    }

    throw formatError(error);
  }
}

function isRetryableError(error: Error): boolean {
  return (
    error instanceof NetworkError ||
    error instanceof TimeoutError ||
    error instanceof RateLimitError
  );
}
```

## Idempotency

### Idempotency Flow

```typescript
interface IdempotentOperation {
  idemKey: string;
  operation: () => Promise<Result>;
  cache: Map<string, CachedResult>;
}

async function executeIdempotent(
  op: IdempotentOperation
): Promise<Result> {
  const { idemKey, operation, cache } = op;

  // Check cache
  if (cache.has(idemKey)) {
    return cache.get(idemKey).result;
  }

  // Execute operation
  const result = await operation();

  // Cache result (short TTL)
  cache.set(idemKey, {
    result,
    expiresAt: Date.now() + 60000,
  });

  return result;
}
```

## Related

- [Protocol Overview](/architecture-book/part-4-gateway-protocol/01-protocol-overview) - Protocol design
- [WebSocket Transport](/architecture-book/part-4-gateway-protocol/02-ws-transport) - Transport layer
- [Events and RPC](/architecture-book/part-4-gateway-protocol/04-events-and-rpc) - Communication patterns