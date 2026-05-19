---
summary: "Agent runtimes, execution loop, and tool integration"
title: "Agent System"
read_when:
  - Understanding AI execution
  - Building agent integrations
---

# Agent System

## Overview

The Agent system is the core execution engine that runs AI inference and manages tool calls.

```mermaid
flowchart TB
    subgraph Input
        UserInput[User Input]
        SessionContext[Session Context]
        MemoryContext[Memory Context]
    end

    subgraph Agent
        PromptBuilder[Prompt Builder]
        Model[Model]
        ToolExecutor[Tool Executor]
        ResponseFormatter[Response Formatter]
    end

    subgraph Output
        TextResponse[Text Response]
        ToolCalls[Tool Calls]
        MemoryUpdates[Memory Updates]
    end

    UserInput --> PromptBuilder
    SessionContext --> PromptBuilder
    MemoryContext --> PromptBuilder
    PromptBuilder --> Model
    Model --> ToolExecutor
    ToolExecutor --> Model
    Model --> ResponseFormatter
    ResponseFormatter --> TextResponse
    ResponseFormatter --> MemoryUpdates
    ToolExecutor --> ToolCalls
```

## Agent Runtimes

OpenClaw supports multiple agent runtime implementations:

| Runtime | Type | Description |
|---------|------|-------------|
| PI | Embedded | Built-in agent with direct model access |
| Codex | External | OpenAI Codex app-server integration |
| ACP | Protocol | Agent Communication Protocol |

### PI Runtime

The embedded runtime provides direct model access:

```mermaid
flowchart TB
    A[Input] --> B[System Prompt]
    B --> C[User Context]
    C --> D[Memory Retrieval]
    D --> E[Full Prompt]
    E --> F[Model Inference]
    F --> G{Response Type}
    G -->|Text| H[Stream Response]
    G -->|Tool| I[Execute Tool]
    I --> F
    H --> J[Memory Update]
```

### Runtime Interface

```typescript
interface AgentRuntime {
  readonly id: string;
  readonly type: "pi" | "codex" | "acp";
  readonly capabilities: RuntimeCapabilities;

  // Lifecycle
  start(config: RuntimeConfig): Promise<void>;
  stop(): Promise<void>;

  // Execution
  run(params: RunParams): AsyncIterable<RunEvent>;
  abort(runId: string): Promise<void>;

  // Tools
  registerTools(tools: Tool[]): void;
  unregisterTools(toolNames: string[]): void;
}
```

## Agent Loop

### Execution Flow

```mermaid
sequenceDiagram
    participant Caller
    participant AgentLoop
    participant ContextEngine
    participant Model
    participant ToolExecutor

    Caller->>AgentLoop: run(params)
    AgentLoop->>ContextEngine: buildContext(params)
    ContextEngine-->>AgentLoop: context
    AgentLoop->>Model: infer(context)

    loop Until Done
        Model-->>AgentLoop: response
        alt Tool Call
            AgentLoop->>ToolExecutor: execute(tool, params)
            ToolExecutor-->>AgentLoop: result
            AgentLoop->>Model: continue(result)
        else Text Response
            AgentLoop-->>Caller: yield text event
        end
    end

    AgentLoop-->>Caller: completion event
```

### Run Parameters

```typescript
interface RunParams {
  sessionKey: string;
  agentId: string;
  input: string;

  // Optional
  systemPrompt?: string;
  tools?: Tool[];
  modelRef?: string;
  temperature?: number;
  maxTokens?: number;

  // Idempotency
  idemKey?: string;
}
```

### Run Events

```typescript
type RunEvent =
  | { type: "start"; runId: string }
  | { type: "assistant.delta"; delta: string }
  | { type: "assistant.text"; text: string }
  | { type: "tool_use"; tool: string; input: unknown }
  | { type: "tool_result"; tool: string; result: unknown }
  | { type: "complete"; summary?: string }
  | { type: "error"; error: string };
```

## Tool System

### Tool Definition

```typescript
interface Tool {
  readonly name: string;
  readonly description: string;
  readonly schema: JsonSchema;
  readonly category?: ToolCategory;

  execute(params: unknown, context: ToolContext): Promise<ToolResult>;
}
```

### Tool Categories

| Category | Examples | Purpose |
|----------|----------|---------|
| search | web_search, wikipedia | Information retrieval |
| compute | calculator, code_execute | Data processing |
| file | read_file, write_file | File operations |
| web | fetch_url, browser | Web interaction |
| messaging | send_message, send_email | External communication |
| system | shell, run_command | System operations |

### Tool Execution Pipeline

```mermaid
sequenceDiagram
    participant Model
    participant Executor
    participant BeforeHooks
    participant Tool
    participant AfterHooks

    Model->>Executor: tool_call(name, params)
    Executor->>BeforeHooks: before:tool
    BeforeHooks-->>Executor: proceed/modify

    alt Skip tool
        Executor-->>Model: tool_skipped
    else Execute
        Executor->>Tool: execute(params)
        Tool-->>Executor: result
        Executor->>AfterHooks: after:tool
        AfterHooks-->>Executor: proceed/modify
        Executor-->>Model: tool_result
    end
```

### Built-in Tools

OpenClaw provides built-in tools:

```typescript
const builtInTools: Tool[] = [
  {
    name: "bash",
    description: "Execute bash commands",
    schema: { command: "string" },
  },
  {
    name: "read_file",
    description: "Read file contents",
    schema: { path: "string", limit: "number?" },
  },
  {
    name: "write_file",
    description: "Write content to file",
    schema: { path: "string", content: "string" },
  },
  {
    name: "web_search",
    description: "Search the web",
    schema: { query: "string", limit: "number?" },
  },
  {
    name: "image_generation",
    description: "Generate images",
    schema: { prompt: "string", size: "string?" },
  },
];
```

## Prompt Assembly

### Context Building

```mermaid
flowchart TB
    A[System Prompt] --> D[Final Prompt]
    B[Tools Definition] --> D
    C[Session History] --> D
    E[Memory Context] --> D
    F[User Input] --> D
    D --> G[Token Budget Check]
    G --> H[Model]
```

### Token Budgeting

```typescript
interface TokenBudget {
  systemPrompt: number;
  tools: number;
  history: number;
  memory: number;
  userInput: number;
  available: number;    // maxTokens - reserved
  total: number;        // contextWindow
}
```

## Session Integration

### Session Context

Each agent run operates within a session:

```typescript
interface SessionContext {
  sessionKey: string;
  channel: string;
  peer: string;
  history: Message[];
  metadata: SessionMetadata;
  activeAgent?: string;
}
```

### Memory Integration

```typescript
interface MemoryContext {
  workingMemory: Message[];       // Current turn
  recentHistory: Message[];        // Last N messages
  shortTerm: MemoryEntry[];         // MEMORY.md, DREAMS.md
  longTerm: MemoryEntry[];         // Wiki, facts
  compacted: CompactedContext[];    // Summarized sessions
}
```

## Multi-Agent Support

### Agent Isolation

Agents can be isolated or shared:

```mermaid
flowchart LR
    A[User] --> B[Router]
    B --> C[Agent A]
    B --> D[Agent B]
    B --> E[Agent C]

    C --> F[Session A]
    D --> G[Session B]
    E --> H[Session C]
```

### Agent Selection

Agents are selected based on:

1. **Explicit routing** - Session or message metadata
2. **Capability matching** - Required capabilities
3. **Load balancing** - Across instances

## Error Handling

### Error Recovery

```typescript
interface ToolErrorHandler {
  onError(error: Error, tool: Tool): ToolErrorAction;
}

type ToolErrorAction =
  | { action: "retry"; after?: number }
  | { action: "skip"; message?: string }
  | { action: "fail"; error: string };
```

### Timeout Handling

```typescript
interface ToolConfig {
  timeout?: number;           // Max execution time (ms)
  retries?: number;            // Retry count
  retryDelay?: number;         // Delay between retries (ms)
}
```

## Monitoring

### Run Metrics

```typescript
interface RunMetrics {
  runId: string;
  startTime: Date;
  endTime?: Date;
  duration?: number;

  inputTokens: number;
  outputTokens: number;
  totalTokens: number;

  toolCalls: number;
  toolErrors: number;

  cacheHits: number;
  cacheMisses: number;
}
```

## Related

- [Session Management](/architecture-book/part-8-session-memory/01-session-management) - Session architecture
- [Memory System](/architecture-book/part-8-session-memory/00-session-memory-overview) - Memory architecture
- [Context Engine](/architecture-book/part-8-session-memory/03-context-engine) - Context assembly
- [MCP Support](/architecture-book/part-2-core-modules/06-mcp) - MCP integration