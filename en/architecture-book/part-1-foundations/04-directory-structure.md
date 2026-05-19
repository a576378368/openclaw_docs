---
summary: "OpenClaw codebase organization and key directories"
title: "Directory Structure Guide"
read_when:
  - Navigating the codebase
  - Finding specific files
---

# Directory Structure Guide

## Overview

OpenClaw is a monorepo organized using pnpm workspaces:

```
openclaw/
├── src/              # Core TypeScript source
├── ui/               # Web UI package
├── packages/         # SDK packages
├── extensions/       # Bundled plugins (130+)
├── docz/             # Documentation
├── test/             # Test utilities and configs
└── scripts/          # Build and utility scripts
```

## Root Level

### Package Configuration

| File | Purpose |
|------|---------|
| `package.json` | Main package definition, scripts, dependencies |
| `pnpm-workspace.yaml` | Workspace configuration |
| `tsconfig.json` | TypeScript base configuration |
| `tsdown.config.ts` | Build tool configuration |
| `vitest.config.ts` | Test configuration |

### Key Scripts

```bash
# Development
pnpm dev              # Start development mode
pnpm gateway:watch    # Watch mode for gateway

# Build
pnpm build            # Build all packages
pnpm build:ui         # Build UI only

# Testing
pnpm test             # Run tests
pnpm test:changed     # Run changed tests only
pnpm test:coverage    # Run with coverage

# Quality
pnpm check:changed    # Type check changed files
pnpm check            # Full type check
pnpm lint             # Run linters

# Docs
pnpm docs:dev         # Start docs dev server
pnpm docs:check-links # Check doc links
```

## src/ - Core Source

The `src/` directory contains the core OpenClaw implementation:

```
src/
├── index.ts              # Main entry point
├── cli/                  # CLI implementation
│   ├── run-main.ts      # Main CLI logic
│   └── commands/        # CLI commands
│
├── gateway/              # Gateway core
│   ├── index.ts         # Gateway main
│   ├── contracts.ts      # Capability contracts
│   ├── capabilities.ts  # Capability system
│   ├── lifecycle.ts     # Lifecycle management
│   ├── live.ts          # Live handling
│   ├── send.ts          # Message sending
│   ├── receive.ts       # Message receiving
│   ├── receipt.ts       # Receipt handling
│   ├── state.ts         # State management
│   ├── types.ts         # Type definitions
│   │
│   └── protocol/        # Wire protocol
│       ├── index.ts     # Protocol definitions
│       └── types.ts     # Protocol types
│
├── agents/               # Agent system
│   ├── agent.ts         # Agent implementation
│   ├── agent-loop.ts    # Agent execution loop
│   ├── agent-command.ts # Command handling
│   ├── agent-scope.ts   # Scope management
│   ├── acp-spawn.ts     # ACP spawning
│   │
│   └── runtimes/       # Runtime implementations
│       ├── pi/         # PI runtime
│       ├── codex/      # Codex runtime
│       └── acp/        # ACP runtime
│
├── channels/             # Channel abstraction
│   ├── index.ts         # Main exports
│   ├── plugins/         # Channel plugins
│   ├── transport/       # Transport layer
│   ├── message/        # Message handling
│   ├── inbound-event/  # Inbound events
│   ├── turn/          # Turn management
│   └── status/         # Status management
│
├── plugins/              # Plugin system
│   ├── index.ts         # Main exports
│   ├── types.ts         # Plugin types
│   ├── registry.ts      # Plugin registry
│   ├── loader.ts        # Plugin loader
│   ├── discover.ts      # Discovery logic
│   │
│   ├── contracts/       # Plugin contracts
│   │   ├── registry.ts  # Registry contract
│   │   └── types.ts     # Contract types
│   │
│   └── runtime/         # Plugin runtime
│       ├── index.ts     # Runtime exports
│       └── types.ts     # Runtime types
│
├── provider-runtime/     # Provider runtime
│   ├── index.ts         # Main exports
│   ├── manager.ts       # Provider manager
│   └── runtime.ts       # Runtime implementation
│
├── sessions/             # Session management
│   ├── index.ts         # Main exports
│   ├── session.ts       # Session class
│   ├── store.ts         # Session store
│   ├── resolver.ts      # Session resolver
│   └── types.ts         # Session types
│
├── memory/               # Memory system
│   ├── index.ts         # Main exports
│   ├── memory.ts        # Memory class
│   ├── store.ts         # Memory store
│   └── types.ts         # Memory types
│
├── tools/                # Agent tools
│   ├── index.ts         # Main exports
│   ├── registry.ts      # Tool registry
│   ├── executor.ts      # Tool executor
│   └── hooks.ts         # Tool hooks
│
├── mcp/                  # MCP support
│   ├── index.ts         # Main exports
│   ├── server.ts        # MCP server
│   ├── client.ts        # MCP client
│   └── bridge.ts        # Tool bridging
│
├── flows/                # Workflow orchestration
│   ├── index.ts         # Main exports
│   ├── flow.ts          # Flow class
│   └── executor.ts      # Flow executor
│
├── tasks/                # Task management
│   ├── index.ts         # Main exports
│   ├── ledger.ts        # Task ledger
│   └── worker.ts        # Task worker
│
├── config/                # Configuration system
│   ├── index.ts         # Main exports
│   ├── schema.ts        # Config schema
│   ├── loader.ts        # Config loader
│   ├── validator.ts     # Config validator
│   └── bundled-channel-config-metadata.generated.ts
│
├── chat/                  # ACP protocol client
│   ├── index.ts         # Main exports
│   ├── client.ts        # ACP client
│   └── events.ts        # Event types
│
├── bootstrap/             # Bootstrap logic
│   └── index.ts         # Bootstrap entry
│
├── daemon/                # Daemon management
│   └── index.ts         # Daemon exports
│
├── tui/                   # Terminal UI
│   └── index.ts         # TUI exports
│
└── plugin-sdk/            # Plugin SDK types
    └── index.ts         # SDK exports
```

## ui/ - Web UI Package

Web-based user interface:

```
ui/
├── src/
│   ├── ui/              # React components
│   ├── i18n/            # Internationalization
│   ├── styles/          # CSS/SCSS files
│   ├── types/           # Type definitions
│   └── test-helpers/   # Test utilities
├── package.json
└── vite.config.ts       # Vite configuration
```

## packages/ - SDK Packages

Published SDK packages for external use:

### packages/sdk/

Client SDK for interacting with OpenClaw:

```
packages/sdk/src/
├── index.ts          # Main exports
├── client.ts         # Client class
├── event-hub.ts      # Event handling
├── transport.ts      # WebSocket transport
├── types.ts          # Type definitions
├── normalize.ts      # Event normalization
└── normalize.ts      # Gateway event normalization
```

### packages/plugin-sdk/

SDK for building OpenClaw plugins:

```
packages/plugin-sdk/
├── src/
│   ├── index.ts       # Main exports (50+ subpaths)
│   ├── runtime/       # Runtime APIs
│   ├── config/        # Config APIs
│   ├── channel/      # Channel APIs
│   ├── provider/      # Provider APIs
│   ├── testing/       # Testing utilities
│   └── types/         # Type definitions
└── package.json       # 50+ export subpaths
```

### packages/memory-host-sdk/

SDK for memory system integration:

```
packages/memory-host-sdk/
├── src/
│   ├── host/          # Host interface
│   ├── engine*.ts     # Engine implementations
│   └── types.ts       # Type definitions
└── package.json
```

### packages/plugin-package-contract/

Contract definitions for plugins:

```
packages/plugin-package-contract/
├── src/
│   ├── types.ts       # Contract types
│   └── ...
└── package.json
```

## extensions/ - Bundled Plugins

Over 130 bundled plugins organized by type:

### By Category

```
extensions/
├── providers/          # AI providers (30+)
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
├── channels/          # Messaging platforms (20+)
│   ├── telegram/
│   ├── discord/
│   ├── whatsapp/
│   ├── slack/
│   ├── matrix/
│   ├── msteams/
│   ├── feishu/
│   └── ...
│
├── tools/            # Tool providers (50+)
│   ├── browser/
│   ├── tavily/
│   ├── firecrawl/
│   ├── exa/
│   ├── brave/
│   └── ...
│
├── memory/           # Memory implementations (10+)
│   ├── memory-core/
│   ├── memory-wiki/
│   ├── memory-lancedb/
│   └── ...
│
└── protocols/       # Protocol implementations
    ├── acpx/         # Agent Client Protocol
    └── codex/       # OpenAI Codex
```

### Plugin Structure

Each plugin follows a standard structure:

```
extensions/telegram/
├── package.json      # Plugin metadata
├── openclaw.plugin.json  # Manifest
├── src/
│   ├── index.ts     # Entry point
│   ├── channel.ts   # Channel implementation
│   ├── config.ts    # Config schema
│   └── ...
├── test/
│   └── *.test.ts    # Plugin tests
├── AGENTS.md        # Plugin-specific guide
└── README.md        # Plugin documentation
```

## docz/ - Documentation

Mintlify-based documentation:

```
docz/
├── docs.json            # Navigation config
├── index.md             # Home page
├── CLAUDE.md            # Docs guide
│
├── concepts/            # Concept docs
│   ├── architecture.md
│   ├── agent-loop.md
│   ├── session.md
│   ├── memory.md
│   └── ...
│
├── reference/          # Reference docs
│   ├── config.md
│   ├── plugin-sdk.md
│   └── ...
│
├── gateway/            # Gateway docs
│   ├── protocol.md
│   ├── configuration.md
│   └── ...
│
├── plugins/            # Plugin docs
│   ├── overview.md
│   ├── writing-plugins.md
│   └── ...
│
├── channels/           # Channel docs
│   ├── telegram.md
│   ├── discord.md
│   └── ...
│
├── providers/         # Provider docs
│   ├── openai.md
│   ├── anthropic.md
│   └── ...
│
├── architecture-book/ # Architecture textbook
│   ├── index.md
│   ├── part-1-foundations/
│   ├── part-2-core-modules/
│   └── ...
│
└── .generated/        # Auto-generated docs
    ├── architecture.md
    ├── sdk-*.md
    └── plugin-inventory.md
```

## test/ - Test Utilities

Test infrastructure:

```
test/
├── vitest/
│   ├── vitest.config.ts     # Shared config
│   └── vitest.shared.config.ts
│
├── helpers/
│   ├── agent/               # Agent test helpers
│   ├── gateway/             # Gateway test helpers
│   ├── plugin/              # Plugin test helpers
│   └── ...
│
└── fixtures/               # Test fixtures
    ├── sessions/
    ├── configs/
    └── ...
```

## scripts/ - Utility Scripts

Build and maintenance scripts:

```
scripts/
├── committer              # Commit helper
├── clawlog.sh             # Log viewer
├── docs-*.mjs             # Documentation scripts
├── generate-*.mjs         # Code generation
├── crabbox-*.mjs          # Crabbox integration
└── run-vitest.mjs        # Test runner
```

## Key File Patterns

### Entry Points

| Path | Purpose |
|------|---------|
| `src/index.ts` | Main library entry |
| `src/cli/index.ts` | CLI entry |
| `src/gateway/index.ts` | Gateway entry |
| `extensions/*/src/index.ts` | Plugin entry |

### Type Definitions

| Pattern | Purpose |
|---------|---------|
| `src/*/types.ts` | Module types |
| `src/*/runtime.ts` | Runtime types |
| `src/*/*.d.ts` | Ambient declarations |

### Test Files

| Pattern | Purpose |
|---------|---------|
| `*.test.ts` | Unit tests |
| `*.e2e.test.ts` | E2E tests |
| `test/helpers/*` | Test utilities |

## Finding Files

### By Component

```bash
# Gateway
src/gateway/**/*.ts

# Agents
src/agents/**/*.ts

# Plugins
src/plugins/**/*.ts
extensions/*/src/**/*.ts

# Channels
src/channels/**/*.ts
extensions/channels/*/src/**/*.ts
```

### By Type

```bash
# Entry points
**/index.ts

# Types
**/types.ts

# Tests
**/*.test.ts
```

## Related

- [Core Concepts](/architecture-book/part-1-foundations/03-core-concepts) - Key abstractions
- [Plugin System](/architecture-book/part-3-plugin-system/01-plugin-architecture) - Plugin architecture
- [Gateway](/architecture-book/part-2-core-modules/01-gateway) - Gateway implementation