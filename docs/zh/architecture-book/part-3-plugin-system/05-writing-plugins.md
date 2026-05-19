---
summary: "构建 OpenClaw 插件的分步指南"
title: "编写插件"
read_when:
  - 创建新插件
  - 理解插件结构
---

# 编写插件

## 概述

本指南逐步介绍如何从头开始创建一个完整的 OpenClaw 插件。

```mermaid
flowchart LR
    A[初始化] --> B[清单]
    B --> C[入口点]
    C --> D[实现]
    D --> E[测试]
    E --> F[打包]
```

## 插件类型

根据要添加的内容选择插件类型：

| 类型 | 使用场景 | 入口函数 |
|------|----------|----------------|
| provider | AI 模型访问 | `providerEntry()` |
| channel | 消息平台 | `channelEntry()` |
| tool | 外部能力 | `toolEntry()` |
| memory | 知识存储 | `memoryEntry()` |
| runtime | Agent 执行 | `runtimeEntry()` |

## 第 1 步：项目设置

### 创建插件目录

```bash
mkdir -p extensions/my-plugin/src
cd extensions/my-plugin
```

### 初始化包

```json
{
  "name": "@openclaw/plugin-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "tsdown",
    "test": "vitest"
  },
  "dependencies": {},
  "devDependencies": {
    "typescript": "^5.0.0",
    "tsdown": "^0.11.0",
    "@openclaw/plugin-sdk": "workspace:*"
  }
}
```

### tsdown 配置

```typescript
// tsdown.config.ts
import { defineConfig } from "tsdown";

export default defineConfig({
  entry: "./src/index.ts",
  outDir: "./dist",
  dts: true,
  splitting: false,
});
```

## 第 2 步：创建清单

### 清单文件

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "type": "provider",
  "description": "自定义 AI Provider 插件",
  "entry": "./dist/index.js",
  "runtime": {
    "node": ">=18.0.0"
  },
  "providers": [
    {
      "id": "my-provider",
      "models": ["my-model-v1", "my-model-v2"]
    }
  ]
}
```

### 插件 JSON

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "type": "provider",
  "description": "自定义 AI Provider 插件"
}
```

## 第 3 步：编写入口点

### 基本入口结构

```typescript
// src/index.ts
import { providerEntry } from "@openclaw/plugin-sdk/runtime/provider";

export const entry = providerEntry({
  id: "my-provider",
  name: "My Provider",

  async listModels(config) {
    return [
      {
        ref: "my-provider:my-model-v1",
        name: "My Model V1",
        provider: "my-provider",
        maxTokens: 4096,
        supportsStreaming: true,
        supportsFunctionCalling: true,
        contextWindow: 128000,
        maxOutputTokens: 4096,
      },
      {
        ref: "my-provider:my-model-v2",
        name: "My Model V2",
        provider: "my-provider",
        maxTokens: 8192,
        supportsStreaming: true,
        supportsFunctionCalling: true,
        supportsVision: true,
        contextWindow: 256000,
        maxOutputTokens: 8192,
      },
    ];
  },

  async createCompletion(config, params) {
    const { model, messages, stream = true } = params;

    // 调用你的 API
    const response = await callMyAPI({
      model: model.replace("my-provider:", ""),
      messages,
      stream,
    });

    return response;
  },
});
```

## 第 4 步：实现功能

### 配置

```typescript
import { defineConfig } from "@openclaw/plugin-sdk/config";

// 配置 schema
export const config = defineConfig({
  apiKey: {
    type: "secret",
    required: true,
    description: "My Provider 的 API 密钥",
  },

  baseUrl: {
    type: "string",
    default: "https://api.my-provider.com",
    description: "API 基础 URL",
  },

  model: {
    type: "string",
    default: "my-model-v1",
    description: "默认使用的模型",
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

### 使用配置激活

```typescript
import { providerEntry } from "@openclaw/plugin-sdk/runtime/provider";
import { config, type MyPluginConfig } from "./config.js";

export const entry = providerEntry({
  id: "my-provider",
  name: "My Provider",
  config,  // 附加配置 schema

  async activate(context) {
    const { config, logger } = context;

    // 使用已验证的配置
    const apiKey = await context.secrets.get("MY_PROVIDER_API_KEY");

    logger.info("My Provider 插件已激活", {
      model: config.model,
      baseUrl: config.baseUrl,
    });

    // 初始化 API 客户端
    context.state.client = createClient({
      apiKey,
      baseUrl: config.baseUrl,
    });
  },

  async listModels(config) {
    // 使用配置
    return fetchModelList(config.baseUrl);
  },

  async createCompletion(config, params) {
    // 使用配置
    return this.state.client.complete({
      model: config.model,
      messages: params.messages,
      temperature: config.temperature,
      maxTokens: config.maxTokens,
      stream: params.stream,
    });
  },
});
```

### 错误处理

```typescript
import { RetryableError, ProviderError } from "@openclaw/plugin-sdk/errors";

export const entry = providerEntry({
  id: "my-provider",
  name: "My Provider",

  async createCompletion(config, params) {
    try {
      const response = await callAPI(params);

      if (response.error) {
        throw new ProviderError(response.error.message, {
          code: response.error.code,
        });
      }

      return response;
    } catch (error) {
      if (error instanceof RateLimitError) {
        throw new RetryableError("速率限制", {
          retryAfter: error.retryAfter,
        });
      }

      if (error instanceof NetworkError) {
        throw new RetryableError("网络错误", {
          retryAfter: 1000,
        });
      }

      throw error;
    }
  },
});
```

## 第 5 步：编写测试

### 测试设置

```typescript
// src/index.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { createTestPlugin, mockContext } from "@openclaw/plugin-sdk/testing";
import { entry } from "./index.js";

describe("My Provider 插件", () => {
  let plugin: TestPlugin;
  const mockCtx = mockContext({
    config: {
      apiKey: "test-key",
      baseUrl: "https://test.example.com",
      model: "test-model",
    },
  });

  beforeEach(() => {
    plugin = createTestPlugin({
      manifest: testManifest,
      entry,
    });
  });

  it("应该成功激活", async () => {
    await plugin.activate(mockCtx);
    expect(plugin.status).toBe("active");
  });

  it("应该列出模型", async () => {
    await plugin.activate(mockCtx);
    const models = await plugin.module.listModels();
    expect(models).toHaveLength(2);
    expect(models[0].ref).toBe("my-provider:my-model-v1");
  });

  it("应该创建补全", async () => {
    await plugin.activate(mockCtx);

    const mockStream = new MockStream([
      { choices: [{ delta: { content: "Hello" } }] },
      { choices: [{ delta: { content: " world" } }] },
    ]);

    vi.stubGlobal("callMyAPI", vi.fn().mockResolvedValue(mockStream));

    const result = await plugin.module.createCompletion({
      model: "my-provider:my-model-v1",
      messages: [{ role: "user", content: "Hi" }],
      stream: true,
    });

    expect(callMyAPI).toHaveBeenCalled();
  });
});
```

### 测试清单

```typescript
const testManifest = {
  id: "my-plugin",
  name: "My Plugin",
  version: "1.0.0",
  type: "provider",
  entry: "./dist/index.js",
  providers: [
    {
      id: "my-provider",
      models: ["my-model-v1", "my-model-v2"],
    },
  ],
};
```

## 第 6 步：构建和打包

### 构建

```bash
pnpm build
```

### 输出结构

```
dist/
├── index.js          # ESM 打包
├── index.d.ts        # 类型声明
└── index.d.ts.map    # 源映射
```

### 本地测试

```bash
# 本地链接插件
cd extensions/my-plugin
npm link

# 在 openclaw 根目录
npm link @openclaw/plugin-my-plugin

# 测试
pnpm openclaw status
```

## 完整示例

### 文件结构

```
extensions/my-plugin/
├── package.json
├── tsdown.config.ts
├── openclaw.plugin.json
├── src/
│   ├── index.ts
│   ├── config.ts
│   ├── client.ts
│   └── index.test.ts
└── README.md
```

### 完整源代码

```typescript
// src/index.ts
import { providerEntry } from "@openclaw/plugin-sdk/runtime/provider";
import { defineConfig } from "@openclaw/plugin-sdk/config";
import { createClient } from "./client.js";

export const config = defineConfig({
  apiKey: {
    type: "secret",
    required: true,
  },
  baseUrl: {
    type: "string",
    default: "https://api.my-provider.com",
  },
  model: {
    type: "string",
    default: "my-model-v1",
  },
});

export const entry = providerEntry({
  id: "my-provider",
  name: "My Provider",
  config,

  async activate(context) {
    const apiKey = await context.secrets.get("MY_PROVIDER_API_KEY");
    context.state.client = createClient({
      apiKey,
      baseUrl: context.config.baseUrl,
    });
    context.logger.info("My Provider 已激活");
  },

  async listModels() {
    return [
      {
        ref: "my-provider:my-model-v1",
        name: "My Model V1",
        provider: "my-provider",
        maxTokens: 4096,
        supportsStreaming: true,
        contextWindow: 128000,
        maxOutputTokens: 4096,
      },
    ];
  },

  async createCompletion(config, params) {
    const client = this.state.client;
    return client.complete({
      model: config.model,
      messages: params.messages,
      stream: params.stream ?? true,
    });
  },
});
```

## 发布

### npm 发布

```bash
# 登录
npm login

# 发布
npm publish --access public
```

### 版本管理

```bash
# 更新版本
npm version patch  # 1.0.0 -> 1.0.1
npm version minor  # 1.0.1 -> 1.1.0
npm version major  # 1.1.0 -> 2.0.0

# 发布
npm publish
```

## 相关

- [插件架构](/architecture-book/part-3-plugin-system/01-plugin-architecture) - 插件设计
- [插件 SDK](/architecture-book/part-3-plugin-system/02-plugin-sdk) - SDK 文档
- [插件契约](/architecture-book/part-3-plugin-system/03-plugin-contracts) - 契约系统