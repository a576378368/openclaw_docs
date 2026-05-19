---
summary: "Channel plugin implementation, transport, and event handling"
title: "Channel Plugins"
read_when:
  - Building channel plugins
  - Understanding platform integration
---

# Channel Plugins

## Overview

Channel plugins connect OpenClaw to messaging platforms, handling the transport, message format conversion, and event normalization.

```mermaid
flowchart TB
    A[Platform] --> B[Channel Plugin]
    B --> C[Gateway]
    C --> B
    B --> D[Platform]

    subgraph ChannelPlugin
        E[Transport]
        F[Parser]
        G[Formatter]
        H[Event Handler]
    end

    A --> E
    E --> F
    F --> H
    H --> G
    G --> D
```

## Channel Plugin Structure

### Entry Point

```typescript
import { channelEntry } from "@openclaw/plugin-sdk/runtime/channel";

export const entry = channelEntry({
  id: "telegram",
  name: "Telegram",

  // Lifecycle
  async connect(config) {
    return new TelegramBot(config.token, config.options);
  },

  async disconnect(bot) {
    await bot.close();
  },

  // Messaging
  async send(target, message, bot) {
    // Implementation
  },

  // Event handling
  onMessage(bot, handler) {
    bot.on("message", handler);
  },

  // Media handling
  async uploadMedia(bot, data, type) {
    // Implementation
  },
});
```

## Transport Layer

### Transport Interface

```typescript
interface ChannelTransport {
  // Connection management
  connect(config: TransportConfig): Promise<void>;
  disconnect(): Promise<void>;
  isConnected(): boolean;

  // Sending
  send(target: Target, payload: PlatformPayload): Promise<void>;

  // Receiving
  onMessage(handler: MessageHandler): void;
  onDisconnect(handler: DisconnectHandler): void;
}
```

### Transport Implementation

```typescript
class TelegramTransport implements ChannelTransport {
  private bot?: TelegramBot;
  private messageHandlers: MessageHandler[] = [];
  private disconnectHandlers: DisconnectHandler[] = [];

  async connect(config: TelegramConfig): Promise<void> {
    this.bot = new TelegramBot(config.token);

    // Set up webhook or long polling
    if (config.useWebhook) {
      await this.bot.setWebhook(config.webhookUrl);
    } else {
      this.bot.on("polling", { interval: 1000 });
    }

    // Forward messages to handlers
    this.bot.on("message", (msg) => {
      const normalized = this.normalizeMessage(msg);
      this.messageHandlers.forEach((h) => h(normalized));
    });

    this.bot.on("disconnect", () => {
      this.disconnectHandlers.forEach((h) => h());
    });
  }

  async disconnect(): Promise<void> {
    await this.bot?.stopPolling();
    this.bot = undefined;
  }

  async send(target: Target, payload: PlatformPayload): Promise<void> {
    if (!this.bot) throw new Error("Not connected");

    if (payload.text) {
      await this.bot.sendMessage(target.peer, payload.text, {
        parse_mode: payload.format,
      });
    }

    if (payload.media) {
      await this.sendMedia(target, payload.media);
    }
  }

  onMessage(handler: MessageHandler): void {
    this.messageHandlers.push(handler);
  }

  onDisconnect(handler: DisconnectHandler): void {
    this.disconnectHandlers.push(handler);
  }
}
```

## Message Format Conversion

### Outbound Formatting

```typescript
interface OutboundFormatter {
  format(target: Target, message: OutboundMessage): PlatformPayload;
}

class TelegramFormatter implements OutboundFormatter {
  format(target: Target, message: OutboundMessage): PlatformPayload {
    const payload: TelegramPayload = {
      chat_id: target.peer,
    };

    if (message.content) {
      payload.text = this.formatText(message.content);
      payload.parse_mode = message.format === "html" ? "HTML" : "Markdown";
    }

    if (message.media) {
      return this.formatMedia(target, message.media);
    }

    if (message.buttons) {
      payload.reply_markup = this.formatButtons(message.buttons);
    }

    return payload;
  }

  private formatText(content: MessageContent): string {
    if (typeof content === "string") {
      return content;
    }
    // Handle structured content
    return content.blocks.map((b) => b.text).join("\n");
  }

  private formatButtons(
    buttons: Button[]
  ): InlineKeyboardMarkup {
    return {
      inline_keyboard: buttons.map((row) =>
        row.buttons.map((btn) => ({
          text: btn.label,
          url: btn.url,
          callback_data: btn.data,
        }))
      ),
    };
  }
}
```

### Inbound Normalization

```typescript
interface MessageNormalizer {
  normalize(platformMessage: unknown): InboundMessage;
}

class TelegramNormalizer implements MessageNormalizer {
  normalize(raw: TelegramMessage): InboundMessage {
    return {
      id: raw.message_id.toString(),
      channel: "telegram",
      peer: raw.chat.id.toString(),
      peerType: this.getPeerType(raw.chat),
      sender: {
        id: raw.from?.id.toString(),
        name: this.getSenderName(raw.from),
        username: raw.from?.username,
      },
      content: this.extractContent(raw),
      media: this.extractMedia(raw),
      timestamp: new Date(raw.date * 1000),
      replyTo: raw.reply_to_message?.message_id.toString(),
      metadata: {
        chatType: raw.chat.type,
        isEdited: !!raw.edit_date,
      },
    };
  }

  private extractContent(msg: TelegramMessage): string {
    if (msg.text) return msg.text;
    if (msg.caption) return msg.caption;
    if (msg.photo) return "[Photo]";
    if (msg.document) return "[Document]";
    if (msg.sticker) return "[Sticker]";
    return "";
  }

  private extractMedia(msg: TelegramMessage): MediaAttachment | undefined {
    if (msg.photo) {
      const largest = msg.photo[msg.photo.length - 1];
      return {
        type: "image",
        id: largest.file_id,
        url: largest.file_id, // Will be resolved later
        mimeType: "image/jpeg",
        width: largest.width,
        height: largest.height,
      };
    }
    if (msg.document) {
      return {
        type: "file",
        id: msg.document.file_id,
        url: msg.document.file_id,
        mimeType: msg.document.mime_type,
        filename: msg.document.file_name,
        size: msg.document.file_size,
      };
    }
    return undefined;
  }
}
```

## Event Handling

### Event Types

```typescript
interface ChannelEvents {
  onMessage(handler: MessageHandler): void;
  onEdit(handler: EditHandler): void;
  onReaction(handler: ReactionHandler): void;
  onCommand(handler: CommandHandler): void;
  onCallback(handler: CallbackHandler): void;
}

type MessageHandler = (message: InboundMessage) => void | Promise<void>;
type EditHandler = (edit: MessageEdit) => void | Promise<void>;
type ReactionHandler = (reaction: Reaction) => void | Promise<void>;
type CommandHandler = (command: Command) => void | Promise<void>;
type CallbackHandler = (callback: Callback) => void | Promise<void>;
```

### Command Handling

```typescript
class TelegramCommandHandler {
  private commands: Map<string, CommandHandler> = new Map();

  register(command: string, handler: CommandHandler): void {
    this.commands.set(command, handler);
  }

  handle(message: InboundMessage): void {
    const text = message.content;
    if (!text.startsWith("/")) return;

    const parts = text.slice(1).split(" ");
    const command = parts[0].split("@")[0]; // Remove @botname
    const args = parts.slice(1);

    const handler = this.commands.get(command);
    if (!handler) return;

    handler({
      command,
      args,
      message,
      fullText: text,
    });
  }
}

// Usage
const cmdHandler = new TelegramCommandHandler();
cmdHandler.register("start", async ({ message }) => {
  // Handle /start command
});

bot.on("message", (msg) => {
  const normalized = normalizer.normalize(msg);
  cmdHandler.handle(normalized);
  // Also forward to general message handler
});
```

### Callback Queries

```typescript
interface CallbackHandler {
  (callback: {
    id: string;
    data: string;
    message: InboundMessage;
    from: Sender;
    answer: (text?: string, showAlert?: boolean) => Promise<void>;
  }): void | Promise<void>;
}

// Implementation
bot.on("callback_query", async (query) => {
  const callback = {
    id: query.id,
    data: query.data || "",
    message: normalizer.normalize(query.message),
    from: normalizer.normalizeSender(query.from),
    answer: async (text?: string, showAlert?: boolean) => {
      await bot.answerCallbackQuery(query.id, {
        text,
        show_alert: showAlert,
      });
    },
  };

  await this.callbackHandler(callback);
});
```

## Media Handling

### Media Upload

```typescript
class TelegramMediaHandler {
  async uploadMedia(
    bot: TelegramBot,
    data: Buffer,
    type: MediaType
  ): Promise<string> {
    switch (type) {
      case "image":
        return this.uploadPhoto(bot, data);
      case "video":
        return this.uploadVideo(bot, data);
      case "audio":
        return this.uploadAudio(bot, data);
      case "file":
      default:
        return this.uploadDocument(bot, data);
    }
  }

  private async uploadPhoto(bot: TelegramBot, data: Buffer): Promise<string> {
    const file = await bot.uploadFile("photo", {
      source: data,
      filename: "image.jpg",
    });
    return file.file_id;
  }

  private async uploadDocument(
    bot: TelegramBot,
    data: Buffer
  ): Promise<string> {
    const file = await bot.uploadFile("document", {
      source: data,
      filename: "document",
    });
    return file.file_id;
  }
}
```

### Media Resolution

```typescript
interface MediaResolver {
  resolveMediaUrl(media: MediaAttachment): Promise<string>;
}

class TelegramMediaResolver implements MediaResolver {
  async resolveMediaUrl(media: MediaAttachment): Promise<string> {
    // Get file path from Telegram API
    const file = await this.bot.getFile(media.id);
    return `https://api.telegram.org/file/bot${this.bot.token}/${file.file_path}`;
  }
}
```

## Platform-Specific Features

### Feature Detection

```typescript
interface ChannelCapabilities {
  text: boolean;
  markdown: boolean;
  html: boolean;
  images: boolean;
  videos: boolean;
  audio: boolean;
  files: boolean;
  buttons: boolean;
  inlineButtons: boolean;
  reactions: boolean;
  threads: boolean;
  replies: boolean;
  forward: boolean;
  stickers: boolean;
}

const telegramCapabilities: ChannelCapabilities = {
  text: true,
  markdown: true,
  html: true,
  images: true,
  videos: true,
  audio: true,
  files: true,
  buttons: true,
  inlineButtons: true,
  reactions: true,
  threads: false,  // Not in 1:1 DMs
  replies: true,
  forward: true,
  stickers: true,
};
```

### Platform Limitations

```typescript
const platformLimitations = {
  telegram: {
    maxMessageLength: 4096,
    maxCaptionLength: 1024,
    maxPhotoSize: "10MB",
    maxVideoSize: "50MB",
    maxFileSize: "2000MB",
    buttonRows: 4,
    buttonsPerRow: 12,
  },
  discord: {
    maxMessageLength: 2000,
    maxEmbedTitle: 256,
    maxEmbedDescription: 4096,
    maxFields: 25,
    maxFieldValue: 1024,
    maxFooterText: 2048,
  },
};
```

## Error Handling

### Retry Logic

```typescript
interface RetryConfig {
  maxRetries: number;
  initialDelay: number;
  maxDelay: number;
  backoffMultiplier: number;
}

async function sendWithRetry(
  target: Target,
  message: OutboundMessage,
  config: RetryConfig = defaultRetryConfig
): Promise<void> {
  let delay = config.initialDelay;
  let lastError: Error;

  for (let i = 0; i <= config.maxRetries; i++) {
    try {
      await this.send(target, message);
      return;
    } catch (error) {
      lastError = error;

      if (!isRetryableError(error)) {
        throw error;
      }

      if (i < config.maxRetries) {
        await sleep(delay);
        delay = Math.min(delay * config.backoffMultiplier, config.maxDelay);
      }
    }
  }

  throw lastError;
}

function isRetryableError(error: Error): boolean {
  // Network errors, rate limits, temporary failures
  return (
    error instanceof NetworkError ||
    error instanceof RateLimitError ||
    error instanceof TemporaryFailureError
  );
}
```

## Testing

### Mock Transport

```typescript
class MockChannelTransport implements ChannelTransport {
  private connected = false;
  private messages: InboundMessage[] = [];

  async connect(): Promise<void> {
    this.connected = true;
  }

  async disconnect(): Promise<void> {
    this.connected = false;
  }

  isConnected(): boolean {
    return this.connected;
  }

  async send(): Promise<void> {
    // No-op for testing
  }

  onMessage(handler: MessageHandler): void {
    // Store handler for test injection
  }

  // Test helpers
  injectMessage(message: InboundMessage): void {
    this.messages.push(message);
    this.messageHandlers.forEach((h) => h(message));
  }
}
```

### Integration Test

```typescript
describe("Telegram Channel Plugin", () => {
  it("should handle incoming messages", async () => {
    const plugin = createTestChannelPlugin({
      entry: telegramEntry,
      config: { token: "test-token" },
    });

    await plugin.activate();
    await plugin.connect();

    // Simulate incoming message
    const mockUpdate = {
      message_id: 1,
      chat: { id: 123, type: "private" },
      from: { id: 456, first_name: "Test" },
      text: "/start",
      date: Date.now() / 1000,
    };

    const handler = plugin.getMessageHandler();
    handler(telegramNormalizer.normalize(mockUpdate));

    // Verify behavior
    expect(plugin.sentMessages).toHaveLength(1);
  });
});
```

## Related

- [Plugin Architecture](/architecture-book/part-3-plugin-system/01-plugin-architecture) - Plugin design
- [Plugin SDK](/architecture-book/part-3-plugin-system/02-plugin-sdk) - SDK documentation
- [Channel System](/architecture-book/part-5-channels/01-channel-architecture) - Channel architecture