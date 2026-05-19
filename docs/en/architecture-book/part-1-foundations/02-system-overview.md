---
summary: "High-level system architecture, layered design, and component responsibilities"
title: "System Architecture Overview"
read_when:
  - Understanding the big picture
  - Tracing data flows
---

# System Architecture Overview

## Layered Architecture

OpenClaw follows a strict layered architecture where each layer has clear responsibilities and boundaries:

```mermaid
flowchart TB
    subgraph Clients
        CLI[CLI]
        WebUI[Web UI]
        MacApp[macOS App]
        Mobile[Mobile Nodes]
    end

    subgraph Gateway["Gateway Layer"]
        WS[WebSocket Server]
        Protocol[Protocol Handler]
        Router[Message Router]
        Sessions[Session Manager]
    end

    subgraph Agent["Agent Layer"]
        Loop[Agent Loop]
        Tools[Tool Executor]
        Memory[Memory Manager]
        Context[Context Engine]
    end

    subgraph Plugin["Plugin Layer"]
        Providers[Providers]
        Channels[Channels]
        Runtimes[Runtimes]
    end

    subgraph Extensions
        AI[AI Providers]
        IM[IM Platforms]
        Tools[External Tools]
    end

    Clients --> Gateway
    Gateway --> Agent
    Gateway --> Plugin
    Plugin --> Extensions
```

## Layer Responsibilities

### Gateway Layer

The Gateway is the central hub that owns all messaging surfaces.

**Core responsibilities:**

| Component | Responsibility |
|-----------|----------------|
| WebSocket Server | Client connections, protocol framing |
| Protocol Handler | Message validation, routing |
| Message Router | Session resolution, agent routing |
| Session Manager | Session lifecycle, state management |

**Key invariants:**
- Exactly one Gateway controls messaging per host
- Handshake is mandatory before any communication
- All clients must authenticate before use

### Agent Layer

The Agent layer handles AI interaction and tool execution.

**Core responsibilities:**

| Component | Responsibility |
|-----------|----------------|
| Agent Loop | Inference, tool calling, response streaming |
| Tool Executor | Tool discovery, execution, result handling |
| Memory Manager | Context assembly, compaction, retrieval |
| Context Engine | Prompt assembly, token budgeting |

**Execution model:**
```mermaid
sequenceDiagram
    participant Gateway
    participant AgentLoop
    participant ToolExecutor
    participant ContextEngine
    participant Model

    Gateway->>AgentLoop: Start run
    AgentLoop->>ContextEngine: Build context
    ContextEngine-->>AgentLoop: Assembled prompt
    AgentLoop->>Model: Inference request
    loop Tool Calls
        Model-->>AgentLoop: Tool call
        AgentLoop->>ToolExecutor: Execute tool
        ToolExecutor-->>AgentLoop: Tool result
        AgentLoop->>Model: Continue inference
    end
    AgentLoop-->>Gateway: Final response
```

### Plugin Layer

The Plugin layer provides extensibility through providers, channels, and runtimes.

**Plugin types:**

| Type | Description | Example |
|------|-------------|---------|
| Provider | AI provider integration | openai, anthropic |
| Channel | Messaging platform | telegram, discord |
| Tool | External capability | browser, tavily |
| Runtime | Agent execution | pi, codex |
| Memory | Knowledge storage | wiki, lancedb |

## Core Design Principles

### 1. Control Plane vs Runtime Plane

OpenClaw separates control plane operations from runtime operations:

**Control plane (lightweight):**
- Plugin discovery and inventory
- Manifest parsing and validation
- Configuration checking
- Status queries

**Runtime plane (on-demand):**
- Actual plugin execution
- Model inference
- Message sending
- Tool execution

```mermaid
flowchart LR
    subgraph Control["Control Plane"]
        Discover[Discovery]
        Validate[Validation]
        Config[Configuration]
    end

    subgraph Runtime["Runtime Plane"]
        Execute[Execution]
        Infer[Inference]
        Send[Message Send]
    end

    Control --> Runtime
```

### 2. Manifest-First Design

Plugin configuration is derived from manifest metadata before runtime execution:

```typescript
// Plugin manifest defines behavior
{
  "id": "provider/openai",
  "name": "OpenAI Provider",
  "version": "1.0.0",
  "providers": [{
    "id": "openai",
    "models": ["gpt-4o", "gpt-4o-mini"]
  }]
}

// Control plane uses manifest
const config = await pluginRegistry.validate(manifest);

// Runtime plane uses validated config
const result = await provider.createCompletion(config);
```

### 3. Lazy Loading

Plugins are loaded on-demand, not eagerly at startup:

```typescript
// Lazy activation pattern
const loader = new PluginRuntimeLoader({
  lazy: true,
  manifestResolver: new ManifestResolver(),
});

await loader.activate("provider/openai", { lazy: true });
```

### 4. Contract Boundaries

Plugins cross into core only through well-defined contracts:

```mermaid
flowchart LR
    Core[OpenClaw Core]
    SDK[Plugin SDK]
    Plugin[Plugin Code]

    Core --> SDK
    SDK --> Plugin
    Plugin --> SDK
    SDK --> Core

    style Core fill:#e1f5fe
    style SDK fill:#fff3e0
    style Plugin fill:#e8f5e9
```

**Allowed crossings:**
- `openclaw/plugin-sdk/*` - Public SDK surface
- `manifest.json` - Metadata only
- Injected runtime helpers
- Documented barrel exports

**Forbidden crossings:**
- `src/**` internal modules
- Other plugin internals
- Core implementation details

## Data Flow

### Message Flow (Inbound)

```mermaid
sequenceDiagram
    participant Channel
    participant Gateway
    participant Router
    participant Session
    participant Agent

    Channel->>Gateway: Inbound message
    Gateway->>Router: Parse and classify
    Router->>Session: Resolve session
    Session->>Agent: Route to agent
    Agent-->>Session: Response
    Session-->>Router: Formatted message
    Router-->>Channel: Send response
```

### Agent Execution Flow

```mermaid
flowchart TB
    A[User Input] --> B[Session Context]
    B --> C[Memory Retrieval]
    C --> D[Prompt Assembly]
    D --> E[Model Inference]
    E --> F{Function Call?}
    F -->|Yes| G[Tool Execution]
    G --> E
    F -->|No| H[Response]
    H --> I[Memory Update]
    I --> J[Stream to Client]
```

## Component Diagram

### Complete System

```mermaid
flowchart TB
    subgraph External
        User[Chat Users]
        Dev[Developers]
    end

    subgraph Surface
        Telegram[Telegram]
        Discord[Discord]
        WhatsApp[WhatsApp]
        Slack[Slack]
    end

    subgraph Core
        Gateway[Gateway]
        Agent[Agent Runtime]
        Session[Session Manager]
        Memory[Memory System]
        Config[Config Manager]
    end

    subgraph Plugins
        ProviderPlugin[Provider Plugins]
        ChannelPlugin[Channel Plugins]
        ToolPlugin[Tool Plugins]
    end

    subgraph Infra
        FileSystem[File System]
        Database[Databases]
        Network[Network]
    end

    User --> Surface
    Dev --> Core
    Surface --> Core
    Core --> Plugins
    Core --> Infra
    Plugins --> Infra
```

## State Management

### Session State

Each session maintains independent state:

```typescript
interface SessionState {
  id: string;
  channel: string;
  context: ConversationContext;
  memory: MemorySnapshot;
  tools: ToolBindings;
  metadata: SessionMetadata;
}
```

### Gateway State

Gateway maintains global state:

```typescript
interface GatewayState {
  sessions: Map<SessionKey, Session>;
  plugins: PluginRegistry;
  channels: ChannelRegistry;
  providers: ProviderRegistry;
  health: HealthStatus;
}
```

## Error Handling

### Layer-Specific Errors

Each layer handles its own error domain:

| Layer | Error Type | Handling |
|-------|------------|----------|
| Gateway | Connection, protocol errors | Reject and close |
| Agent | Inference, tool errors | Graceful degradation |
| Plugin | Load, execution errors | Plugin isolation |
| Channel | Send, receive errors | Retry and queue |

### Error Propagation

Errors flow up through layers with context:

```typescript
// Error with context
throw new PluginError("provider/openai", "Model unavailable", {
  cause: originalError,
  recoverable: true,
  retryAfter: 5000,
});
```

## Related

- [Core Concepts](/architecture-book/part-1-foundations/03-core-concepts) - Key abstractions
- [Gateway Protocol](/architecture-book/part-4-gateway-protocol/01-protocol-overview) - Protocol details
- [Plugin System](/architecture-book/part-3-plugin-system/01-plugin-architecture) - Plugin architecture