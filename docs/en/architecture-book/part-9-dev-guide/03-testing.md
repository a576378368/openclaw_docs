# Testing

OpenClaw uses [Vitest](https://vitest.dev/) as its testing framework, with comprehensive test coverage across unit, integration, and end-to-end scenarios.

## Test Commands

### Basic Test Commands

| Command | Description |
|---------|-------------|
| `pnpm test` | Run all test projects |
| `pnpm test:fast` | Run fast unit tests only |
| `pnpm test:unit` | Run unit tests (fast + full unit config) |
| `pnpm test:changed` | Run tests affected by current changes |
| `pnpm test:serial` | Run tests sequentially (1 worker) |
| `pnpm test:watch` | Watch mode for development |

### Coverage

| Command | Description |
|---------|-------------|
| `pnpm test:coverage` | Run tests with coverage report |
| `pnpm test:coverage:changed` | Coverage for changed files only |

### Specialized Tests

| Command | Description |
|---------|-------------|
| `pnpm test:extensions` | Test all bundled plugins |
| `pnpm test:channels` | Test channel implementations |
| `pnpm test:contracts` | Test channel/plugin contracts |
| `pnpm test:e2e` | End-to-end tests |
| `pnpm test:ui` | UI component tests |

### Docker Tests

| Command | Description |
|---------|-------------|
| `pnpm test:docker:all` | All Docker-based E2E tests |
| `pnpm test:docker:live:all` | Live integration tests |
| `pnpm test:docker:local:all` | Local Docker tests only |

### Live Tests (Real API)

| Command | Description |
|---------|-------------|
| `pnpm test:live` | Live integration tests |
| `pnpm test:live:codex-harness` | Codex harness live tests |
| `pnpm test:live:media` | Media processing live tests |

## Test Configuration

### Vitest Config Structure

Tests use a modular configuration approach:

```
test/vitest/
├── vitest.config.ts              # Root configuration
├── vitest.shared.config.ts       # Shared settings
├── vitest.unit.config.ts         # Unit tests
├── vitest.unit-fast.config.ts    # Fast unit tests
├── vitest.infra.config.ts        # Infrastructure tests
├── vitest.boundary.config.ts     # Boundary tests
├── vitest.gateway.config.ts      # Gateway tests
├── vitest.extensions.config.ts   # Extension tests
└── vitest.e2e.config.ts          # E2E tests
```

### Shared Configuration

Key settings in `vitest.shared.config.ts`:

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

### Running Specific Test Files

```bash
# Run a specific test file
pnpm test src/agents/agent-command.test.ts

# Run tests matching a pattern
pnpm test src/gateway/

# Run with coverage for specific file
OPENCLAW_VITEST_INCLUDE_FILE=src/agents/agent.test.ts pnpm test:coverage
```

## Writing Tests

### Basic Test Structure

```typescript
// src/agents/example.test.ts
import { describe, it, expect, vi } from "vitest";

describe("Agent Command", () => {
  it("should parse command arguments", () => {
    const result = parseCommand("/help arg1 arg2");
    expect(result.command).toBe("help");
    expect(result.args).toEqual(["arg1", "arg2"]);
  });

  it("should handle empty arguments", () => {
    const result = parseCommand("/help");
    expect(result.command).toBe("help");
    expect(result.args).toEqual([]);
  });
});
```

### Mocking with Vitest

```typescript
import { vi, describe, it, expect, beforeEach } from "vitest";

// Mock a module
vi.mock("openclaw/plugin-sdk/runtime", () => ({
  runtime: {
    getConfig: vi.fn().mockReturnValue({}),
  },
}));

// Spy on functions
const consoleSpy = vi.spyOn(console, "log").mockImplementation(() => {});

beforeEach(() => {
  consoleSpy.mockClear();
});
```

### Testing Runtime Behaviors

```typescript
import { describe, it, expect } from "vitest";
import { testRuntime } from "openclaw/plugin-sdk/test-runtime";

describe("Plugin Runtime", () => {
  it("should load plugin configuration", async () => {
    const config = await testRuntime.loadPluginConfig("discord");
    expect(config).toBeDefined();
    expect(config.id).toBe("discord");
  });
});
```

## Test Environment

### Environment Variables

| Variable | Description |
|----------|-------------|
| `OPENCLAW_TEST_WORKERS` | Override max test workers |
| `OPENCLAW_VITEST_MAX_WORKERS` | Limit workers (e.g., `OPENCLAW_VITEST_MAX_WORKERS=1`) |
| `OPENCLAW_VITEST_INCLUDE_FILE` | Run only files matching pattern |
| `OPENCLAW_VITEST_EXTRA_EXCLUDE_FILE` | Exclude files matching pattern |
| `OPENCLAW_LIVE_TEST` | Enable live integration tests |
| `OPENCLAW_LIVE_TEST_QUIET` | Reduce live test output |

### Test Isolation

Tests run with `isolate: false` by default for performance. Use isolation when tests interfere:

```bash
# Run with test isolation
OPENCLAW_VITEST_ISOLATE=1 pnpm test src/agents/
```

## Coverage Reports

Coverage is generated using V8 provider. Reports are output to `coverage/`:

```bash
# View text summary
pnpm test:coverage

# Generate HTML report (open coverage/index.html)
open coverage/index.html
```

### Coverage Thresholds

Current minimums enforced by CI:

- Lines: 70%
- Functions: 70%
- Branches: 55%
- Statements: 70%

## Extension Testing

### Testing Plugins

```bash
# Test a specific extension
pnpm test extensions/discord

# Test all bundled plugins
pnpm test:extensions

# Batch test multiple extensions
pnpm test:extensions:batch
```

### Test Contract Validation

```bash
# Validate channel contracts
pnpm test:contracts:channels

# Validate plugin contracts
pnpm test:contracts:plugins
```

## Performance Testing

| Command | Description |
|---------|-------------|
| `pnpm test:perf:budget` | Run performance budget tests |
| `pnpm test:perf:hotspots` | Identify test hotspots |
| `pnpm test:perf:profile:main` | Profile main thread |
| `pnpm test:perf:profile:runner` | Profile test runner |
| `pnpm test:perf:imports` | Measure import durations |

## Continuous Integration

In CI environments, tests run with reduced parallelism:

```typescript
// Resolved worker count for CI
if (isCI) {
  return {
    fileParallelism: true,
    maxWorkers: isWindows ? 2 : 3,
  };
}
```

## Troubleshooting

### Memory Issues

```bash
# Reduce workers for low-memory systems
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

### Flaky Tests

```bash
# Run test multiple times
pnpm test:serial  # Sequential execution helps identify race conditions
```

### Timeout Issues

Increase timeouts for slow CI:

```bash
export VITEST_TIMEOUT=300000
pnpm test
```

## Related

- [Debugging](/architecture-book/part-9-dev-guide/04-debugging) - Debugging techniques
- [Build System](/architecture-book/part-9-dev-guide/02-build-system) - Build configuration