---
summary: "插件 SDK 结构、导出和使用"
title: "插件 SDK"
read_when:
  - 使用插件 SDK
  - 构建插件入口点
---

# 插件 SDK

## 概述

插件 SDK（`@openclaw/plugin-sdk`）提供了构建 OpenClaw 插件的 API 和工具。

```mermaid
flowchart LR
    A[插件代码] --> B[@openclaw/plugin-sdk]
    B --> C[运行时 API]
    B --> D[配置 API]
    B --> E[测试]
    B --> F[类型]
```

## 包结构

### 包导出

SDK 从 50+ 个子路径导出：

```typescript
// 核心入口
import { ... } from "@openclaw/plugin-sdk";

// 运行时 API
import { pluginRuntime, channelRuntime, providerEntry } from "@openclaw/plugin-sdk/runtime";

// 配置 API
import { configRuntime, defineConfig } from "@openclaw/plugin-sdk/config";

// 测试
import { createTestPlugin, mockContext } from "@openclaw/plugin-sdk/testing";

// 类型
import type { PluginManifest, PluginContext } from "@openclaw/plugin-sdk/types";
```

### 子路径导出

| 子路径 | 描述 |
|---------|-------------|
| `runtime` | 插件运行时入口点 |
| `runtime/channel` | Channel 插件运行时 |
| `runtime/provider` | Provider 插件运行时 |
| `runtime/tool` | Tool 插件运行时 |
| `config` | 配置工具 |
| `testing` | 测试工具 |
| `types` | TypeScript 类型 |

## 入口点

### Provider 入口

```typescript
import { providerEntry } from "@openclaw/plugin-sdk/runtime/provider";

export const entry = providerEntry({
  id: "openai",
  name: "OpenAI",
  models: ["gpt-4o", "gpt-4o-mini", "gpt-4-turbo"],

  // 必需方法
  async listModels(config) {
    const client = new OpenAI(config.apiKey);
    const models = await client.models.list();
    return models.map(model => ({
      ref: `openai:${model.id}`,
      name: model.id,
      capabilities: extractCapabilities(model),
    }));
  },

  async createCompletion(config, params) {
    const client = new OpenAI(config.apiKey);
    return client.chat.completions.create({
      model: params.model,
      messages: params.messages,
      stream: params.stream ?? true,
    });
  },
});
```

### Channel 入口

```typescript
import { channelEntry } from "@openclaw/plugin-sdk/runtime/channel";

export const entry = channelEntry({
  id: "telegram",
  name: "Telegram",
  features: ["text", "media", "commands"],

  // 生命周期
  async connect(config) {
    return new TelegramBot(config.token, config.options);
  },

  async disconnect(bot) {
    await bot.close();
  },

  // 消息
  async send(target, message, bot) {
    if (message.content) {
      await bot.sendMessage(target.peer, message.content);
    }
    if (message.media) {
      await bot.sendMedia(target.peer, message.media);
    }
  },

  // 事件处理
  onMessage(bot, handler) {
    bot.on("message", handler);
  },
});
```

### Tool 入口

```typescript
import { toolEntry } from "@openclaw/plugin-sdk/runtime/tool";

export const entry = toolEntry({
  id: "tavily",
  name: "Tavily Search",

  tools: [
    {
      name: "web_search",
      description: "搜索网络获取信息",
      schema: {
        type: "object",
        properties: {
          query: { type: "string", description: "搜索查询" },
          limit: { type: "number", default: 10 },
        },
        required: ["query"],
      },
    },
  ],

  async execute(tool, params, context) {
    const result = await tavily.search(params.query, {
      limit: params.limit ?? 10,
    });
    return {
      success: true,
      content: { type: "text", text: JSON.stringify(result) },
    };
  },
});
```

## 运行时 API

### 插件上下文

```typescript
import { pluginContext } from "@openclaw/plugin-sdk/runtime";

export async function activate() {
  const ctx = pluginContext.get();

  // 配置
  const apiKey = await ctx.secrets.get("API_KEY");

  // 服务
  ctx.logger.info("插件已激活");

  // API
  ctx.session.onMessage(myHandler);
}
```

### Session API

```typescript
interface SessionAPI {
  // 获取当前会话
  getCurrent(): Session;

  // 会话操作
  addMessage(sessionKey: string, message: Message): Promise<void>;
  getHistory(sessionKey: string, limit?: number): Promise<Message[]>;
  reset(sessionKey: string): Promise<void>;
}
```

### Tools API

```typescript
interface ToolsAPI {
  // 注册工具
  register(tools: Tool[]): void;
  unregister(names: string[]): void;

  // 获取工具信息
  get(name: string): Tool | undefined;
  list(): Tool[];

  // 钩子
  addHook(hook: ToolHook): void;
}
```

### Memory API

```typescript
interface MemoryAPI {
  // 存储
  store(entry: MemoryEntry): Promise<void>;
  update(key: string, entry: Partial<MemoryEntry>): Promise<void>;

  // 检索
  get(key: string): Promise<MemoryEntry | null>;
  search(query: string, options?: SearchOptions): Promise<MemoryResult[]>;

  // 上下文
  buildContext(prompt: string): Promise<MemoryContext>;
}
```

## 配置 API

### 配置定义

```typescript
import { defineConfig } from "@openclaw/plugin-sdk/config";

export const config = defineConfig({
  apiKey: {
    type: "secret",
    required: true,
    description: "用于认证的 API 密钥",
  },

  model: {
    type: "string",
    default: "gpt-4o",
    description: "默认使用的模型",
    enum: ["gpt-4o", "gpt-4o-mini", "gpt-4-turbo"],
  },

  temperature: {
    type: "number",
    default: 0.7,
    min: 0,
    max: 2,
  },

  maxTokens: {
    type: "number",
    default: 4096,
  },
});
```

### 配置 Schema

```typescript
// 从配置定义生成的 Zod schema
const configSchema = config.toSchema();

// 验证配置
const validated = configSchema.safeParse(rawConfig);

if (!validated.success) {
  throw new ConfigError(validated.error);
}
```

### 配置运行时

```typescript
import { configRuntime } from "@openclaw/plugin-sdk/config";

export async function activate(context: PluginContext) {
  // 访问已验证的配置
  const config = configRuntime.get();

  // 监听配置变更
  configRuntime.onChange((newConfig) => {
    context.logger.info("配置已更新", newConfig);
  });
}
```

## 测试 API

### 测试插件

```typescript
import {
  createTestPlugin,
  mockContext,
  mockProvider,
} from "@openclaw/plugin-sdk/testing";

describe("我的插件", () => {
  const plugin = createTestPlugin({
    manifest: testManifest,
    entry: myPluginEntry,
  });

  it("应该能激活", async () => {
    const ctx = mockContext();
    await plugin.activate(ctx);
    expect(plugin.status).toBe("active");
  });
});
```

### Mock 上下文

```typescript
const mockCtx = mockContext({
  config: { apiKey: "test-key" },
  secrets: {
    get: async (key) => "mock-value",
  },
  logger: {
    info: vi.fn(),
    error: vi.fn(),
  },
});
```

### Mock Provider

```typescript
const mockProv = mockProvider({
  id: "test",
  models: [{ ref: "test:model", name: "Test Model" }],
  createCompletion: vi.fn().mockResolvedValue({
    choices: [{ message: { content: "test response" } }],
  }),
});
```

## 类型

### 插件类型

```typescript
import type {
  PluginManifest,
  PluginContext,
  PluginLifecycle,
  PluginStatus,
} from "@openclaw/plugin-sdk/types";
```

### Provider 类型

```typescript
import type {
  Provider,
  Model,
  CompletionParams,
  CompletionResult,
  ModelCapabilities,
} from "@openclaw/plugin-sdk/types";
```

### Channel 类型

```typescript
import type {
  Channel,
  ChannelTarget,
  InboundMessage,
  OutboundMessage,
  MessageContent,
  MediaAttachment,
} from "@openclaw/plugin-sdk/types";
```

### Tool 类型

```typescript
import type {
  Tool,
  ToolDefinition,
  ToolResult,
  ToolContext,
  ToolContent,
} from "@openclaw/plugin-sdk/types";
```

## SDK 版本控制

### 版本兼容性

```json
{
  "@openclaw/plugin-sdk": "^1.0.0",
  "peerDependencies": {
    "openclaw": ">=1.0.0"
  }
}
```

### 迁移指南

升级 SDK 版本时：

1. 检查 CHANGELOG 中的破坏性变更
2. 必要时更新入口点导入
3. 更新类型引用
4. 运行测试以验证兼容性

## 最佳实践

### 导入组织

```typescript
// 1. 外部导入
import { z } from "zod";
import OpenAI from "openai";

// 2. SDK 导入
import { providerEntry } from "@openclaw/plugin-sdk/runtime/provider";
import type { Model } from "@openclaw/plugin-sdk/types";

// 3. 内部导入
import { myUtils } from "./utils.js";
```

### 错误处理

```typescript
export const entry = providerEntry({
  async createCompletion(config, params) {
    try {
      // 实现
      return await doCompletion(params);
    } catch (error) {
      if (error instanceof RateLimitError) {
        throw new RetryableError("速率限制", { retryAfter: 5000 });
      }
      throw new ProviderError("补全失败", { cause: error });
    }
  },
});
```

### 日志记录

```typescript
export async function activate(context: PluginContext) {
  const log = context.logger.child({ plugin: "my-plugin" });

  log.info("正在激活插件");
  try {
    await initialize();
    log.info("插件激活成功");
  } catch (error) {
    log.error({ error }, "插件激活失败");
    throw error;
  }
}
```

## 相关

- [插件架构](/architecture-book/part-3-plugin-system/01-plugin-architecture) - 插件设计
- [插件契约](/architecture-book/part-3-plugin-system/03-plugin-contracts) - 契约系统
- [编写插件](/architecture-book/part-3-plugin-system/05-writing-plugins) - 插件开发