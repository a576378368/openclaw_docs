---
summary: "Vitest 测试框架、测试策略和最佳实践"
title: "测试"
read_when:
  - 编写单元和集成测试
  - 配置测试环境
---

# 测试

OpenClaw 使用 [Vitest](https://vitest.dev/) 作为测试框架，在单元、集成和端到端场景中提供全面的测试覆盖。

## 测试命令

### 基础测试命令

| 命令 | 描述 |
|------|------|
| `pnpm test` | 运行所有测试项目 |
| `pnpm test:fast` | 仅运行快速单元测试 |
| `pnpm test:unit` | 运行单元测试（快速 + 完整单元配置）|
| `pnpm test:changed` | 运行受当前更改影响的测试 |
| `pnpm test:serial` | 串行运行测试（1 个 worker）|
| `pnpm test:watch` | 监视模式用于开发 |

### 覆盖率

| 命令 | 描述 |
|------|------|
| `pnpm test:coverage` | 带覆盖率报告运行测试 |
| `pnpm test:coverage:changed` | 仅对更改文件的覆盖率 |

### 专业测试

| 命令 | 描述 |
|------|------|
| `pnpm test:extensions` | 测试所有捆绑的 Plugin |
| `pnpm test:channels` | 测试 Channel 实现 |
| `pnpm test:contracts` | 测试 Channel/Plugin 契约 |
| `pnpm test:e2e` | 端到端测试 |
| `pnpm test:ui` | UI 组件测试 |

### Docker 测试

| 命令 | 描述 |
|------|------|
| `pnpm test:docker:all` | 所有基于 Docker 的 E2E 测试 |
| `pnpm test:docker:live:all` | 实时集成测试 |
| `pnpm test:docker:local:all` | 仅本地 Docker 测试 |

### 实时测试（真实 API）

| 命令 | 描述 |
|------|------|
| `pnpm test:live` | 实时集成测试 |
| `pnpm test:live:codex-harness` | Codex harness 实时测试 |
| `pnpm test:live:media` | 媒体处理实时测试 |

## 测试配置

### Vitest 配置结构

测试使用模块化配置方法：

```
test/vitest/
├── vitest.config.ts              # 根配置
├── vitest.shared.config.ts       # 共享设置
├── vitest.unit.config.ts         # 单元测试
├── vitest.unit-fast.config.ts    # 快速单元测试
├── vitest.infra.config.ts        # 基础设施测试
├── vitest.boundary.config.ts     # 边界测试
├── vitest.gateway.config.ts      # Gateway 测试
├── vitest.extensions.config.ts   # Extension 测试
└── vitest.e2e.config.ts          # E2E 测试
```

### 共享配置

`vitest.shared.config.ts` 中的关键设置：

```typescript
export const sharedVitestConfig = {
  test: {
    testTimeout: 120_000,
    hookTimeout: isWindows ? 180_000 : 120_000,
    unstubEnvs: true,
    unstubGlobals: true,
    pool: "threads",
    maxWorkers: workerConfig.maxWorkers,
    fileParallelism: workerConfig.fileParallelism,
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov"],
      thresholds: {
        lines: 70,
        functions: 70,
        branches: 55,
        statements: 70,
      },
    },
  },
};
```

### 运行特定测试文件

```bash
# 运行特定测试文件
pnpm test src/agents/agent-command.test.ts

# 运行匹配模式的测试
pnpm test src/gateway/

# 对特定文件带覆盖率运行
OPENCLAW_VITEST_INCLUDE_FILE=src/agents/agent.test.ts pnpm test:coverage
```

## 编写测试

### 基础测试结构

```typescript
// src/agents/example.test.ts
import { describe, it, expect, vi } from "vitest";

describe("Agent 命令", () => {
  it("应该解析命令参数", () => {
    const result = parseCommand("/help arg1 arg2");
    expect(result.command).toBe("help");
    expect(result.args).toEqual(["arg1", "arg2"]);
  });

  it("应该处理空参数", () => {
    const result = parseCommand("/help");
    expect(result.command).toBe("help");
    expect(result.args).toEqual([]);
  });
});
```

### 使用 Vitest 模拟

```typescript
import { vi, describe, it, expect, beforeEach } from "vitest";

// 模拟模块
vi.mock("openclaw/plugin-sdk/runtime", () => ({
  runtime: {
    getConfig: vi.fn().mockReturnValue({}),
  },
}));

// 监视函数
const consoleSpy = vi.spyOn(console, "log").mockImplementation(() => {});

beforeEach(() => {
  consoleSpy.mockClear();
});
```

### 测试 Runtime 行为

```typescript
import { describe, it, expect } from "vitest";
import { testRuntime } from "openclaw/plugin-sdk/test-runtime";

describe("Plugin Runtime", () => {
  it("应该加载 Plugin 配置", async () => {
    const config = await testRuntime.loadPluginConfig("discord");
    expect(config).toBeDefined();
    expect(config.id).toBe("discord");
  });
});
```

## 测试环境

### 环境变量

| 变量 | 描述 |
|------|------|
| `OPENCLAW_TEST_WORKERS` | 覆盖最大测试 worker 数 |
| `OPENCLAW_VITEST_MAX_WORKERS` | 限制 worker（如 `OPENCLAW_VITEST_MAX_WORKERS=1`）|
| `OPENCLAW_VITEST_INCLUDE_FILE` | 仅运行匹配模式的文件 |
| `OPENCLAW_VITEST_EXTRA_EXCLUDE_FILE` | 排除匹配模式的文件 |
| `OPENCLAW_LIVE_TEST` | 启用实时集成测试 |
| `OPENCLAW_LIVE_TEST_QUIET` | 减少实时测试输出 |

### 测试隔离

测试默认以 `isolate: false` 运行以提高性能。当测试相互干扰时使用隔离：

```bash
# 带测试隔离运行
OPENCLAW_VITEST_ISOLATE=1 pnpm test src/agents/
```

## 覆盖率报告

使用 V8 提供程序生成覆盖率。报告输出到 `coverage/`：

```bash
# 查看文本摘要
pnpm test:coverage

# 生成 HTML 报告（打开 coverage/index.html）
open coverage/index.html
```

### 覆盖率阈值

CI 强制执行的当前最低值：

- Lines: 70%
- Functions: 70%
- Branches: 55%
- Statements: 70%

## Extension 测试

### 测试 Plugin

```bash
# 测试特定 Extension
pnpm test extensions/discord

# 测试所有捆绑的 Plugin
pnpm test:extensions

# 批量测试多个 Extension
pnpm test:extensions:batch
```

### 测试契约验证

```bash
# 验证 Channel 契约
pnpm test:contracts:channels

# 验证 Plugin 契约
pnpm test:contracts:plugins
```

## 性能测试

| 命令 | 描述 |
|------|------|
| `pnpm test:perf:budget` | 运行性能预算测试 |
| `pnpm test:perf:hotspots` | 识别测试热点 |
| `pnpm test:perf:profile:main` | 分析主线程 |
| `pnpm test:perf:profile:runner` | 分析测试运行器 |
| `pnpm test:perf:imports` | 测量导入时长 |

## 持续集成

在 CI 环境中，测试以降低的并行度运行：

```typescript
// CI 解析的 worker 数
if (isCI) {
  return {
    fileParallelism: true,
    maxWorkers: isWindows ? 2 : 3,
  };
}
```

## 故障排除

### 内存问题

```bash
# 为低内存系统减少 worker
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

### 不稳定测试

```bash
# 多次运行测试
pnpm test:serial  # 串行执行有助于识别竞态条件
```

### 超时问题

为慢 CI 增加超时：

```bash
export VITEST_TIMEOUT=300000
pnpm test
```

## 相关

- [调试](/architecture-book/part-9-dev-guide/04-debugging) - 调试技术
- [构建系统](/architecture-book/part-9-dev-guide/02-build-system) - 构建配置
