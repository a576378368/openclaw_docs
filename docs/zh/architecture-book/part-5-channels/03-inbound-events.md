---
summary: "事件规范化、发送者身份和会话绑定"
title: "入站事件"
read_when:
  - 处理传入消息
  - 理解事件规范化
---

# 入站事件

## 概述

来自消息平台的入站事件在由网关处理之前会被规范化为标准格式。

## 事件规范化管道

```mermaid
flowchart TB
    A[平台事件] --> B[解析]
    B --> C[规范化发送者]
    C --> D[规范化内容]
    D --> E[提取元数据]
    E --> F[验证]
    F --> G[入站消息]
```

## 事件类型

### 支持的事件类型

| 事件 | 描述 | 处理器 |
|-------|-------------|---------|
| message | 新消息 | `onMessage` |
| edited_message | 消息已编辑 | `onEdit` |
| deleted_message | 消息已删除 | `onDelete` |
| callback_query | 按钮点击 | `onCallback` |
| reaction | 反应添加/移除 | `onReaction` |
| command | Bot 命令 | `onCommand` |

## 发送者规范化

### 发送者解析

```typescript
interface SenderResolver {
  resolve(rawSender: unknown, platform: string): Sender;
}

// Telegram 发送者
function resolveTelegramSender(raw: TelegramUser): Sender {
  return {
    id: raw.id.toString(),
    name: raw.first_name + (raw.last_name ? ` ${raw.last_name}` : ""),
    username: raw.username,
    isBot: raw.is_bot,
  };
}

// Discord 发送者
function resolveDiscordSender(raw: DiscordUser): Sender {
  return {
    id: raw.id,
    name: raw.global_name || raw.username,
    username: raw.username,
    isBot: raw.bot,
  };
}
```

### 匿名 vs 已识别

```typescript
interface Sender {
  id: string;
  name: string;
  username?: string;
  mention?: string;
  isBot: boolean;
}

// 匿名发送者（无法识别用户身份的群组）
const anonymousSender: Sender = {
  id: "anonymous",
  name: "Anonymous",
  isBot: false,
};
```

## 会话绑定

### 会话解析

```typescript
interface ConversationBinding {
  channel: string;
  peer: string;
  peerType: PeerType;
  thread?: string;
}

function resolveConversation(
  event: PlatformEvent,
  platform: string
): ConversationBinding {
  switch (platform) {
    case "telegram":
      return resolveTelegramConversation(event);
    case "discord":
      return resolveDiscordConversation(event);
    default:
      throw new Error(`未知平台: ${platform}`);
  }
}

function resolveTelegramConversation(event: TelegramEvent): ConversationBinding {
  const chat = event.chat;

  if (chat.type === "private") {
    return {
      channel: "telegram",
      peer: chat.id.toString(),
      peerType: "user",
    };
  }

  if (chat.type === "group" || chat.type === "supergroup") {
    return {
      channel: "telegram",
      peer: chat.id.toString(),
      peerType: "group",
      thread: event.message_thread_id?.toString(),
    };
  }

  return {
    channel: "telegram",
    peer: chat.id.toString(),
    peerType: "channel",
  };
}
```

### 线程解析

```typescript
interface ThreadInfo {
  id: string;
  parentId?: string;
  title?: string;
  messageCount: number;
}

function resolveThread(event: PlatformEvent): string | undefined {
  // Telegram 线程
  if (event.message_thread_id) {
    return event.message_thread_id.toString();
  }

  // Discord 线程
  if (event.thread) {
    return event.thread.id;
  }

  // Slack 线程
  if (event.thread_ts) {
    return event.thread_ts;
  }

  return undefined;
}
```

## 内容规范化

### 文本规范化

```typescript
function normalizeText(content: string, platform: string): string {
  let text = content.trim();

  // 移除平台特定的格式化标签
  text = removeFormatting(text, platform);

  // 规范化空白字符
  text = text.replace(/\s+/g, " ");

  // 处理提及
  text = normalizeMentions(text, platform);

  // 处理命令
  text = normalizeCommands(text);

  return text;
}

function removeFormatting(text: string, platform: string): string {
  switch (platform) {
    case "telegram":
      return text
        .replace(/<b>(.*?)<\/b>/g, "*$1*")
        .replace(/<i>(.*?)<\/i>/g, "_$1_");
    case "discord":
      return text
        .replace(/\*\*(.*?)\*\*/g, "**$1**")
        .replace(/__(.*?)__/g, "__$1__");
    default:
      return text;
  }
}
```

### 提及规范化

```typescript
function normalizeMentions(text: string, platform: string): string {
  switch (platform) {
    case "telegram":
      // @username -> @username
      // tg://user?id=123 -> @123
      return text.replace(/tg:\/\/user\?id=(\d+)/g, "@$1");

    case "discord":
      // <@123> -> @username
      // <#456> -> #channel
      return text
        .replace(/<@(\d+)>/g, "@user:$1")
        .replace(/<#(\d+)>/g, "#channel:$1")
        .replace(/<@&(\d+)>/g, "@role:$1");

    case "slack":
      // <@U123> -> @username
      // <#C456> -> #channel
      return text
        .replace(/<@U([A-Z0-9]+)\|([^>]+)>/g, "@$2")
        .replace(/<#C([A-Z0-9]+)\|([^>]+)>/g, "#$2");

    default:
      return text;
  }
}
```

## 命令提取

### 命令检测

```typescript
interface Command {
  name: string;
  args: string[];
  raw: string;
}

function extractCommand(
  content: string,
  platform: string,
  botUsername?: string
): Command | null {
  // 检查命令前缀
  const prefixes = getCommandPrefixes(platform);
  const prefix = prefixes.find(p => content.startsWith(p));

  if (!prefix) return null;

  const remainder = content.slice(prefix.length);
  const parts = remainder.split(/\s+/);
  let name = parts[0].toLowerCase();

  // 处理 @botname 后缀 (Telegram)
  if (platform === "telegram" && botUsername) {
    const [, suffix] = name.split("@");
    if (suffix && suffix !== botUsername.toLowerCase()) {
      return null; // 其他 bot 的命令
    }
    name = name.split("@")[0];
  }

  return {
    name,
    args: parts.slice(1),
    raw: remainder,
  };
}

function getCommandPrefixes(platform: string): string[] {
  switch (platform) {
    case "telegram":
      return ["/"];
    case "discord":
      return ["!", "/"];
    case "slack":
      return ["/"];
    case "whatsapp":
      return ["/"];
    default:
      return ["/"];
  }
}
```

## 媒体处理

### 媒体提取

```typescript
function extractMedia(event: PlatformEvent): MediaAttachment | undefined {
  if (event.photo) {
    const largest = event.photo[event.photo.length - 1];
    return {
      id: largest.file_id,
      type: "image",
      mimeType: "image/jpeg",
      width: largest.width,
      height: largest.height,
      size: largest.file_size,
    };
  }

  if (event.document) {
    return {
      id: event.document.file_id,
      type: event.document.mime_type?.startsWith("image/") ? "image" :
            event.document.mime_type?.startsWith("video/") ? "video" :
            event.document.mime_type?.startsWith("audio/") ? "audio" : "document",
      mimeType: event.document.mime_type || "application/octet-stream",
      filename: event.document.file_name,
      size: event.document.file_size,
    };
  }

  if (event.voice) {
    return {
      id: event.voice.file_id,
      type: "audio",
      mimeType: "audio/ogg",
      duration: event.voice.duration,
    };
  }

  return undefined;
}
```

## 元数据提取

### 平台特定元数据

```typescript
interface MetadataExtractor {
  (event: PlatformEvent): MessageMetadata;
}

const telegramMetadataExtractor: MetadataExtractor = (event) => ({
  channelId: "telegram",
  originalId: event.message_id?.toString(),
  threadId: event.message_thread_id?.toString(),
  forwardedFrom: event.forward_from ? {
    channel: "telegram",
    messageId: event.forward_from_message_id?.toString(),
  } : undefined,
  command: extractCommand(event.text || event.caption || "", "telegram")?.name,
  commandArgs: extractCommand(event.text || event.caption || "", "telegram")?.args,
});

const discordMetadataExtractor: MetadataExtractor = (event) => ({
  channelId: "discord",
  originalId: event.id,
  threadId: event.thread?.id,
  guildId: event.guild_id,
  command: extractCommand(event.content, "discord")?.name,
});
```

## 验证

### 消息验证

```typescript
interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
}

function validateInboundMessage(message: InboundMessage): ValidationResult {
  const errors: ValidationError[] = [];

  // 检查必填字段
  if (!message.id) {
    errors.push({ field: "id", message: "消息 ID 是必填的" });
  }

  if (!message.channel) {
    errors.push({ field: "channel", message: "通道是必填的" });
  }

  if (!message.peer) {
    errors.push({ field: "peer", message: "对端是必填的" });
  }

  // 检查内容或媒体
  if (!message.content && !message.media) {
    errors.push({ field: "content", message: "消息必须有内容或媒体" });
  }

  // 验证发送者
  if (!message.sender?.id) {
    errors.push({ field: "sender.id", message: "发送者 ID 是必填的" });
  }

  return {
    valid: errors.length === 0,
    errors,
  };
}
```

## 相关

- [通道架构](./01-channel-architecture.md) - 通道设计
- [通道抽象](./02-channel-abstract.md) - 接口定义
- [消息处理](./04-message-processing.md) - 处理管道