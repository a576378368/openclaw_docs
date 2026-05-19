# Debugging

OpenClaw provides comprehensive debugging tools for troubleshooting runtime issues, from local development to production environments.

## Logging

### Log Levels

| Level | Environment Variable | Use Case |
|-------|---------------------|----------|
| `error` | `OPENCLAW_LOG_LEVEL=error` | Critical errors only |
| `warn` | `OPENCLAW_LOG_LEVEL=warn` | Warnings and errors |
| `info` | `OPENCLAW_LOG_LEVEL=info` | Normal operation (default) |
| `debug` | `OPENCLAW_LOG_LEVEL=debug` | Detailed debugging |

### Enable Debug Logging

```bash
# Enable debug logging for all modules
OPENCLAW_LOG_LEVEL=debug pnpm dev

# Enable debug logging to file
OPENCLAW_LOG_LEVEL=debug OPENCLAW_LOG_FILE=/tmp/openclaw.log pnpm dev
```

### Gateway Logging

```bash
# Start gateway with verbose logging
OPENCLAW_LOG_LEVEL=debug pnpm openclaw gateway

# View real-time gateway logs
./scripts/clawlog.sh

# Filter logs by time range
./scripts/clawlog.sh --time 10m

# Search logs for specific text
./scripts/clawlog.sh --search "error" --time 1h
```

## Debug Commands

### Gateway Debug Mode

```bash
# Start gateway in debug mode
pnpm openclaw gateway --debug

# Attach debugger to running gateway
pnpm openclaw debug --attach

# Inspect gateway state
pnpm openclaw debug --inspect
```

### Configuration Debugging

```bash
# Validate configuration
pnpm openclaw doctor

# Export current configuration
pnpm openclaw config export > config.toml

# Check config for issues
pnpm openclaw config validate
```

## Runtime Inspection

### Gateway Status

```bash
# Check gateway status
pnpm openclaw gateway status

# View connected agents
pnpm openclaw agents list

# Check plugin status
pnpm openclaw plugins list
```

### Process Inspection

```bash
# View running processes
ps aux | grep openclaw

# Check port usage
lsof -i :18789

# View gateway memory usage
node scripts/check-cli-startup-memory.mjs
```

## Node.js Debugging

### Inspect Mode

```bash
# Start with Node inspector
node --inspect dist/index.js gateway

# Enable break on start
node --inspect-brk dist/index.js gateway
```

### Chrome DevTools

1. Start gateway with `--inspect`:
   ```bash
   node --inspect=0.0.0.0:9229 dist/index.js gateway
   ```

2. Open Chrome and navigate to `chrome://inspect`

3. Click "Open dedicated DevTools for Node"

## VS Code Debugging

Add to `.vscode/launch.json`:

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

## Common Issues

### Gateway Won't Start

```bash
# Check for port conflicts
lsof -i :18789

# Check config validity
pnpm openclaw doctor --fix

# View startup logs
pnpm openclaw gateway --verbose 2>&1 | head -100
```

### Plugin Loading Issues

```bash
# Check plugin manifest
pnpm openclaw plugins validate my-plugin

# View plugin loading logs
OPENCLAW_LOG_LEVEL=debug pnpm openclaw gateway 2>&1 | grep -i plugin
```

### Authentication Failures

```bash
# Check credentials
cat ~/.openclaw/credentials/*

# Verify auth configuration
pnpm openclaw auth validate

# Debug auth flow
OPENCLAW_LOG_LEVEL=debug pnpm openclaw gateway 2>&1 | grep -i auth
```

## Telemetry

### OpenTelemetry

Enable OpenTelemetry for distributed tracing:

```bash
# Enable OTLP export
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 pnpm dev

# Configure service name
OTEL_SERVICE_NAME=openclaw-gateway pnpm dev
```

### Prometheus Metrics

```bash
# Enable Prometheus endpoint
OPENCLAW_PROMETHEUS_ENABLED=1 pnpm dev

# Access metrics
curl http://localhost:18789/metrics
```

## Network Debugging

### WebSocket Inspection

```bash
# View WebSocket connections
./scripts/clawlog.sh --category ws

# Check protocol messages
OPENCLAW_LOG_LEVEL=debug pnpm gateway 2>&1 | grep -i ws
```

### Proxy for Debugging

```bash
# Start with proxy for HTTP inspection
pnpm proxy:run

# Capture traffic
pnpm proxy:coverage
```

## Docker Debugging

### Container Logs

```bash
# View gateway logs
docker logs openclaw-gateway

# Follow logs in real-time
docker logs -f openclaw-gateway

# View with timestamps
docker logs -t openclaw-gateway
```

### Container Inspection

```bash
# Check running containers
docker ps

# Inspect container configuration
docker inspect openclaw-gateway

# Execute shell in container
docker exec -it openclaw-gateway /bin/sh
```

## Memory Profiling

### Heap Snapshots

```bash
# Start with heap profiling
node --expose-gc --max-old-space-size=4096 dist/index.js gateway

# Take heap snapshot
# Send SIGUSR2 to process
kill -USR2 <pid>

# Or use diagnostic endpoint
curl http://localhost:18789/debug/heap
```

### Memory Leak Detection

```bash
# Run memory leak test
pnpm leak:embedded-run

# Monitor memory over time
watch -n 5 'ps aux | grep openclaw'
```

## Performance Debugging

### Startup Timing

```bash
# Benchmark startup
pnpm test:startup:bench:smoke

# Detailed startup analysis
pnpm test:startup:gateway
```

### Import Analysis

```bash
# Analyze import durations
OPENCLAW_VITEST_IMPORT_DURATIONS=1 pnpm test:perf:imports
```

## Related

- [Testing](/architecture-book/part-9-dev-guide/03-testing) - Test strategies
- [Deployment](/architecture-book/part-9-dev-guide/05-deployment) - Production deployment