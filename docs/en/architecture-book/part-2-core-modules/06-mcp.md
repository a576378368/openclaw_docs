---
summary: "Model Context Protocol integration, tool bridging, and MCP server"
title: "MCP Support"
read_when:
  - Understanding MCP integration
  - Building MCP bridges
---

# MCP Support

## Overview

OpenClaw supports the Model Context Protocol (MCP) for extending agent capabilities through standardized tool and resource interfaces.

```mermaid
flowchart TB
    subgraph OpenClaw
        Agent[Agent]
        Bridge[MCP Bridge]
        Registry[Tool Registry]
    end

    subgraph MCP["MCP Servers"]
        Server1[File System Server]
        Server2[Git Server]
        Server3[Custom Server]
    end

    Agent --> Bridge
    Bridge --> Registry
    Bridge --> Server1
    Bridge --> Server2
    Bridge --> Server3
```

## What is MCP?

MCP (Model Context Protocol) is a protocol that enables AI models to interact with external tools and resources in a standardized way:

| Aspect | Description |
|--------|-------------|
| Transport | JSON-RPC over stdio or HTTP |
| Tools | Standardized tool invocation |
| Resources | File and data access |
| Prompts | Reusable prompt templates |

## MCP Architecture

### Component Overview

```mermaid
flowchart TB
    A[Agent] --> B[MCP Bridge]
    B --> C[Tool Adapter]
    B --> D[Resource Adapter]
    B --> E[Prompt Adapter]

    C --> F[MCP Client]
    D --> F
    E --> F

    F --> G[JSON-RPC Transport]
    G --> H[MCP Server]

    H --> I[File System]
    H --> J[Git]
    H --> K[Database]
```

### Bridge Responsibilities

```typescript
interface MCPMCPBridge {
  // Server management
  registerServer(config: MCPServerConfig): Promise<void>;
  unregisterServer(name: string): Promise<void>;

  // Tool bridging
  bridgeTools(serverName: string): Promise<Tool[]>;
  invokeTool(server: string, tool: string, params: unknown): Promise<ToolResult>;

  // Resource access
  listResources(serverName: string): Promise<Resource[]>;
  readResource(server: string, uri: string): Promise<ResourceContent>;

  // Health
  getServerStatus(serverName: string): ServerStatus;
}
```

## MCP Server Configuration

### Server Definition

```typescript
interface MCPServerConfig {
  name: string;
  command: string;
  args?: string[];
  env?: Record<string, string>;
  transport?: "stdio" | "http";
  url?: string;                    // For HTTP transport
}

const config: MCPServerConfig = {
  name: "filesystem",
  command: "npx",
  args: ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
  transport: "stdio",
};
```

### Multi-Server Setup

```typescript
const mcpConfig = {
  servers: {
    filesystem: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "./workspace"],
    },
    git: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-git"],
    },
    memory: {
      command: "python",
      args: ["mcp_server.py"],
      env: {
        DB_PATH: "./memory.db",
      },
    },
  },
};
```

## Tool Bridging

### Bridge Process

```mermaid
sequenceDiagram
    participant Agent
    participant Bridge
    participant MCPServer
    participant Tool

    Agent->>Bridge: invoke tool
    Bridge->>Bridge: validate params
    Bridge->>MCPServer: jsonrpc request
    MCPServer->>Tool: execute
    Tool-->>MCPServer: result
    MCPServer-->>Bridge: jsonrpc response
    Bridge->>Bridge: transform result
    Bridge-->>Agent: ToolResult
```

### Tool Schema Mapping

MCP tools use their own schema format, bridged to OpenClaw tools:

```typescript
// MCP tool schema
interface MCPTool {
  name: string;
  description: string;
  inputSchema: object;
}

// Bridge to OpenClaw Tool
function bridgeMCPTool(mcpTool: MCPTool): Tool {
  return {
    name: `mcp_${serverName}_${mcpTool.name}`,
    description: mcpTool.description,
    schema: mcpTool.inputSchema,
    execute: async (params, context) => {
      return await mcpBridge.invokeTool(serverName, mcpTool.name, params);
    },
  };
}
```

### Tool Naming

Bridged tools are prefixed:

```
mcp_filesystem_read_file
mcp_filesystem_write_file
mcp_git_clone
mcp_git_status
```

## Resource Access

### Resource Types

```typescript
interface MCPResource {
  uri: string;
  name: string;
  description?: string;
  mimeType?: string;
}

interface MCPResourceContent {
  uri: string;
  mimeType: string;
  content: string | Uint8Array;
}
```

### Resource Bridging

```typescript
interface MCPResourceBridge {
  listResources(): Promise<MCPResource[]>;
  readResource(uri: string): Promise<MCPResourceContent>;

  // Subscribe to changes
  subscribe(uri: string): void;
  unsubscribe(uri: string): void;
}
```

## MCP Client Implementation

### Client Lifecycle

```mermaid
sequenceDiagram
    participant Bridge
    participant Client
    participant Server

    Bridge->>Client: initialize()
    Client->>Server: initialize
    Server-->>Client: initialize result
    Client-->>Bridge: ready

    loop Normal Operation
        Bridge->>Client: invoke tool
        Client->>Server: tools/call
        Server-->>Client: result
        Client-->>Bridge: result
    end

    Bridge->>Client: shutdown()
    Client->>Server: shutdown
    Server-->>Client: acknowledged
```

### Client Interface

```typescript
interface MCPClient {
  readonly serverName: string;
  readonly capabilities: ServerCapabilities;

  initialize(): Promise<void>;
  listTools(): Promise<MCPTool[]>;
  callTool(name: string, args: Record<string, unknown>): Promise<ToolResult>;
  listResources(): Promise<MCPResource[]>;
  readResource(uri: string): Promise<MCPResourceContent>;

  close(): Promise<void>;
}
```

## Error Handling

### Error Types

```typescript
interface MCPError {
  code: number;
  message: string;
  data?: unknown;
}

// Error codes
const MCP_ERRORS = {
  PARSE_ERROR: -32700,
  INVALID_REQUEST: -32600,
  METHOD_NOT_FOUND: -32601,
  INVALID_PARAMS: -32602,
  INTERNAL_ERROR: -32603,
};
```

### Error Recovery

```typescript
async function withMCPErrorHandling<T>(
  fn: () => Promise<T>
): Promise<T> {
  try {
    return await fn();
  } catch (error) {
    if (error instanceof MCPError) {
      if (error.code === MCP_ERRORS.SERVER_NOT_INITIALIZED) {
        // Re-initialize
        await client.initialize();
        return await fn();
      }
    }
    throw error;
  }
}
```

## Built-in MCP Tools

### File System Tools

```typescript
const fsTools = [
  "mcp_filesystem_read_file",
  "mcp_filesystem_write_file",
  "mcp_filesystem_list_directory",
  "mcp_filesystem_create_directory",
  "mcp_filesystem_move_file",
  "mcp_filesystem_delete_file",
  "mcp_filesystem_search_files",
];
```

### Git Tools

```typescript
const gitTools = [
  "mcp_git_status",
  "mcp_git_log",
  "mcp_git_diff",
  "mcp_git_commit",
  "mcp_git_branch",
  "mcp_git_checkout",
];
```

## Testing MCP Bridges

### Test Pattern

```typescript
describe("MCP Bridge", () => {
  let bridge: MCPMCPBridge;

  beforeEach(async () => {
    bridge = new MCPMCPBridge();
    await bridge.registerServer({
      name: "test",
      command: "echo",
      args: ["test-server"],
    });
  });

  afterEach(async () => {
    await bridge.unregisterServer("test");
  });

  it("should bridge tools", async () => {
    const tools = await bridge.bridgeTools("test");
    expect(tools.length).toBeGreaterThan(0);
  });

  it("should invoke tool", async () => {
    const result = await bridge.invokeTool("test", "echo", { msg: "hello" });
    expect(result.success).toBe(true);
  });
});
```

## Related

- [Agent Tools](/architecture-book/part-2-core-modules/05-tools) - Tool system
- [Plugin System](/architecture-book/part-3-plugin-system/01-plugin-architecture) - Plugin architecture
- [MCP Server](/reference/mcp-server) - MCP reference