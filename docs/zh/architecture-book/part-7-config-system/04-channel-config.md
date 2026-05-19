---
summary: "Channel configuration for messaging platforms, permissions, and routing"
title: "Channel Config"
read_when:
  - Configuring messaging platforms
  - Setting up channel permissions
  - Understanding channel capabilities
---

# Channel Config

## Overview

OpenClaw's channel configuration system manages messaging platform integrations. Each channel has its own configuration namespace with platform-specific settings, access controls, and routing options.

```mermaid
flowchart TB
    subgraph Channels["ChannelsConfig"]
        Defaults["defaults"]
        ModelOverrides["modelByChannel"]
        PlatformConfigs["discord, telegram, slack, ..."]
    end

    subgraph Platform["Channel-Specific Config"]
        Core["enabled, allowFrom, dmPolicy"]
        Health["healthMonitor"]
        Threads["threadBindings"]
        Media["mediaMaxMb, callbackBaseUrl"]
    end

    subgraph Permissions["Access Control"]
        DMPolicy["dmPolicy"]
        GroupPolicy["groupPolicy"]
        ContextVisibility["contextVisibility"]
    end

    Defaults --> Platform
    PlatformConfigs --> Platform
    Platform --> Permissions
```

## Configuration Structure

### Main Channels Config

```typescript
// src/config/types.channels.ts
interface ChannelsConfig {
  /** Default settings applied to all channels. */
  defaults?: ChannelDefaultsConfig;
  /** Per-channel model overrides (provider -> channelId -> modelRef). */
  modelByChannel?: ChannelModelByChannelConfig;
  /** Platform-specific configurations. */
  discord?: DiscordConfig;
  googlechat?: GoogleChatConfig;
  imessage?: IMessageConfig;
  irc?: IrcConfig;
  msteams?: MSTeamsConfig;
  signal?: SignalConfig;
  slack?: SlackConfig;
  telegram?: TelegramConfig;
  whatsapp?: WhatsAppConfig;
  /** Plugin-owned channel configs keyed by arbitrary channel ids. */
  [key: string]: any;
}
```

### Channel Defaults

```typescript
interface ChannelDefaultsConfig {
  /** Default group policy. */
  groupPolicy?: GroupPolicy;
  /** Default context visibility. */
  contextVisibility?: ContextVisibilityMode;
  /** Default heartbeat visibility. */
  heartbeat?: ChannelHeartbeatVisibilityConfig;
  /** Default bot loop protection. */
  botLoopProtection?: ChannelBotLoopProtectionConfig;
}
```

## Common Channel Settings

### Access Control

```typescript
interface ExtensionChannelConfig {
  /** Enable or disable the channel. */
  enabled?: boolean;
  /** Sender allowlist (user ids or patterns). */
  allowFrom?: Array<string | number>;
  /** Default delivery target for CLI --deliver. */
  defaultTo?: string | number;
  /** Default account when multiple are configured. */
  defaultAccount?: string;
  /** Direct message policy (pairing, all, none). */
  dmPolicy?: string;
  /** Group conversation policy. */
  groupPolicy?: GroupPolicy;
  /** Message context visibility. */
  contextVisibility?: ContextVisibilityMode;
}
```

### Health Monitoring

```typescript
interface ChannelHealthMonitorConfig {
  /** Enable health monitoring. */
  enabled?: boolean;
  /** Health check interval in seconds. */
  intervalSeconds?: number;
  /** Timeout for health checks. */
  timeoutSeconds?: number;
  /** Number of consecutive failures before unhealthy. */
  failureThreshold?: number;
}
```

### Thread Bindings

```typescript
interface SessionThreadBindingsConfig {
  /** Enable session threading. */
  enabled?: boolean;
  /** Spawn sessions for new threads. */
  spawnSessions?: boolean;
  /** Default context mode for spawned sessions. */
  defaultSpawnContext?: "isolated" | "fork";
  /** @deprecated Use spawnSessions. */
  spawnAcpSessions?: boolean;
  /** @deprecated Use spawnSessions. */
  spawnSubagentSessions?: boolean;
}
```

## Channel-Specific Configurations

### Discord

```typescript
// src/config/types.discord.ts
interface DiscordConfig extends ExtensionChannelConfig {
  /** Bot token for authentication. */
  botToken?: SecretInput;
  /** Application ID for Discord API. */
  applicationId?: string;
  /** Guild-specific configurations. */
  guilds?: Record<string, DiscordGuildEntry>;
  /** Direct message settings. */
  dm?: DiscordDmConfig;
  /** Action permissions (reactions, stickers, etc.). */
  actions?: DiscordActionConfig;
  /** Streaming mode (off, partial, block, progress). */
  streamMode?: DiscordStreamMode;
  /** Reaction notification mode. */
  reactionNotifications?: DiscordReactionNotificationMode;
  /** PluralKit integration. */
  pluralKit?: DiscordPluralKitConfig;
  /** Mention aliases for bot commands. */
  mentionAliases?: DiscordMentionAliasesConfig;
}
```

### Discord Guild Configuration

```typescript
interface DiscordGuildEntry {
  slug?: string;
  requireMention?: boolean;
  ignoreOtherMentions?: boolean;
  tools?: GroupToolPolicyConfig;
  toolsBySender?: GroupToolPolicyBySenderConfig;
  reactionNotifications?: DiscordReactionNotificationMode;
  users?: string[];
  roles?: string[];
  channels?: Record<string, DiscordGuildChannelConfig>;
}

interface DiscordGuildChannelConfig {
  requireMention?: boolean;
  ignoreOtherMentions?: boolean;
  tools?: GroupToolPolicyConfig;
  skills?: string[];
  enabled?: boolean;
  users?: string[];
  roles?: string[];
  systemPrompt?: string;
  includeThreadStarter?: boolean;
  autoThread?: boolean;
  autoArchiveDuration?: "60" | "1440" | "4320" | "10080";
  autoThreadName?: "message" | "generated";
}
```

### Telegram

```typescript
// src/config/types.telegram.ts
interface TelegramConfig extends ExtensionChannelConfig {
  /** Bot token from @BotFather. */
  botToken?: SecretInput;
  /** API ID for MTProto authentication. */
  apiId?: string | number;
  /** API Hash for MTProto authentication. */
  apiHash?: string;
  /** Session string for existing account. */
  sessionString?: SecretInput;
  /** Phone number for new session. */
  phone?: string;
  /** Action permissions. */
  actions?: TelegramActionConfig;
  /** Thread bindings. */
  threadBindings?: TelegramThreadBindingsConfig;
  /** Network settings. */
  network?: TelegramNetworkConfig;
  /** Streaming mode. */
  streamMode?: TelegramStreamingMode;
  /** Inline button visibility scope. */
  inlineButtonsScope?: TelegramInlineButtonsScope;
  /** Exec approval configuration. */
  execApproval?: TelegramExecApprovalConfig;
  /** Capabilities override. */
  capabilities?: TelegramCapabilitiesConfig;
}
```

### Slack

```typescript
// src/config/types.slack.ts
interface SlackConfig extends ExtensionChannelConfig {
  /** Bot token for authentication. */
  botToken?: SecretInput;
  /** Signing secret for request verification. */
  signingSecret?: SecretInput;
  /** App token for Socket Mode. */
  appToken?: SecretInput;
  /** Socket Mode connection. */
  socketMode?: boolean;
  /** Workspace allowlist. */
  allowFrom?: string[];
  /** Channel-specific configs. */
  channels?: Record<string, SlackChannelConfig>;
}
```

## Model Routing

### Channel Model Overrides

Override the default model for specific channels:

```typescript
// Channel model configuration
interface ChannelModelByChannelConfig {
  // provider -> channelId -> modelRef
  [provider: string]: Record<string, string>;
}

// Example: Use different models per channel
{
  "channels": {
    "modelByChannel": {
      "anthropic": {
        "discord:guild1:channel1": "claude-opus-4",
        "discord:guild1:channel2": "claude-sonnet-4"
      },
      "openai": {
        "telegram:default": "gpt-4o"
      }
    }
  }
}
```

## Media Configuration

### Common Media Settings

```typescript
interface MediaConfig {
  /** Maximum file size in MB. */
  mediaMaxMb?: number;
  /** Callback base URL for webhooks. */
  callbackBaseUrl?: string;
  /** Preserve original filenames. */
  preserveFilenames?: boolean;
  /** Retention period in hours. */
  ttlHours?: number;
}
```

### Per-Channel Media Limits

```typescript
// Set media limits per channel
{
  "channels": {
    "discord": {
      "mediaMaxMb": 25
    },
    "telegram": {
      "mediaMaxMb": 50
    }
  }
}
```

## Health and Monitoring

### Channel Health Visibility

```typescript
interface ChannelHeartbeatVisibilityConfig {
  /** Show heartbeat in messages. */
  showInMessages?: boolean;
  /** Show in channel status. */
  showInStatus?: boolean;
}
```

### Bot Loop Protection

Prevent infinite loops in channels:

```typescript
interface ChannelBotLoopProtectionConfig {
  /** Enable loop detection. */
  enabled?: boolean;
  /** Messages per time window. */
  maxPerWindow?: number;
  /** Window duration in seconds. */
  windowSeconds?: number;
}
```

## Integration Architecture

```mermaid
flowchart TB
    subgraph Gateway
        Router["Message Router"]
        SessionMgr["Session Manager"]
    end

    subgraph Channels
        Config["Channel Config"]
        Discord["Discord Plugin"]
        Telegram["Telegram Plugin"]
        Slack["Slack Plugin"]
    end

    subgraph Runtime
        Health["Health Monitor"]
        Auth["Auth/Permissions"]
    end

    Router --> SessionMgr
    SessionMgr --> Config
    Config --> Discord
    Config --> Telegram
    Config --> Slack
    Discord --> Health
    Telegram --> Health
    Slack --> Health
```

## Example Configuration

### Full Discord Setup

```json
{
  "channels": {
    "defaults": {
      "groupPolicy": "allowlist",
      "contextVisibility": "channel",
      "heartbeat": {
        "showInMessages": true,
        "showInStatus": true
      }
    },
    "discord": {
      "botToken": { "env": "DISCORD_BOT_TOKEN" },
      "guilds": {
        "123456789": {
          "requireMention": true,
          "users": ["987654321"],
          "roles": ["111222333"],
          "channels": {
            "general": {
              "requireMention": false,
              "enabled": true
            },
            "ai-chat": {
              "requireMention": true,
              "autoThread": true,
              "autoArchiveDuration": "1440"
            }
          }
        }
      },
      "dm": {
        "enabled": true,
        "policy": "pairing"
      },
      "actions": {
        "reactions": true,
        "stickers": true,
        "threads": true
      }
    }
  }
}
```

### Full Telegram Setup

```json
{
  "channels": {
    "telegram": {
      "botToken": { "env": "TELEGRAM_BOT_TOKEN" },
      "actions": {
        "reactions": true,
        "sendMessage": true,
        "poll": true
      },
      "threadBindings": {
        "enabled": true,
        "spawnSessions": true,
        "defaultSpawnContext": "isolated"
      },
      "network": {
        "autoSelectFamily": true,
        "dnsResultOrder": "ipv4first"
      },
      "streamMode": "partial",
      "execApproval": {
        "enabled": "auto",
        "target": "dm"
      }
    }
  }
}
```

## Related

- [Config Schema](/architecture-book/part-7-config-system/01-config-schema) - Schema architecture
- [Channel Capabilities](/architecture-book/part-3-channels/01-capabilities) - Capability system
- [Health Monitoring](/architecture-book/part-2-core-modules/03-health) - Health system