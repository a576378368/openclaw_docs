---
summary: "Docker 部署、配置管理和生产最佳实践"
title: "部署"
read_when:
  - 部署到生产环境
  - 配置和监控
---

# 部署

OpenClaw 支持多种部署选项，Docker 是生产环境的推荐方法。

## Docker 部署

### 构建镜像

```bash
# 使用默认 Plugin 构建
docker build -t openclaw:local .

# 使用特定 Plugin 构建
docker build --build-arg OPENCLAW_EXTENSIONS="discord,slack" -t openclaw:mychannels .

# 带附加 apt 包构建
docker build --build-arg OPENCLAW_IMAGE_APT_PACKAGES="python3 wget" -t openclaw:custom .
```

### Docker Compose

运行 OpenClaw 的最简单方式：

```yaml
# docker-compose.yml
services:
  openclaw-gateway:
    image: openclaw:local
    environment:
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      OPENCLAW_GATEWAY_BIND: lan
    ports:
      - "18789:18789"
    volumes:
      - ~/.openclaw:/home/node/.openclaw
    restart: unless-stopped
```

### Docker 环境变量

| 变量 | 默认值 | 描述 |
|------|--------|------|
| `OPENCLAW_GATEWAY_BIND` | `lan` | 绑定地址（loopback/lan/all）|
| `OPENCLAW_GATEWAY_PORT` | `18789` | Gateway 端口 |
| `OPENCLAW_STATE_DIR` | `~/.openclaw` | 状态目录 |
| `OPENCLAW_CONFIG_PATH` | `~/.openclaw/openclaw.json` | 配置文件路径 |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | - | OpenTelemetry 端点 |
| `TZ` | `UTC` | 时区 |

### Docker 构建参数

| 参数 | 描述 |
|------|------|
| `OPENCLAW_EXTENSIONS` | 逗号分隔的 Plugin 列表 |
| `OPENCLAW_IMAGE_APT_PACKAGES` | 附加系统包 |
| `OPENCLAW_INSTALL_BROWSER` | 设为 `1` 以包含 Chromium |
| `OPENCLAW_INSTALL_DOCKER_CLI` | 设为 `1` 以支持沙箱 |

### Runtime 选项

```bash
# 使用自定义配置
docker run -v /path/to/config:/home/node/.openclaw/openclaw.json \
  openclaw:local node dist/index.js gateway

# 带健康检查验证
docker run --health-cmd "node -e \"fetch('http://127.0.0.1:18789/healthz').then(r=>process.exit(r.ok?0:1))\"" \
  openclaw:local
```

## 配置管理

### 基于环境的配置

```bash
# 使用环境文件
echo "OPENCLAW_GATEWAY_TOKEN=my-secret-token" > .env
docker-compose up

# 覆盖特定变量
docker run -e OPENCLAW_LOG_LEVEL=debug openclaw:local
```

### 配置文件

```json
// ~/.openclaw/openclaw.json
{
  "version": "1.0",
  "gateway": {
    "bind": "lan",
    "port": 18789
  },
  "plugins": {
    "enabled": ["discord", "slack"]
  }
}
```

### 密钥管理

Channel 凭证应安全存储：

```bash
# 在 credentials 目录存储凭证
~/.openclaw/credentials/
├── discord.json
├── slack.json
└── telegram.json
```

### 配置验证

```bash
# 验证配置
pnpm openclaw doctor

# 检查问题
pnpm openclaw doctor --fix
```

## 健康检查

### 内置端点

| 端点 | 用途 |
|------|------|
| `/healthz` | 存活探针 |
| `/readyz` | 就绪探针 |
| `/health` | healthz 的别名 |
| `/ready` | readyz 的别名 |

### Docker 健康检查

```dockerfile
HEALTHCHECK --interval=3m --timeout=10s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:18789/healthz').then((r)=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
```

### Kubernetes 探针

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 18789
  initialDelaySeconds: 15
  periodSeconds: 30
readinessProbe:
  httpGet:
    path: /readyz
    port: 18789
  initialDelaySeconds: 5
  periodSeconds: 10
```

## 监控

### Prometheus 指标

启用指标端点：

```bash
docker run -e OPENCLAW_PROMETHEUS_ENABLED=1 openclaw:local
```

访问 `http://localhost:18789/metrics` 的指标

### OpenTelemetry

```bash
# 配置 OTLP 导出
docker run \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318 \
  -e OTEL_SERVICE_NAME=openclaw-gateway \
  openclaw:local
```

### 日志

```bash
# 查看日志
docker logs openclaw-gateway

# 结构化日志
docker logs -f openclaw-gateway | jq .

# 导出日志
docker logs openclaw-gateway > openclaw.log
```

## 安全

### 以非 root 用户运行

Docker 镜像以 `node` 用户（UID 1000）运行：

```dockerfile
USER node
```

### 网络安全

```yaml
# docker-compose.yml
services:
  openclaw-gateway:
    cap_drop:
      - NET_RAW
      - NET_ADMIN
    security_opt:
      - no-new-privileges:true
```

### 密钥管理

```bash
# 挂载密钥目录（只读）
-v /path/to/secrets:/home/node/.config/openclaw:ro
```

## 扩展

### 多副本

水平扩展时，使用反向代理：

```yaml
# Traefik 示例
services:
  openclaw-gateway:
    image: openclaw:local
    deploy:
      replicas: 3
    networks:
      - openclaw-net

  traefik:
    image: traefik:v3.0
    command:
      - --providers.docker
      - --entrypoints.web.address=:80
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

### 会话亲和性

WebSocket 连接需要粘性会话。相应配置负载均衡器。

## 备份和恢复

### 备份状态

```bash
# 备份配置和状态
tar -czf openclaw-backup.tar.gz ~/.openclaw/

# 备份特定目录
tar -czf openclaw-workspace.tar.gz ~/.openclaw/workspace/
```

### 恢复

```bash
# 停止 Gateway
docker-compose down

# 从备份恢复
tar -xzf openclaw-backup.tar.gz -C ~/

# 启动 Gateway
docker-compose up -d
```

## 生产检查清单

部署到生产前：

- [ ] 设置强 `OPENCLAW_GATEWAY_TOKEN`
- [ ] 适当配置 `OPENCLAW_GATEWAY_BIND`
- [ ] 在负载均衡器启用 TLS 终止
- [ ] 设置监控和告警
- [ ] 配置日志聚合
- [ ] 测试备份和恢复程序
- [ ] 审查资源限制（CPU、内存）
- [ ] 启用健康检查

## 部署故障排除

### 容器无法启动

```bash
# 检查日志
docker logs openclaw-gateway

# 验证卷权限
docker exec openclaw-gateway ls -la /home/node/.openclaw

# 检查入口点
docker run --rm openclaw:local --help
```

### 连接被拒绝

```bash
# 验证端口绑定
docker port openclaw-gateway

# 检查防火墙规则
sudo iptables -L -n | grep 18789

# 在容器内本地测试
docker exec openclaw-gateway curl localhost:18789/healthz
```

### 内存不足

```bash
# 增加内存限制
docker run --memory=2g openclaw:local

# 或在 compose 中
deploy:
  resources:
    limits:
      memory: 2G
```

## 相关

- [调试](/architecture-book/part-9-dev-guide/04-debugging) - 调试技术
- [构建系统](/architecture-book/part-9-dev-guide/02-build-system) - 构建配置
