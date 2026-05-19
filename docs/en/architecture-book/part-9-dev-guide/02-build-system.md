# Build System

OpenClaw uses a modern TypeScript build pipeline built on tsdown for bundling, with pnpm for package management and workspaces.

## Build Tools

### Core Build Commands

| Command | Description |
|---------|-------------|
| `pnpm build` | Full build of all packages |
| `pnpm build:docker` | Docker-optimized production build |
| `pnpm build:strict-smoke` | Build with type declaration verification |
| `pnpm ui:build` | Build the UI components |
| `pnpm plugins:assets:build` | Build plugin assets |

### Type Checking

| Command | Description |
|---------|-------------|
| `pnpm tsgo:core` | Type check core packages with incremental build |
| `pnpm tsgo:extensions` | Type check all extensions |
| `pnpm tsgo:all` | Full type check of all projects |
| `pnpm check:test-types` | Type check test files |

## Package Structure

```
openclaw/
├── src/                    # Core application source
│   ├── agents/             # Agent implementations
│   ├── channels/           # Channel implementations
│   ├── gateway/            # Gateway server
│   ├── plugins/            # Plugin loader
│   └── cli/                # CLI implementation
├── extensions/             # Bundled plugins
│   ├── discord/            # Discord integration
│   ├── slack/              # Slack integration
│   └── ...
├── packages/               # Internal packages
│   └── plugin-sdk/         # Plugin SDK
├── ui/                     # UI components
└── dist/                   # Build output
```

## tsdown Configuration

The build uses [tsdown](https://tsdown.dev/) configured in `tsdown.config.ts`:

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

### Build Entries

Core build entries are defined in `tsdown.config.ts`:

```typescript
function buildCoreDistEntries(): Record<string, string> {
  return {
    index: "src/index.ts",
    entry: "src/entry.ts",
    "cli/daemon-cli": "src/cli/daemon-cli.ts",
    "agents/model-catalog.runtime": "src/agents/model-catalog.runtime.ts",
    // ... more entries
  };
}
```

### Bundled Dependencies

Certain dependencies are never bundled (always external):

```typescript
const explicitNeverBundleDependencies = [
  "@anthropic-ai/vertex-sdk",
  "@slack/bolt",
  "@slack/web-api",
  "@discordjs/voice",
  "@vitest/expect",
  "typescript",
  "vitest",
  // ... more
];
```

## Workspace Structure

OpenClaw uses pnpm workspaces for monorepo management:

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
  - "extensions/*"
  - "ui"
```

## Environment Variables for Build

| Variable | Description |
|----------|-------------|
| `OPENCLAW_BUILD_VERBOSE` | Enable verbose build output |
| `OUTPUT_SOURCE_MAPS` | Generate source maps |
| `OPENCLAW_BUILD_PRIVATE_QA` | Include private QA entries |
| `NODE_OPTIONS` | Node options (e.g., `--max-old-space-size=8192`) |

## Build Artifacts

### Distribution Output

The build produces:

- `dist/` - Compiled JavaScript with TypeScript declarations
- `dist/plugin-sdk/` - Plugin SDK subpath exports
- `dist/extensions/` - Bundled plugin implementations

### File Exclusions

The following are excluded from npm package:

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

## Common Build Tasks

### Build for Development

```bash
# Incremental build with watch mode
pnpm build
```

### Build for Production

```bash
# Production build with all optimizations
pnpm build:docker
```

### Rebuild After Dependency Changes

```bash
# Clean and rebuild
pnpm clean:dist
pnpm install
pnpm build
```

### Cross-Platform Builds

The build supports Docker multi-platform builds:

```bash
# Build for specific architecture
docker buildx build --platform linux/amd64,linux/arm64 .
```

## Build Performance

### Tips for Faster Builds

1. Use `pnpm build` instead of full `pnpm build:docker` for development
2. Enable incremental type checking with `pnpm tsgo:core`
3. Use `OPENCLAW_BUILD_VERBOSE=1` to identify slow builds
4. Increase Node memory for large builds:

```bash
NODE_OPTIONS=--max-old-space-size=8192 pnpm build
```

### Build Caching

- pnpm store caches dependencies at `~/.local/share/pnpm/store`
- TypeScript incremental builds use `.artifacts/tsgo-cache/`
- Vitest caches at `.vitest/cache/`

## Troubleshooting

### Build Failures

```bash
# Check for dependency issues
pnpm deps:pins:check
pnpm deps:patches:check

# Verify package structure
pnpm check:import-cycles
```

### Missing Type Declarations

```bash
# Regenerate Plugin SDK types
pnpm build:plugin-sdk:dts
pnpm build:strict-smoke
```

### Source Map Generation

For debugging builds:

```bash
OUTPUT_SOURCE_MAPS=1 pnpm build
```

## Related

- [Plugin System](/architecture-book/part-3-plugin-system/01-plugin-architecture) - Plugin development
- [Testing](/architecture-book/part-9-dev-guide/03-testing) - Test configuration