# Deployment

OpenClaw supports multiple deployment options, with Docker being the recommended approach for production environments.

## Docker Deployment

### Building the Image

```bash
# Build with default plugins
docker build -t openclaw:local .

# Build with specific plugins
docker build --build-arg OPENCLAW_EXTENSIONS="discord,slack" -t openclaw:mychannels .

# Build with additional apt packages
docker build --build-arg OPENCLAW_IMAGE_APT_PACKAGES="python3 wget" -t openclaw:custom .
```

### Docker Compose

The simplest way to run OpenClaw:

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

### Docker Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENCLAW_GATEWAY_BIND` | `lan` | Bind address (loopback/lan/all) |
| `OPENCLAW_GATEWAY_PORT` | `18789` | Gateway port |
| `OPENCLAW_STATE_DIR` | `~/.openclaw` | State directory |
| `OPENCLAW_CONFIG_PATH` | `~/.openclaw/openclaw.json` | Config file path |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | - | OpenTelemetry endpoint |
| `TZ` | `UTC` | Timezone |

### Docker Build Arguments

| Argument | Description |
|----------|-------------|
| `OPENCLAW_EXTENSIONS` | Comma-separated plugin list |
| `OPENCLAW_IMAGE_APT_PACKAGES` | Additional system packages |
| `OPENCLAW_INSTALL_BROWSER` | Set to `1` to include Chromium |
| `OPENCLAW_INSTALL_DOCKER_CLI` | Set to `1` for sandbox support |

### Runtime Options

```bash
# With custom configuration
docker run -v /path/to/config:/home/node/.openclaw/openclaw.json \
  openclaw:local node dist/index.js gateway

# With health check verification
docker run --health-cmd "node -e \"fetch('http://127.0.0.1:18789/healthz').then(r=>process.exit(r.ok?0:1))\"" \
  openclaw:local
```

## Configuration Management

### Environment-Based Configuration

```bash
# Using environment file
echo "OPENCLAW_GATEWAY_TOKEN=my-secret-token" > .env
docker-compose up

# Override specific variables
docker run -e OPENCLAW_LOG_LEVEL=debug openclaw:local
```

### Config File

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

### Secret Management

Channel credentials should be stored securely:

```bash
# Store credentials in credentials directory
~/.openclaw/credentials/
├── discord.json
├── slack.json
└── telegram.json
```

### Configuration Validation

```bash
# Validate configuration
pnpm openclaw doctor

# Check for issues
pnpm openclaw doctor --fix
```

## Health Checks

### Built-in Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/healthz` | Liveness probe |
| `/readyz` | Readiness probe |
| `/health` | Alias for healthz |
| `/ready` | Alias for readyz |

### Docker Health Check

```dockerfile
HEALTHCHECK --interval=3m --timeout=10s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:18789/healthz').then((r)=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
```

### Kubernetes Probes

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

## Monitoring

### Prometheus Metrics

Enable metrics endpoint:

```bash
docker run -e OPENCLAW_PROMETHEUS_ENABLED=1 openclaw:local
```

Access metrics at `http://localhost:18789/metrics`

### OpenTelemetry

```bash
# Configure OTLP export
docker run \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318 \
  -e OTEL_SERVICE_NAME=openclaw-gateway \
  openclaw:local
```

### Logging

```bash
# View logs
docker logs openclaw-gateway

# Structured logs
docker logs -f openclaw-gateway | jq .

# Export logs
docker logs openclaw-gateway > openclaw.log
```

## Security

### Running as Non-Root

The Docker image runs as the `node` user (UID 1000):

```dockerfile
USER node
```

### Network Security

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

### Secrets Management

```bash
# Mount secrets directory (read-only)
-v /path/to/secrets:/home/node/.config/openclaw:ro
```

## Scaling

### Multiple Replicas

For horizontal scaling, use a reverse proxy:

```yaml
# Traefik example
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

### Session Affinity

WebSocket connections require sticky sessions. Configure your load balancer accordingly.

## Backup and Recovery

### Backup State

```bash
# Backup configuration and state
tar -czf openclaw-backup.tar.gz ~/.openclaw/

# Backup specific directories
tar -czf openclaw-workspace.tar.gz ~/.openclaw/workspace/
```

### Restore

```bash
# Stop gateway
docker-compose down

# Restore from backup
tar -xzf openclaw-backup.tar.gz -C ~/

# Start gateway
docker-compose up -d
```

## Production Checklist

Before deploying to production:

- [ ] Set strong `OPENCLAW_GATEWAY_TOKEN`
- [ ] Configure `OPENCLAW_GATEWAY_BIND` appropriately
- [ ] Enable TLS termination at load balancer
- [ ] Set up monitoring and alerting
- [ ] Configure log aggregation
- [ ] Test backup and restore procedures
- [ ] Review resource limits (CPU, memory)
- [ ] Enable health checks

## Troubleshooting Deployment

### Container Won't Start

```bash
# Check logs
docker logs openclaw-gateway

# Verify volume permissions
docker exec openclaw-gateway ls -la /home/node/.openclaw

# Check entrypoint
docker run --rm openclaw:local --help
```

### Connection Refused

```bash
# Verify port binding
docker port openclaw-gateway

# Check firewall rules
sudo iptables -L -n | grep 18789

# Test locally in container
docker exec openclaw-gateway curl localhost:18789/healthz
```

### Out of Memory

```bash
# Increase memory limit
docker run --memory=2g openclaw:local

# Or in compose
deploy:
  resources:
    limits:
      memory: 2G
```

## Related

- [Debugging](/architecture-book/part-9-dev-guide/04-debugging) - Debugging techniques
- [Build System](/architecture-book/part-9-dev-guide/02-build-system) - Build configuration