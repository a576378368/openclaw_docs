---
summary: "TypeScript 构建管道、包管理和项目结构"
title: "构建系统"
read_when:
  - 理解构建流程
  - 配置构建选项
---

# 构建系统

OpenClaw 使用基于 tsdown 构建的现代 TypeScript 构建管道，使用 pnpm 进行包管理和工作区管理。

## 构建工具

### 核心构建命令

| 命令 | 描述 |
|------|------|
| `pnpm build` | 构建所有包 |
| `pnpm build:docker` | Docker 优化的生产构建 |
| `pnpm build:strict-smoke` | 带类型声明验证的构建 |
| `pnpm ui:build` | 构建 UI 组件 |
| `pnpm plugins:assets:build` | 构建 Plugin 资源 |

### 类型检查

| 命令 | 描述 |
|------|------|
| `pnpm tsgo:core` | 带增量构建的核心包类型检查 |
| `pnpm tsgo:extensions` | 检查所有 Extension |
| `pnpm tsgo:all` | 完整类型检查所有项目 |
| `pnpm check:test-types` | 检查测试文件类型 |

## 包结构

```
openclaw/
├── src/                    # 核心应用源码
│   ├── agents/             # Agent 实现
│   ├── channels/           # Channel 实现
│   ├── gateway/            # Gateway 服务器
│   ├── plugins/            # Plugin 加载器
│   └── cli/                # CLI 实现
├── extensions/             # 捆绑的 Plugin
│   ├── discord/            # Discord 集成
│   ├── slack/              # Slack 集成
│   └── ...
├── packages/               # 内部包
│   └── plugin-sdk/         # Plugin SDK
├── ui/                     # UI 组件
└── dist/                   # 构建输出
```

## tsdown 配置

构建使用 [tsdown](https://tsdown.dev/)，配置在 `tsdown.config.ts` 中：

```typescript
// tsdown.config.ts
export default defineConfig([
  nodeBuildConfig({
    clean: true,
    entry: buildUnifiedDistEntries(),
    deps: {
      alwaysBundle: shouldAlwaysBundleDependency,
      neverBundle: shouldNeverBundleDependency,
    },
  }),
]);
```

### 构建入口

核心构建入口在 `tsdown.config.ts` 中定义：

```typescript
function buildCoreDistEntries(): Record<string, string> {
  return {
    index: "src/index.ts",
    entry: "src/entry.ts",
    "cli/daemon-cli": "src/cli/daemon-cli.ts",
    "agents/model-catalog.runtime": "src/agents/model-catalog.runtime.ts",
    // ... 更多入口
  };
}
```

### 捆绑的依赖

某些依赖从不捆绑（始终外部化）：

```typescript
const explicitNeverBundleDependencies = [
  "@anthropic-ai/vertex-sdk",
  "@slack/bolt",
  "@slack/web-api",
  "@discordjs/voice",
  "@vitest/expect",
  "typescript",
  "vitest",
  // ... 更多
];
```

## 工作区结构

OpenClaw 使用 pnpm 工作区进行 monorepo 管理：

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
  - "extensions/*"
  - "ui"
```

## 构建环境变量

| 变量 | 描述 |
|------|------|
| `OPENCLAW_BUILD_VERBOSE` | 启用详细构建输出 |
| `OUTPUT_SOURCE_MAPS` | 生成 Source Map |
| `OPENCLAW_BUILD_PRIVATE_QA` | 包含私有 QA 入口 |
| `NODE_OPTIONS` | Node 选项（如 `--max-old-space-size=8192`）|

## 构建产物

### 发行版输出

构建产生：

- `dist/` - 带 TypeScript 声明的编译 JavaScript
- `dist/plugin-sdk/` - Plugin SDK 子路径导出
- `dist/extensions/` - 捆绑的 Plugin 实现

### 文件排除

以下内容从 npm 包中排除：

```json
{
  "files": [
    "!dist/.buildstamp",
    "!dist/**/*.map",
    "!dist/plugin-sdk/*.tsbuildinfo",
    "!dist/extensions/*/node_modules/**"
  ]
}
```

## 常见构建任务

### 开发构建

```bash
# 带监视模式的增量构建
pnpm build
```

### 生产构建

```bash
# 带所有优化的生产构建
pnpm build:docker
```

### 依赖更改后重建

```bash
# 清理并重建
pnpm clean:dist
pnpm install
pnpm build
```

### 跨平台构建

构建支持 Docker 多平台构建：

```bash
# 为特定架构构建
docker buildx build --platform linux/amd64,linux/arm64 .
```

## 构建性能

### 更快构建的技巧

1. 开发时使用 `pnpm build` 而不是完整的 `pnpm build:docker`
2. 使用 `pnpm tsgo:core` 启用增量类型检查
3. 使用 `OPENCLAW_BUILD_VERBOSE=1` 识别慢构建
4. 为大构建增加 Node 内存：

```bash
NODE_OPTIONS=--max-old-space-size=8192 pnpm build
```

### 构建缓存

- pnpm store 在 `~/.local/share/pnpm/store` 缓存依赖
- TypeScript 增量构建使用 `.artifacts/tsgo-cache/`
- Vitest 在 `.vitest/cache/` 缓存

## 故障排除

### 构建失败

```bash
# 检查依赖问题
pnpm deps:pins:check
pnpm deps:patches:check

# 验证包结构
pnpm check:import-cycles
```

### 缺少类型声明

```bash
# 重新生成 Plugin SDK 类型
pnpm build:plugin-sdk:dts
pnpm build:strict-smoke
```

### Source Map 生成

用于调试构建：

```bash
OUTPUT_SOURCE_MAPS=1 pnpm build
```

## 相关

- [Plugin 系统](/architecture-book/part-3-plugin-system/01-plugin-architecture) - Plugin 开发
- [测试](/architecture-book/part-9-dev-guide/03-testing) - 测试配置
