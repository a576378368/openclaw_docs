---
summary: "工具定义、执行管道和 Hook 系统"
title: "Agent 工具"
read_when:
  - 理解工具系统
  - 构建自定义工具
---

# Agent 工具

## 概述

工具扩展 Agent 的能力，使其能够与外部系统交互、执行代码和执行操作。

```mermaid
flowchart TB
    A[用户输入] --> B[Agent]
    B --> C{操作类型}
    C -->|文本| D[直接响应]
    C -->|工具调用| E[工具执行器]
    E --> F{内置工具?}
    F -->|是| G[内部处理器]
    F -->|否| H[插件工具]
    G --> I[结果]
    H --> I
    I --> B
```

## 工具接口

### 核心定义

```typescript
interface Tool {
  readonly name: string;
  readonly description: string;
  readonly schema: JsonSchema;
  readonly category?: ToolCategory;
  readonly examples?: ToolExample[];

  execute(params: unknown, context: ToolContext): Promise<ToolResult>;
}
```

### 工具 Schema

工具使用 JSON Schema 进行参数验证：

```typescript
// 示例工具 schema
const toolSchema = {
  type: "object",
  properties: {
    query: {
      type: "string",
      description: "搜索查询",
    },
    limit: {
      type: "number",
      description: "最大结果数",
      default: 10,
    },
  },
  required: ["query"],
};
```

### 工具结果

```typescript
interface ToolResult {
  success: boolean;
  content: ToolContent;
  metadata?: ToolMetadata;
}

type ToolContent =
  | { type: "text"; text: string }
  | { type: "json"; data: unknown }
  | { type: "file"; path: string }
  | { type: "image"; url: string }
  | { type: "error"; message: string };

interface ToolMetadata {
  duration?: number;
  tokens?: number;
  cacheHit?: boolean;
}
```

## 工具类别

### 类别层次

| 类别 | 描述 | 示例 |
|----------|-------------|----------|
| search | 信息检索 | web_search, wikipedia |
| compute | 数据处理 | calculator, code_execute |
| file | 文件操作 | read_file, write_file |
| web | Web 交互 | fetch_url, browser |
| messaging | 外部通信 | send_email, send_sms |
| system | 系统操作 | shell, run_command |
| media | 媒体处理 | image_gen, tts |
| database | 数据存储 | query, insert |

### 内置工具

#### 文件工具

```typescript
const fileTools: Tool[] = [
  {
    name: "read_file",
    description: "读取文件内容",
    schema: {
      properties: {
        path: { type: "string" },
        limit: { type: "number" },
        offset: { type: "number" },
      },
      required: ["path"],
    },
  },
  {
    name: "write_file",
    description: "写入文件内容",
    schema: {
      properties: {
        path: { type: "string" },
        content: { type: "string" },
      },
      required: ["path", "content"],
    },
  },
  {
    name: "list_files",
    description: "列出目录中的文件",
    schema: {
      properties: {
        path: { type: "string" },
        recursive: { type: "boolean" },
      },
    },
  },
];
```

#### Web 工具

```typescript
const webTools: Tool[] = [
  {
    name: "web_search",
    description: "网络搜索信息",
    schema: {
      properties: {
        query: { type: "string" },
        limit: { type: "number" },
      },
      required: ["query"],
    },
  },
  {
    name: "fetch_url",
    description: "从 URL 获取内容",
    schema: {
      properties: {
        url: { type: "string" },
        method: { type: "string", enum: ["GET", "POST"] },
        headers: { type: "object" },
      },
      required: ["url"],
    },
  },
];
```

#### Shell 工具

```typescript
const shellTool: Tool = {
  name: "bash",
  description: "执行 bash 命令",
  category: "system",
  schema: {
    properties: {
      command: { type: "string" },
      timeout: { type: "number" },
      cwd: { type: "string" },
    },
    required: ["command"],
  },
};
```

## 工具执行管道

### 执行流程

```mermaid
sequenceDiagram
    participant Model
    participant Executor
    participant BeforeHooks
    participant Tool
    participant AfterHooks

    Model->>Executor: tool_call(tool_name, params)
    Executor->>BeforeHooks: call_hooks("before", tool, params)
    BeforeHooks-->>Executor: proceed/modify/skip

    alt 跳过
        Executor-->>Model: tool_skipped(reason)
    else 执行
        Executor->>Tool: execute(params, context)
        alt 超时
            Tool-->>Executor: timeout_error
        else 成功
            Tool-->>Executor: result
        end
        Executor->>AfterHooks: call_hooks("after", tool, result)
        AfterHooks-->>Executor: proceed/modify
        Executor-->>Model: tool_result(content)
    end
```

### Hook 系统

```typescript
interface ToolHooks {
  before?: (
    tool: Tool,
    params: unknown
  ) => Promise<HookResult>;
  after?: (
    tool: Tool,
    result: ToolResult
  ) => Promise<HookResult>;
}

type HookResult =
  | { action: "proceed" }
  | { action: "modify"; params: unknown }
  | { action: "skip"; reason: string }
  | { action: "error"; error: string };
```

### Hook 示例

```typescript
const hooks: ToolHooks = {
  before: async (tool, params) => {
    // 记录工具调用
    logger.debug(`工具调用: ${tool.name}`, { params });
    return { action: "proceed" };
  },
  after: async (tool, result) => {
    // 清理敏感数据
    if (result.content.type === "text") {
      result.content.text = sanitizeOutput(result.content.text);
    }
    return { action: "proceed", result };
  },
};
```

## 工具注册表

### 注册表接口

```typescript
interface ToolRegistry {
  register(tool: Tool): void;
  unregister(name: string): void;
  get(name: string): Tool | undefined;
  list(category?: ToolCategory): Tool[];
  search(query: string): Tool[];
}
```

### 工具发现

```typescript
// 发现所有可用工具
const tools = await registry.list();

// 按类别搜索
const searchTools = await registry.list("search");

// 按名称搜索
const browserTool = registry.get("browser");
```

## 工具配置

### 工具设置

```typescript
interface ToolConfig {
  enabled: boolean;
  timeout?: number;
  retries?: number;
  retryDelay?: number;
  permissions?: ToolPermission[];
}

interface ToolPermission {
  type: "allow" | "deny";
  patterns?: string[];      // 文件路径、URL 等
  commands?: string[];      // Shell 命令
  domains?: string[];        // 网络请求
}
```

### 全局配置

```typescript
const config = {
  tools: {
    timeout: 30000,          // 30 秒默认超时
    retries: 3,
    retryDelay: 1000,
    bash: {
      enabled: true,
      timeout: 60000,
      permissions: {
        allow: ["/usr/bin/*", "/bin/*"],
        deny: ["rm -rf /*", "dd if=*"],
      },
    },
    web_search: {
      enabled: true,
      provider: "tavily",
    },
  },
};
```

## 插件工具

### 工具插件

工具可以由插件提供：

```typescript
// 在工具插件中
export const tools: Tool[] = [
  {
    name: "my_custom_tool",
    description: "来自插件的自定义工具",
    schema: { /* ... */ },
  },
];

export const hooks: ToolHooks = {
  before: async (tool, params) => {
    // 插件特定验证
    return { action: "proceed" };
  },
};
```

### MCP 工具桥接

来自 MCP 服务器的工具被桥接：

```typescript
interface MCPToolBridge {
  registerServer(server: MCPServer): void;
  bridgeTool(mcpTool: MCPTool): Tool;
  handleRequest(tool: string, params: unknown): Promise<ToolResult>;
}
```

## 错误处理

### 错误类型

```typescript
type ToolError =
  | { type: "validation"; message: string }
  | { type: "timeout"; limit: number }
  | { type: "permission"; required: string }
  | { type: "execution"; cause: Error }
  | { type: "not_found"; tool: string };
```

### 错误恢复

```typescript
const toolExecutor = new ToolExecutor({
  onError: (error, tool) => {
    if (error.type === "timeout") {
      return { action: "retry", after: 5000 };
    }
    if (error.type === "permission") {
      return { action: "skip", reason: "权限被拒绝" };
    }
    return { action: "error", error: error.message };
  },
});
```

## 工具测试

### 测试模式

```typescript
describe("my_tool", () => {
  const tool = createTestTool({
    name: "my_tool",
    execute: async (params) => {
      return { success: true, content: { type: "text", text: "ok" } };
    },
  });

  it("应验证参数", async () => {
    const result = await tool.execute({});
    expect(result.success).toBe(false);
    expect(result.content.type).toBe("error");
  });

  it("应使用有效参数执行", async () => {
    const result = await tool.execute({ param: "value" });
    expect(result.success).toBe(true);
  });
});
```

## 相关

- [MCP 支持](/architecture-book/part-2-core-modules/06-mcp) - MCP 集成
- [插件系统](/architecture-book/part-3-plugin-system/01-plugin-architecture) - 插件架构
- [Context Engine](/architecture-book/part-8-session-memory/03-context-engine) - 上下文组装
