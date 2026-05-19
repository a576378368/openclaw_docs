---
summary: "日志记录、调试工具和生产问题排查"
title: "调试"
read_when:
  - 调试本地开发问题
  - 生产环境故障排除
---

# 调试

OpenClaw 为从本地开发到生产环境的运行时问题提供了全面的调试工具。

## 日志记录

### 日志级别

| 级别 | 环境变量 | 使用场景 |
|------|---------|---------|
| `error` | `OPENCLAW_LOG_LEVEL=error` | 仅关键错误 |
| `warn` | `OPENCLAW_LOG_LEVEL=warn` | 警告和错误 |
| `info` | `OPENCLAW_LOG_LEVEL=info` | 正常运行（默认）|
| `debug` | `OPENCLAW_LOG_LEVEL=debug` | 详细调试 |

### 启用调试日志

```bash
# 为所有模块启用调试日志
OPENCLAW_LOG_LEVEL=debug pnpm dev

# 启用日志到文件
OPENCLAW_LOG_LEVEL=debug OPENCLAW_LOG_FILE=/tmp/openclaw.log pnpm dev
```

### Gateway 日志

```bash
# 使用详细日志启动 Gateway
OPENCLAW_LOG_LEVEL=debug pnpm openclaw gateway

# 查看实时 Gateway 日志
./scripts/clawlog.sh

# 按时间范围过滤日志
./scripts/clawlog.sh --time 10m

# 搜索特定文本的日志
./scripts/clawlog.sh --search "error" --time 1h
```

## 调试命令

### Gateway 调试模式

```bash
# 在调试模式下启动 Gateway
pnpm openclaw gateway --debug

# 附加调试器到运行中的 Gateway
pnpm openclaw debug --attach

# 检查 Gateway 状态
pnpm openclaw debug --inspect
```

### 配置调试

```bash
# 验证配置
pnpm openclaw doctor

# 导出当前配置
pnpm openclaw config export > config.toml

# 检查配置问题
pnpm openclaw config validate
```

## Runtime 检查

### Gateway 状态

```bash
# 检查 Gateway 状态
pnpm openclaw gateway status

# 查看已连接的 Agent
pnpm openclaw agents list

# 检查 Plugin 状态
pnpm openclaw plugins list
```

### 进程检查

```bash
# 查看运行中的进程
ps aux | grep openclaw

# 检查端口使用
lsof -i :18789

# 查看 Gateway 内存使用
node scripts/check-cli-startup-memory.mjs
```

## Node.js 调试

### Inspect 模式

```bash
# 使用 Node 检查器启动
node --inspect dist/index.js gateway

# 在开始时启用断点
node --inspect-brk dist/index.js gateway
```

### Chrome DevTools

1. 使用 `--inspect` 启动 Gateway：
   ```bash
   node --inspect=0.0.0.0:9229 dist/index.js gateway
   ```

2. 打开 Chrome 并导航到 `chrome://inspect`

3. 点击 "Open dedicated DevTools for Node"

## VS Code 调试

添加到 `.vscode/launch.json`：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Gateway",
      "program": "${workspaceRoot}/dist/index.js",
      "args": ["gateway"],
      "runtimeArgs": ["--nolazy"],
      "env": {
        "OPENCLAW_LOG_LEVEL": "debug"
      },
      "sourceMaps": true,
      "outFiles": ["${workspaceRoot}/dist/**/*.js"]
    }
  ]
}
```

## 常见问题

### Gateway 无法启动

```bash
# 检查端口冲突
lsof -i :18789

# 检查配置有效性
pnpm openclaw doctor --fix

# 查看启动日志
pnpm openclaw gateway --verbose 2>&1 | head -100
```

### Plugin 加载问题

```bash
# 检查 Plugin 清单
pnpm openclaw plugins validate my-plugin

# 查看 Plugin 加载日志
OPENCLAW_LOG_LEVEL=debug pnpm openclaw gateway 2>&1 | grep -i plugin
```

### 认证失败

```bash
# 检查凭证
cat ~/.openclaw/credentials/*

# 验证认证配置
pnpm openclaw auth validate

# 调试认证流程
OPENCLAW_LOG_LEVEL=debug pnpm openclaw gateway 2>&1 | grep -i auth
```

## 遥测

### OpenTelemetry

启用分布式追踪：

```bash
# 启用 OTLP 导出
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 pnpm dev

# 配置服务名称
OTEL_SERVICE_NAME=openclaw-gateway pnpm dev
```

### Prometheus 指标

```bash
# 启用 Prometheus 端点
OPENCLAW_PROMETHEUS_ENABLED=1 pnpm dev

# 访问指标
curl http://localhost:18789/metrics
```

## 网络调试

### WebSocket 检查

```bash
# 查看 WebSocket 连接
./scripts/clawlog.sh --category ws

# 检查协议消息
OPENCLAW_LOG_LEVEL=debug pnpm gateway 2>&1 | grep -i ws
```

### 代理调试

```bash
# 使用代理启动以进行 HTTP 检查
pnpm proxy:run

# 捕获流量
pnpm proxy:coverage
```

## Docker 调试

### 容器日志

```bash
# 查看 Gateway 日志
docker logs openclaw-gateway

# 实时跟踪日志
docker logs -f openclaw-gateway

# 带时间戳查看
docker logs -t openclaw-gateway
```

### 容器检查

```bash
# 检查运行中的容器
docker ps

# 检查容器配置
docker inspect openclaw-gateway

# 在容器中执行 shell
docker exec -it openclaw-gateway /bin/sh
```

## 内存分析

### 堆快照

```bash
# 启用堆分析启动
node --expose-gc --max-old-space-size=4096 dist/index.js gateway

# 获取堆快照
# 向进程发送 SIGUSR2
kill -USR2 <pid>

# 或使用诊断端点
curl http://localhost:18789/debug/heap
```

### 内存泄漏检测

```bash
# 运行内存泄漏测试
pnpm leak:embedded-run

# 随时间监控内存
watch -n 5 'ps aux | grep openclaw'
```

## 性能调试

### 启动计时

```bash
# 基准测试启动
pnpm test:startup:bench:smoke

# 详细启动分析
pnpm test:startup:gateway
```

### 导入分析

```bash
# 分析导入时长
OPENCLAW_VITEST_IMPORT_DURATIONS=1 pnpm test:perf:imports
```

## 相关

- [测试](/architecture-book/part-9-dev-guide/03-testing) - 测试策略
- [部署](/architecture-book/part-9-dev-guide/05-deployment) - 生产部署
