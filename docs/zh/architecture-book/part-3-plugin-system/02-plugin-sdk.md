---
summary: "Plugin SDK structure, exports, and usage"
title: "Plugin SDK"
read_when:
  - Using the Plugin SDK
  - Building plugin entry points
---

# Plugin SDK

## Overview

The Plugin SDK (`@openclaw/plugin-sdk`) provides the APIs and utilities for building OpenClaw plugins.

```mermaid
flowchart LR
    A[Plugin Code] --> B[@openclaw/plugin-sdk]
    B --> C[Runtime APIs]
    B --> D[Config APIs]
    B --> E[Testing]
    B --> F[Types]
```

## Package Structure

### Package Exports

The SDK exports from 50+ subpaths:

```typescript
// Core entry
import { ... } from "@openclaw/plugin-sdk";

// Runtime APIs
import { pluginRuntime, channelRuntime, providerEntry } from "@openclaw/plugin-sdk/runtime";

// Config APIs
import { configRuntime, defineConfig } from "@openclaw/plugin-sdk/config";

// Testing
import { createTestPlugin, mockContext } from "@openclaw/plugin-sdk/testing";

// Types
import type { PluginManifest, PluginContext } from "@openclaw/plugin-sdk/types";
```

### Subpath Exports

| Subpath | Description |
|---------|-------------|
| `runtime` | Plugin runtime entry points |
| `runtime/channel` | Channel plugin runtime |
| `runtime/provider` | Provider plugin runtime |
| `runtime/tool` | Tool plugin runtime |
| `config` | Configuration utilities |
| `testing` | Test utilities |
| `types` | TypeScript types |

## Entry Points

### Provider Entry

```typescript
import { providerEntry } from "@openclaw/plugin-sdk/runtime/provider";

export const entry = providerEntry({
  id: "openai",
  name: "OpenAI",
  models: ["gpt-4o", "gpt-4o-mini", "gpt-4-turbo"],

  // Required methods
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

### Channel Entry

```typescript
import { channelEntry } from "@openclaw/plugin-sdk/runtime/channel";

export const entry = channelEntry({
  id: "telegram",
  name: "Telegram",
  features: ["text", "media", "commands"],

  // Lifecycle
  async connect(config) {
    return new TelegramBot(config.token, config.options);
  },

  async disconnect(bot) {
    await bot.close();
  },

  // Messaging
  async send(target, message, bot) {
    if (message.content) {
      await bot.sendMessage(target.peer, message.content);
    }
    if (message.media) {
      await bot.sendMedia(target.peer, message.media);
    }
  },

  // Event handling
  onMessage(bot, handler) {
    bot.on("message", handler);
  },
});
```

### Tool Entry

```typescript
import { toolEntry } from "@openclaw/plugin-sdk/runtime/tool";

export const entry = toolEntry({
  id: "tavily",
  name: "Tavily Search",

  tools: [
    {
      name: "web_search",
      description: "Search the web for information",
      schema: {
        type: "object",
        properties: {
          query: { type: "string", description: "Search query" },
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

## Runtime APIs

### Plugin Context

```typescript
import { pluginContext } from "@openclaw/plugin-sdk/runtime";

export async function activate() {
  const ctx = pluginContext.get();

  // Configuration
  const apiKey = await ctx.secrets.get("API_KEY");

  // Services
  ctx.logger.info("Plugin activated");

  // APIs
  ctx.session.onMessage(myHandler);
}
```

### Session API

```typescript
interface SessionAPI {
  // Get current session
  getCurrent(): Session;

  // Session operations
  addMessage(sessionKey: string, message: Message): Promise<void>;
  getHistory(sessionKey: string, limit?: number): Promise<Message[]>;
  reset(sessionKey: string): Promise<void>;
}
```

### Tools API

```typescript
interface ToolsAPI {
  // Register tools
  register(tools: Tool[]): void;
  unregister(names: string[]): void;

  // Get tool info
  get(name: string): Tool | undefined;
  list(): Tool[];

  // Hooks
  addHook(hook: ToolHook): void;
}
```

### Memory API

```typescript
interface MemoryAPI {
  // Store
  store(entry: MemoryEntry): Promise<void>;
  update(key: string, entry: Partial<MemoryEntry>): Promise<void>;

  // Retrieve
  get(key: string): Promise<MemoryEntry | null>;
  search(query: string, options?: SearchOptions): Promise<MemoryResult[]>;

  // Context
  buildContext(prompt: string): Promise<MemoryContext>;
}
```

## Configuration APIs

### Config Definition

```typescript
import { defineConfig } from "@openclaw/plugin-sdk/config";

export const config = defineConfig({
  apiKey: {
    type: "secret",
    required: true,
    description: "API key for authentication",
  },

  model: {
    type: "string",
    default: "gpt-4o",
    description: "Default model to use",
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

### Config Schema

```typescript
// Generated Zod schema from config definition
const configSchema = config.toSchema();

// Validate configuration
const validated = configSchema.safeParse(rawConfig);

if (!validated.success) {
  throw new ConfigError(validated.error);
}
```

### Config Runtime

```typescript
import { configRuntime } from "@openclaw/plugin-sdk/config";

export async function activate(context: PluginContext) {
  // Access validated config
  const config = configRuntime.get();

  // Watch for config changes
  configRuntime.onChange((newConfig) => {
    context.logger.info("Config updated", newConfig);
  });
}
```

## Testing APIs

### Test Plugin

```typescript
import {
  createTestPlugin,
  mockContext,
  mockProvider,
} from "@openclaw/plugin-sdk/testing";

describe("My Plugin", () => {
  const plugin = createTestPlugin({
    manifest: testManifest,
    entry: myPluginEntry,
  });

  it("should activate", async () => {
    const ctx = mockContext();
    await plugin.activate(ctx);
    expect(plugin.status).toBe("active");
  });
});
```

### Mock Context

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

## Types

### Plugin Types

```typescript
import type {
  PluginManifest,
  PluginContext,
  PluginLifecycle,
  PluginStatus,
} from "@openclaw/plugin-sdk/types";
```

### Provider Types

```typescript
import type {
  Provider,
  Model,
  CompletionParams,
  CompletionResult,
  ModelCapabilities,
} from "@openclaw/plugin-sdk/types";
```

### Channel Types

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

### Tool Types

```typescript
import type {
  Tool,
  ToolDefinition,
  ToolResult,
  ToolContext,
  ToolContent,
} from "@openclaw/plugin-sdk/types";
```

## SDK Versioning

### Version Compatibility

```json
{
  "@openclaw/plugin-sdk": "^1.0.0",
  "peerDependencies": {
    "openclaw": ">=1.0.0"
  }
}
```

### Migration Guide

When upgrading SDK versions:

1. Check breaking changes in CHANGELOG
2. Update entry point imports if needed
3. Update type references
4. Run tests to verify compatibility

## Best Practices

### Import Organization

```typescript
// 1. External imports
import { z } from "zod";
import OpenAI from "openai";

// 2. SDK imports
import { providerEntry } from "@openclaw/plugin-sdk/runtime/provider";
import type { Model } from "@openclaw/plugin-sdk/types";

// 3. Internal imports
import { myUtils } from "./utils.js";
```

### Error Handling

```typescript
export const entry = providerEntry({
  async createCompletion(config, params) {
    try {
      // Implementation
      return await doCompletion(params);
    } catch (error) {
      if (error instanceof RateLimitError) {
        throw new RetryableError("Rate limited", { retryAfter: 5000 });
      }
      throw new ProviderError("Completion failed", { cause: error });
    }
  },
});
```

### Logging

```typescript
export async function activate(context: PluginContext) {
  const log = context.logger.child({ plugin: "my-plugin" });

  log.info("Activating plugin");
  try {
    await initialize();
    log.info("Plugin activated successfully");
  } catch (error) {
    log.error({ error }, "Failed to activate plugin");
    throw error;
  }
}
```

## Related

- [Plugin Architecture](/architecture-book/part-3-plugin-system/01-plugin-architecture) - Plugin design
- [Plugin Contracts](/architecture-book/part-3-plugin-system/03-plugin-contracts) - Contract system
- [Writing Plugins](/architecture-book/part-3-plugin-system/05-writing-plugins) - Plugin development