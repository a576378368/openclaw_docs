---
summary: "OpenClaw 代码库组织结构和关键目录"
title: "目录结构指南"
read_when:
  - 浏览代码库
  - 查找特定文件
---

# 目录结构指南

## 概述

OpenClaw 是一个使用 pnpm workspaces 组织的 monorepo：

```
openclaw/
├── src/              # 核心 TypeScript 源码
├── ui/               # Web UI 包
├── packages/         # SDK 包
├── extensions/       # 捆绑插件（130+）
├── docz/             # 文档
├── test/             # 测试工具和配置
└── scripts/          # 构建和实用脚本
```

## 根目录

### 包配置

| 文件 | 用途 |
|------|---------|
| `package.json` | 主包定义、脚本、依赖 |
| `pnpm-workspace.yaml` | 工作区配置 |
| `tsconfig.json` | TypeScript 基础配置 |
| `tsdown.config.ts` | 构建工具配置 |
| `vitest.config.ts` | 测试配置 |

### 关键脚本

```bash
# 开发
pnpm dev              # 启动开发模式
pnpm gateway:watch    # 网关监视模式

# 构建
pnpm build            # 构建所有包
pnpm build:ui         # 仅构建 UI

# 测试
pnpm test             # 运行测试
pnpm test:changed     # 仅运行变更的测试
pnpm test:coverage    # 带覆盖率运行

# 质量检查
pnpm check:changed    # 类型检查变更的文件
pnpm check            # 完整类型检查
pnpm lint             # 运行 linter

# 文档
pnpm docs:dev         # 启动文档开发服务器
pnpm docs:check-links # 检查文档链接
```

## src/ - 核心源码

`src/` 目录包含 OpenClaw 核心实现：

```
src/
├── index.ts              # 主入口点
├── cli/                  # CLI 实现
│   ├── run-main.ts      # 主 CLI 逻辑
│   └── commands/        # CLI 命令
│
├── gateway/              # 网关核心
│   ├── index.ts         # 网关主入口
│   ├── contracts.ts      # 能力契约
│   ├── capabilities.ts  # 能力系统
│   ├── lifecycle.ts     # 生命周期管理
│   ├── live.ts          # 实时处理
│   ├── send.ts          # 消息发送
│   ├── receive.ts       # 消息接收
│   ├── receipt.ts       # 回执处理
│   ├── state.ts         # 状态管理
│   ├── types.ts         # 类型定义
│   │
│   └── protocol/        # 线路协议
│       ├── index.ts     # 协议定义
│       └── types.ts     # 协议类型
│
├── agents/               # Agent 系统
│   ├── agent.ts         # Agent 实现
│   ├── agent-loop.ts    # Agent 执行循环
│   ├── agent-command.ts # 命令处理
│   ├── agent-scope.ts   # 作用域管理
│   ├── acp-spawn.ts     # ACP 生成
│   │
│   └── runtimes/       # 运行时实现
│       ├── pi/         # PI 运行时
│       ├── codex/      # Codex 运行时
│       └── acp/        # ACP 运行时
│
├── channels/             # Channel 抽象
│   ├── index.ts         # 主导出
│   ├── plugins/         # Channel 插件
│   ├── transport/       # 传输层
│   ├── message/        # 消息处理
│   ├── inbound-event/  # 入站事件
│   ├── turn/          # 轮次管理
│   └── status/         # 状态管理
│
├── plugins/              # 插件系统
│   ├── index.ts         # 主导出
│   ├── types.ts         # 插件类型
│   ├── registry.ts      # 插件注册表
│   ├── loader.ts        # 插件加载器
│   ├── discover.ts      # 发现逻辑
│   │
│   ├── contracts/       # 插件契约
│   │   ├── registry.ts  # 注册表契约
│   │   └── types.ts     # 契约类型
│   │
│   └── runtime/         # 插件运行时
│       ├── index.ts     # 运行时导出
│       └── types.ts     # 运行时类型
│
├── provider-runtime/     # Provider 运行时
│   ├── index.ts         # 主导出
│   ├── manager.ts       # Provider 管理器
│   └── runtime.ts       # 运行时实现
│
├── sessions/             # 会话管理
│   ├── index.ts         # 主导出
│   ├── session.ts       # Session 类
│   ├── store.ts         # Session 存储
│   ├── resolver.ts      # Session 解析器
│   └── types.ts         # Session 类型
│
├── memory/               # 内存系统
│   ├── index.ts         # 主导出
│   ├── memory.ts        # Memory 类
│   ├── store.ts         # Memory 存储
│   └── types.ts         # Memory 类型
│
├── tools/                # Agent 工具
│   ├── index.ts         # 主导出
│   ├── registry.ts      # 工具注册表
│   ├── executor.ts      # 工具执行器
│   └── hooks.ts         # 工具 Hook
│
├── mcp/                  # MCP 支持
│   ├── index.ts         # 主导出
│   ├── server.ts        # MCP 服务器
│   ├── client.ts        # MCP 客户端
│   └── bridge.ts        # 工具桥接
│
├── flows/                # 工作流编排
│   ├── index.ts         # 主导出
│   ├── flow.ts          # Flow 类
│   └── executor.ts      # Flow 执行器
│
├── tasks/                # 任务管理
│   ├── index.ts         # 主导出
│   ├── ledger.ts        # 任务账本
│   └── worker.ts        # 任务 Worker
│
├── config/                # 配置系统
│   ├── index.ts         # 主导出
│   ├── schema.ts        # 配置 schema
│   ├── loader.ts        # 配置加载器
│   ├── validator.ts     # 配置验证器
│   └── bundled-channel-config-metadata.generated.ts
│
├── chat/                  # ACP 协议客户端
│   ├── index.ts         # 主导出
│   ├── client.ts        # ACP 客户端
│   └── events.ts        # 事件类型
│
├── bootstrap/             # 引导逻辑
│   └── index.ts         # 引导入口
│
├── daemon/                # Daemon 管理
│   └── index.ts         # Daemon 导出
│
├── tui/                   # 终端 UI
│   └── index.ts         # TUI 导出
│
└── plugin-sdk/            # 插件 SDK 类型
    └── index.ts         # SDK 导出
```

## ui/ - Web UI 包

基于 Web 的用户界面：

```
ui/
├── src/
│   ├── ui/              # React 组件
│   ├── i18n/            # 国际化
│   ├── styles/          # CSS/SCSS 文件
│   ├── types/           # 类型定义
│   └── test-helpers/   # 测试工具
├── package.json
└── vite.config.ts       # Vite 配置
```

## packages/ - SDK 包

发布的 SDK 包供外部使用：

### packages/sdk/

用于与 OpenClaw 交互的客户端 SDK：

```
packages/sdk/src/
├── index.ts          # 主导出
├── client.ts         # 客户端类
├── event-hub.ts      # 事件处理
├── transport.ts      # WebSocket 传输
├── types.ts          # 类型定义
├── normalize.ts      # 事件规范化
└── normalize.ts      # 网关事件规范化
```

### packages/plugin-sdk/

用于构建 OpenClaw 插件的 SDK：

```
packages/plugin-sdk/
├── src/
│   ├── index.ts       # 主导出（50+ 子路径）
│   ├── runtime/       # 运行时 API
│   ├── config/        # 配置 API
│   ├── channel/      # Channel API
│   ├── provider/      # Provider API
│   ├── testing/       # 测试工具
│   └── types/         # 类型定义
└── package.json       # 50+ 导出子路径
```

### packages/memory-host-sdk/

内存系统集成的 SDK：

```
packages/memory-host-sdk/
├── src/
│   ├── host/          # 主机接口
│   ├── engine*.ts     # 引擎实现
│   └── types.ts       # 类型定义
└── package.json
```

### packages/plugin-package-contract/

插件的契约定义：

```
packages/plugin-package-contract/
├── src/
│   ├── types.ts       # 契约类型
│   └── ...
└── package.json
```

## extensions/ - 捆绑插件

130+ 捆绑插件按类型组织：

### 按类别

```
extensions/
├── providers/          # AI Provider（30+）
│   ├── openai/
│   ├── anthropic/
│   ├── google/
│   ├── azure-openai/
│   ├── deepseek/
│   ├── ollama/
│   ├── lmstudio/
│   ├── openrouter/
│   └── ...
│
├── channels/          # 消息平台（20+）
│   ├── telegram/
│   ├── discord/
│   ├── whatsapp/
│   ├── slack/
│   ├── matrix/
│   ├── msteams/
│   ├── feishu/
│   └── ...
│
├── tools/            # 工具提供者（50+）
│   ├── browser/
│   ├── tavily/
│   ├── firecrawl/
│   ├── exa/
│   ├── brave/
│   └── ...
│
├── memory/           # 内存实现（10+）
│   ├── memory-core/
│   ├── memory-wiki/
│   ├── memory-lancedb/
│   └── ...
│
└── protocols/       # 协议实现
    ├── acpx/         # Agent Client Protocol
    └── codex/       # OpenAI Codex
```

### 插件结构

每个插件遵循标准结构：

```
extensions/telegram/
├── package.json      # 插件元数据
├── openclaw.plugin.json  # Manifest
├── src/
│   ├── index.ts     # 入口点
│   ├── channel.ts   # Channel 实现
│   ├── config.ts    # 配置 schema
│   └── ...
├── test/
│   └── *.test.ts    # 插件测试
├── AGENTS.md        # 插件特定指南
└── README.md        # 插件文档
```

## docz/ - 文档

基于 Mintlify 的文档：

```
docz/
├── docs.json            # 导航配置
├── index.md             # 首页
├── CLAUDE.md            # 文档指南
│
├── concepts/            # 概念文档
│   ├── architecture.md
│   ├── agent-loop.md
│   ├── session.md
│   ├── memory.md
│   └── ...
│
├── reference/          # 参考文档
│   ├── config.md
│   ├── plugin-sdk.md
│   └── ...
│
├── gateway/            # 网关文档
│   ├── protocol.md
│   ├── configuration.md
│   └── ...
│
├── plugins/            # 插件文档
│   ├── overview.md
│   ├── writing-plugins.md
│   └── ...
│
├── channels/           # Channel 文档
│   ├── telegram.md
│   ├── discord.md
│   └── ...
│
├── providers/         # Provider 文档
│   ├── openai.md
│   ├── anthropic.md
│   └── ...
│
├── architecture-book/ # 架构书籍
│   ├── index.md
│   ├── part-1-foundations/
│   ├── part-2-core-modules/
│   └── ...
│
└── .generated/        # 自动生成的文档
    ├── architecture.md
    ├── sdk-*.md
    └── plugin-inventory.md
```

## test/ - 测试工具

测试基础设施：

```
test/
├── vitest/
│   ├── vitest.config.ts     # 共享配置
│   └── vitest.shared.config.ts
│
├── helpers/
│   ├── agent/               # Agent 测试助手
│   ├── gateway/             # 网关测试助手
│   ├── plugin/              # 插件测试助手
│   └── ...
│
└── fixtures/               # 测试 fixtures
    ├── sessions/
    ├── configs/
    └── ...
```

## scripts/ - 实用脚本

构建和维护脚本：

```
scripts/
├── committer              # 提交助手
├── clawlog.sh             # 日志查看器
├── docs-*.mjs             # 文档脚本
├── generate-*.mjs         # 代码生成
├── crabbox-*.mjs          # Crabbox 集成
└── run-vitest.mjs        # 测试运行器
```

## 关键文件模式

### 入口点

| 路径 | 用途 |
|------|---------|
| `src/index.ts` | 主库入口 |
| `src/cli/index.ts` | CLI 入口 |
| `src/gateway/index.ts` | 网关入口 |
| `extensions/*/src/index.ts` | 插件入口 |

### 类型定义

| 模式 | 用途 |
|---------|---------|
| `src/*/types.ts` | 模块类型 |
| `src/*/runtime.ts` | 运行时类型 |
| `src/*/*.d.ts` | 环境声明 |

### 测试文件

| 模式 | 用途 |
|---------|---------|
| `*.test.ts` | 单元测试 |
| `*.e2e.test.ts` | 端到端测试 |
| `test/helpers/*` | 测试工具 |

## 查找文件

### 按组件

```bash
# 网关
src/gateway/**/*.ts

# Agent
src/agents/**/*.ts

# 插件
src/plugins/**/*.ts
extensions/*/src/**/*.ts

# Channel
src/channels/**/*.ts
extensions/channels/*/src/**/*.ts
```

### 按类型

```bash
# 入口点
**/index.ts

# 类型
**/types.ts

# 测试
**/*.test.ts
```

## 相关内容

- [核心概念](/architecture-book/part-1-foundations/03-core-concepts) - 关键抽象
- [插件系统](/architecture-book/part-3-plugin-system/01-plugin-architecture) - 插件架构
- [网关](/architecture-book/part-2-core-modules/01-gateway) - 网关实现