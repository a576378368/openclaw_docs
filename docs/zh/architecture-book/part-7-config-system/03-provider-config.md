---
summary: "模型 Provider 配置、API 设置和模型定义"
title: "Provider 配置"
read_when:
  - 配置 AI Provider
  - 设置模型定义
  - 管理 Provider 凭据
---

# Provider 配置

## 概述

OpenClaw 的 Provider 配置系统管理 AI 模型 Provider 及其关联的模型定义。每个 Provider 可以包含多个具有特定能力、定价和兼容性设置的模型。

```mermaid
flowchart TB
    subgraph Models["ModelsConfig"]
        Providers["providers"]
        Pricing["pricing"]
    end

    subgraph Provider["ModelProviderConfig"]
        BaseUrl["baseUrl"]
        Auth["apiKey/auth"]
        APISettings["api, timeout, params"]
        LocalService["localService"]
        Models["models[]"]
    end

    subgraph Model["ModelDefinitionConfig"]
        Identity["id, name, api"]
        Capabilities["reasoning, input types"]
        Cost["cost structure"]
        Context["contextWindow, maxTokens"]
        Compat["compat settings"]
    end

    Providers --> Provider
    Provider --> Models
    Models --> Model
```

## 配置结构

### 主 Models 配置

```typescript
// src/config/types.models.ts
interface ModelsConfig {
  /** Merge or replace provider configs ("merge" is default). */
  mode?: "merge" | "replace";
  /** Map of provider id -> provider config. */
  providers?: Record<string, ModelProviderConfig>;
  /** Optional pricing configuration. */
  pricing?: ModelPricingConfig;
}
```

### Provider 配置

```typescript
interface ModelProviderConfig {
  /** Base URL for API requests. */
  baseUrl: string;
  /** API key or credential for authentication. */
  apiKey?: SecretInput;
  /** Authentication mode (api-key, aws-sdk, oauth, token). */
  auth?: ModelProviderAuthMode;
  /** Override default API type. */
  api?: ModelApi;
  /** Default context window size. */
  contextWindow?: number;
  /** Override for runtime token budgeting. */
  contextTokens?: number;
  /** Override max output tokens. */
  maxTokens?: number;
  /** Request timeout in seconds. */
  timeoutSeconds?: number;
  /** Inject numCtx for OpenAI compat. */
  injectNumCtxForOpenAICompat?: boolean;
  /** Provider-specific runtime parameters. */
  params?: Record<string, unknown>;
  /** Default agent runtime for this provider's models. */
  agentRuntime?: AgentRuntimePolicyConfig;
  /** Local service to start before calling provider. */
  localService?: ModelProviderLocalServiceConfig;
  /** Custom headers (credentials supported). */
  headers?: Record<string, SecretInput>;
  /** Auth header injection control. */
  authHeader?: boolean;
  /** Request configuration. */
  request?: ConfiguredModelProviderRequest;
  /** Array of model definitions. */
  models: ModelDefinitionConfig[];
}
```

### 本地服务配置

对于需要本地服务的 Provider（如 Ollama）：

```typescript
interface ModelProviderLocalServiceConfig {
  /** Command to execute. */
  command: string;
  /** Command arguments. */
  args?: string[];
  /** Working directory. */
  cwd?: string;
  /** Environment variables. */
  env?: Record<string, string>;
  /** Health check URL. */
  healthUrl?: string;
  /** Timeout for service readiness. */
  readyTimeoutMs?: number;
  /** Idle time before stopping service. */
  idleStopMs?: number;
}
```

## 模型定义

### 基础模型结构

```typescript
interface ModelDefinitionConfig {
  /** Unique model identifier (e.g., "claude-sonnet-4"). */
  id: string;
  /** Human-readable model name. */
  name: string;
  /** API type override. */
  api?: ModelApi;
  /** Base URL for this specific model. */
  baseUrl?: string;
  /** Whether model supports reasoning. */
  reasoning: boolean;
  /** Supported input types. */
  input: Array<"text" | "image" | "video" | "audio">;
  /** Pricing information. */
  cost: {
    input: number;           // USD per million tokens
    output: number;
    cacheRead: number;
    cacheWrite: number;
    /** Optional tiered pricing. */
    tieredPricing?: TieredPricing[];
  };
  /** Context window size in tokens. */
  contextWindow: number;
  /** Runtime token cap for compaction. */
  contextTokens?: number;
  /** Maximum output tokens. */
  maxTokens: number;
  /** Provider-specific parameters. */
  params?: Record<string, unknown>;
  /** Agent runtime override for this model. */
  agentRuntime?: AgentRuntimePolicyConfig;
  /** Custom request headers. */
  headers?: Record<string, string>;
  /** Compatibility settings. */
  compat?: ModelCompatConfig;
  /** Metadata source indicator. */
  metadataSource?: "models-add";
}
```

### 分层定价

对于基于使用量有可变定价的模型：

```typescript
interface TieredPricing {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  /** Bounded tier: [start, end). Open-ended top tier: [start]. */
  range: [number, number] | [number];
}
```

## API 类型

OpenClaw 支持多种 API 类型：

```typescript
const MODEL_APIS = [
  "openai-completions",
  "openai-responses",
  "openai-codex-responses",
  "anthropic-messages",
  "google-generative-ai",
  "github-copilot",
  "bedrock-converse-stream",
  "ollama",
  "azure-openai-responses",
] as const;

type ModelApi = (typeof MODEL_APIS)[number];
```

### API 兼容性

兼容性设置控制 Provider 特定行为：

```typescript
interface ModelCompatConfig {
  // OpenAI Completions compat
  supportsStore?: boolean;
  supportsDeveloperRole?: boolean;
  supportsReasoningEffort?: boolean;
  supportsUsageInStreaming?: boolean;
  maxTokensField?: string;
  thinkingFormat?: "deepseek" | "openrouter" | "together";

  // Anthropic compat
  supportsEagerToolInputStreaming?: boolean;
  supportsLongCacheRetention?: boolean;

  // Provider-specific
  requiresMistralToolIds?: boolean;
  requiresOpenAiAnthropicToolPayload?: boolean;
  nativeWebSearchTool?: boolean;
}
```

## 认证模式

```typescript
type ModelProviderAuthMode = "api-key" | "aws-sdk" | "oauth" | "token";
```

| 模式 | 描述 | 使用场景 |
|------|-------------|----------|
| `api-key` | 请求头中的标准 API 密钥 | OpenAI, Anthropic |
| `aws-sdk` | AWS SDK 凭据链 | Bedrock |
| `oauth` | OAuth 认证 | Google Gemini |
| `token` | 基于 Token 的认证 | 本地/自定义 Provider |

## Provider 规范化

Provider 可以定义规范化函数：

```typescript
// src/config/provider-policy.ts
interface ProviderPolicySurface {
  normalizeConfig?: (params: {
    provider: string;
    providerConfig: ModelProviderConfig;
  }) => ModelProviderConfig;
  applyConfigDefaults?: (params: {
    provider: string;
    config: OpenClawConfig;
    env: NodeJS.ProcessEnv;
  }) => OpenClawConfig;
}
```

### 规范化流程

```mermaid
flowchart TB
    A[Raw Config] --> B[Load Provider Policy]
    B --> C{Policy Exists?}
    C -->|Yes| D[Apply Normalization]
    C -->|No| E[Use Raw Config]
    D --> F[Apply Config Defaults]
    E --> F
    F --> G[Runtime Config]
```

## 密钥管理

Provider 凭据支持密钥引用：

```typescript
interface SecretInput {
  /** Direct secret value. */
  value?: string;
  /** Reference to credential file. */
  file?: string;
  /** Reference to environment variable. */
  env?: string;
}
```

### 密钥解析

```typescript
// Credentials stored in ~/.openclaw/credentials/
// Provider config references secrets by key
{
  "providers": {
    "openai": {
      "apiKey": {
        "file": "~/.openclaw/credentials/openai.key"
      }
    }
  }
}
```

## 示例配置

### OpenAI Provider

```json
{
  "models": {
    "providers": {
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "apiKey": { "env": "OPENAI_API_KEY" },
        "api": "openai-responses",
        "models": [
          {
            "id": "gpt-4o",
            "name": "GPT-4o",
            "reasoning": false,
            "input": ["text", "image"],
            "cost": {
              "input": 5.00,
              "output": 15.00,
              "cacheRead": 1.25,
              "cacheWrite": 3.75
            },
            "contextWindow": 128000,
            "maxTokens": 16384
          },
          {
            "id": "o3",
            "name": "o3",
            "reasoning": true,
            "input": ["text"],
            "cost": {
              "input": 15.00,
              "output": 60.00,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 200000,
            "maxTokens": 100000
          }
        ]
      }
    }
  }
}
```

### 带本地服务的 Ollama

```json
{
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://localhost:11434/v1",
        "apiKey": "ollama",
        "localService": {
          "command": "ollama",
          "args": ["serve"],
          "healthUrl": "http://localhost:11434/api/tags",
          "readyTimeoutMs": 30000,
          "idleStopMs": 300000
        },
        "models": [
          {
            "id": "llama3.1:8b",
            "name": "Llama 3.1 8B",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 8192,
            "maxTokens": 4096
          }
        ]
      }
    }
  }
}
```

## 相关内容

- [配置 Schema](./01-config-schema) - Schema 架构
- [Agent 配置](./05-agent-config) - Agent 配置
- [认证和安全](../part-4-gateway-protocol/05-auth-security) - 认证和安全
