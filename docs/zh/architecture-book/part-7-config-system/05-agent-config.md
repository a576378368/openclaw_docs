---
summary: "Agent configuration, model selection, runtime policies, and capabilities"
title: "Agent Config"
read_when:
  - Configuring AI agents
  - Setting up agent runtimes
  - Managing agent capabilities
---

# Agent Config

## Overview

OpenClaw's agent configuration system defines AI agents with their runtime policies, model selections, tools, and behavioral settings. Each agent can be customized with specific capabilities and defaults.

```mermaid
flowchart TB
    subgraph AgentsConfig
        Defaults["defaults"]
        List["list[]"]
    end

    subgraph AgentConfig
        Identity["id, name, description"]
        Runtime["agentRuntime, runtime"]
        Model["model, models"]
        Skills["skills, skillsLimits"]
        Memory["memorySearch"]
        Tools["tools"]
        Context["contextLimits, contextTokens"]
    end

    subgraph RuntimeTypes
        PI["embedded (PI)"]
        ACP["acp"]
    end

    Defaults --> AgentConfig
    List --> AgentConfig
    Runtime --> RuntimeTypes
```

## Configuration Structure

### Main Agents Config

```typescript
// src/config/types.agents.ts
interface AgentsConfig {
  /** Default settings for all agents. */
  defaults?: AgentDefaultsConfig;
  /** Array of agent configurations. */
  list?: AgentConfig[];
}
```

### Agent Configuration

```typescript
interface AgentConfig {
  /** Unique agent identifier. */
  id: string;
  /** Mark as the default agent. */
  default?: boolean;
  /** Human-readable name. */
  name?: string;
  /** Human-authored description. */
  description?: string;
  /** Workspace directory path. */
  workspace?: string;
  /** Agent directory path. */
  agentDir?: string;
  /** Full system prompt replacement. */
  systemPromptOverride?: string;
  /** Runtime policy override. */
  agentRuntime?: AgentRuntimePolicyConfig;
  /** @deprecated Use agentRuntime. */
  embeddedHarness?: AgentEmbeddedHarnessConfig;
  /** Model configuration. */
  model?: AgentModelConfig;
  /** Per-model metadata overrides. */
  models?: Record<string, AgentModelEntryConfig>;
  /** @deprecated Legacy compaction config. */
  compaction?: CompactionConfig;
  /** Default thinking level. */
  thinkingDefault?: ThinkingLevel;
  /** Default verbosity level. */
  verboseDefault?: "off" | "on" | "full";
  /** Tool progress detail mode. */
  toolProgressDetail?: ToolProgressDetailConfig;
  /** Default reasoning visibility. */
  reasoningDefault?: "on" | "off" | "stream";
  /** Default for fast mode. */
  fastModeDefault?: boolean;
  /** Bootstrap/context injection mode. */
  contextInjection?: ContextInjectionConfig;
  /** Max chars per bootstrap file. */
  bootstrapMaxChars?: number;
  /** Max chars across bootstrap files. */
  bootstrapTotalMaxChars?: number;
  /** Skills allowlist. */
  skills?: string[];
  /** Memory search configuration. */
  memorySearch?: MemorySearchConfig;
  /** Human-like delay settings. */
  humanDelay?: HumanDelayConfig;
  /** TTS overrides. */
  tts?: TtsConfig;
  /** Skills subsystem limits. */
  skillsLimits?: Pick<SkillsLimitsConfig, "maxSkillsPromptChars">;
  /** Context/token limit overrides. */
  contextLimits?: AgentContextLimitsConfig;
  /** Context window override. */
  contextTokens?: number;
  /** Heartbeat overrides. */
  heartbeat?: HeartbeatConfig;
  /** Identity configuration. */
  identity?: IdentityConfig;
  /** Group chat settings. */
  groupChat?: GroupChatConfig;
  /** Subagent configuration. */
  subagents?: SubagentConfig;
  /** Run loop retry boundaries. */
  runRetries?: RunRetriesConfig;
  /** Embedded Pi overrides. */
  embeddedPi?: EmbeddedPiConfig;
  /** Sandbox configuration. */
  sandbox?: AgentSandboxConfig;
  /** Stream parameters. */
  params?: Record<string, unknown>;
  /** Tools configuration. */
  tools?: AgentToolsConfig;
  /** Runtime descriptor. */
  runtime?: AgentRuntimeConfig;
}
```

## Agent Runtimes

### Runtime Types

```typescript
// Embedded (PI) Runtime
type EmbeddedRuntime = { type: "embedded" };

// ACP Runtime
type AcpRuntime = {
  type: "acp";
  acp?: {
    agent?: string;      // Harness adapter id (codex, claude)
    backend?: string;    // ACP backend override
    mode?: "persistent" | "oneshot";
    cwd?: string;         // Working directory
  };
};

type AgentRuntimeConfig = EmbeddedRuntime | AcpRuntime;
```

### Runtime Policy

```typescript
interface AgentRuntimePolicyConfig {
  /** Runtime type preference. */
  type?: "pi" | "codex" | "acp";
  /** Optional type override. */
  typeOverride?: "pi" | "codex" | "acp";
  /** ACP harness adapter id. */
  acpAgent?: string;
  /** ACP backend override. */
  acpBackend?: string;
  /** ACP session mode. */
  acpMode?: "persistent" | "oneshot";
  /** ACP working directory. */
  acpCwd?: string;
}
```

## Model Configuration

### Model Selection

```typescript
interface AgentModelConfig {
  /** Primary model provider. */
  provider?: string;
  /** Primary model id. */
  model?: string;
  /** Fallback model chain. */
  fallbacks?: string[];
  /** Reasoning model override. */
  reasoning?: string;
  /** Reasoning model fallbacks. */
  reasoningFallbacks?: string[];
  /** System prompt for model. */
  systemPrompt?: string;
  /** Temperature setting. */
  temperature?: number;
  /** Maximum tokens. */
  maxTokens?: number;
  /** Thinking budget (when supported). */
  thinkingBudget?: number;
}
```

### Per-Model Overrides

```typescript
interface AgentModelEntryConfig {
  /** Provider override. */
  provider?: string;
  /** Model id override. */
  model?: string;
  /** Fallback chain. */
  fallbacks?: string[];
  /** System prompt override. */
  systemPrompt?: string;
  /** Temperature override. */
  temperature?: number;
  /** Max tokens override. */
  maxTokens?: number;
}
```

## Thinking Levels

```typescript
type ThinkingLevel =
  | "off"        // No thinking
  | "minimal"    // Minimal reasoning
  | "low"        // Low reasoning effort
  | "medium"     // Medium reasoning
  | "high"       // High reasoning
  | "xhigh"      // Extra high reasoning
  | "adaptive"   // Adaptive based on query
  | "max";       // Maximum reasoning
```

## Tools Configuration

### Tool Allowlist

```typescript
interface AgentToolsConfig {
  /** Allowlist of tool names. */
  allow?: string[];
  /** Denylist of tool names. */
  deny?: string[];
  /** Also allow these tools (in addition to defaults). */
  alsoAllow?: string[];
  /** Custom tool definitions. */
  definitions?: ToolDefinition[];
}
```

### Group Tool Policies

```typescript
interface GroupToolPolicyConfig {
  /** Default policy for group messages. */
  default?: ToolPolicy;
  /** Per-sender tool policies. */
  bySender?: Record<string, ToolPolicy>;
}

type ToolPolicy = "allow" | "deny" | "ask";
```

## Memory Configuration

### Memory Search Settings

```typescript
interface MemorySearchConfig {
  /** Enable memory search. */
  enabled?: boolean;
  /** Minimum relevance score. */
  minScore?: number;
  /** Maximum results to return. */
  maxResults?: number;
  /** Recency boost factor. */
  recencyBoost?: number;
}
```

## Context Limits

### Token Budget Control

```typescript
interface AgentContextLimitsConfig {
  /** Max system prompt tokens. */
  maxSystemPromptChars?: number;
  /** Max context tokens. */
  maxContextTokens?: number;
  /** Max session history tokens. */
  maxSessionHistoryTokens?: number;
  /** Compact threshold percentage. */
  compactThresholdPct?: number;
  /** Compact minimum turns. */
  compactMinTurns?: number;
}
```

## Subagent Configuration

### Delegation Settings

```typescript
interface SubagentConfig {
  /** How strongly to delegate work. */
  delegationMode?: SubagentDelegationMode;
  /** Allowed subagent agent ids. */
  allowAgents?: string[];
  /** Default model for subagents. */
  model?: AgentModelConfig;
  /** Require explicit agentId in spawn. */
  requireAgentId?: boolean;
}

type SubagentDelegationMode =
  | "never"      // Never delegate
  | "eager"      // Always delegate
  | "reluctant"  // Delegate only when needed
  | "balanced";  // Balanced approach
```

## Identity Configuration

```typescript
interface IdentityConfig {
  /** Display name. */
  name?: string;
  /** Avatar (emoji, text, or URL). */
  avatar?: string;
  /** User persona prompt. */
  userPersona?: string;
}
```

## Agent Bindings

### Route Bindings

```typescript
// Route agent to specific channel/account
interface AgentBinding {
  type: "route" | "acp";
  agentId: string;
  comment?: string;
  match: AgentBindingMatch;
  session?: {
    dmScope?: DmScope;
  };
}

interface AgentBindingMatch {
  channel: string;
  accountId?: string;
  peer?: { kind: ChatType; id: string };
  guildId?: string;
  teamId?: string;
  roles?: string[];
}
```

## Example Configuration

### Basic Agent

```json
{
  "agents": {
    "defaults": {
      "model": {
        "provider": "anthropic",
        "model": "claude-sonnet-4",
        "fallbacks": ["claude-3-5-sonnet-20241022"]
      },
      "thinkingDefault": "medium",
      "verboseDefault": "on"
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "name": "Main Assistant",
        "description": "Primary assistant for general tasks",
        "skills": ["code", "research"],
        "memorySearch": {
          "enabled": true,
          "maxResults": 5
        }
      },
      {
        "id": "code-reviewer",
        "name": "Code Reviewer",
        "description": "Specialized in code review",
        "model": {
          "provider": "anthropic",
          "model": "claude-opus-4",
          "temperature": 0.3
        },
        "tools": {
          "allow": ["read_file", "bash", "grep"]
        },
        "contextLimits": {
          "maxSystemPromptChars": 50000,
          "maxContextTokens": 180000
        }
      },
      {
        "id": "researcher",
        "name": "Research Assistant",
        "thinkingDefault": "high",
        "skills": ["web-search", "research"],
        "memorySearch": {
          "enabled": true,
          "maxResults": 10,
          "recencyBoost": 1.5
        }
      }
    ]
  }
}
```

### ACP Agent with Subagents

```json
{
  "agents": {
    "list": [
      {
        "id": "coordinator",
        "name": "Coordinator",
        "runtime": {
          "type": "acp",
          "acp": {
            "agent": "codex",
            "mode": "persistent"
          }
        },
        "subagents": {
          "delegationMode": "balanced",
          "allowAgents": ["coder", "reviewer", "tester"],
          "model": {
            "provider": "anthropic",
            "model": "claude-sonnet-4"
          }
        }
      }
    ]
  },
  "bindings": [
    {
      "type": "route",
      "agentId": "coordinator",
      "match": {
        "channel": "discord",
        "guildId": "123456789"
      }
    }
  ]
}
```

## Related

- [Config Schema](/architecture-book/part-7-config-system/01-config-schema) - Schema architecture
- [Provider Config](/architecture-book/part-7-config-system/03-provider-config) - Model provider settings
- [Agent System](/architecture-book/part-2-core-modules/02-agents) - Agent runtime details
- [Tools System](/architecture-book/part-2-core-modules/04-tools) - Tools configuration