---
summary: "用于知识库和搜索集成的内存 Host SDK"
title: "内存 Host SDK"
read_when:
  - 构建内存插件
  - 集成知识库
---

# 内存 Host SDK

## 概述

内存 Host SDK 提供了用于构建与 OpenClaw 内存系统集成的内存插件的接口。

```mermaid
flowchart TB
    Agent[Agent]
    MemoryHost[Memory Host]
    Engine[Memory Engine]
    KB[Knowledge Base]

    Agent --> MemoryHost
    MemoryHost --> Engine
    Engine --> KB
```

## SDK 结构

### 包导出

```typescript
import { createMemoryHost } from "@openclaw/memory-host-sdk";
import type { MemoryHost, MemoryEntry, SearchOptions } from "@openclaw/memory-host-sdk";
```

## 内存 Host

### Host 接口

```typescript
interface MemoryHost {
  // 生命周期
  initialize(config: MemoryConfig): Promise<void>;
  shutdown(): Promise<void>;

  // 存储
  store(entry: MemoryEntry): Promise<void>;
  get(key: string): Promise<MemoryEntry | null>;
  update(key: string, entry: Partial<MemoryEntry>): Promise<void>;
  delete(key: string): Promise<void>;

  // 搜索
  search(query: string, options?: SearchOptions): Promise<MemoryResult[]>;
  suggest(partial: string): Promise<string[]>;

  // 上下文
  buildContext(sessionId: string, prompt: string): Promise<MemoryContext>;

  // 维护
  compact(sessionId: string): Promise<void>;
  vacuum(): Promise<void>;
}
```

## 内存条目

### 条目结构

```typescript
interface MemoryEntry {
  id: string;
  key: string;
  type: MemoryType;
  content: string;
  metadata: MemoryMetadata;
  createdAt: Date;
  updatedAt: Date;
  embedding?: number[];
}

type MemoryType =
  | "fact"
  | "task"
  | "preference"
  | "knowledge"
  | "context"
  | "summary"
  | "reflection"
  | "commitment";

interface MemoryMetadata {
  source: "user" | "model" | "system" | "plugin";
  sessionKey?: string;
  channel?: string;
  importance?: number;
  confidence?: number;
  tags?: string[];
  links?: string[];
}
```

## 内存配置

### 配置 Schema

```typescript
interface MemoryConfig {
  // 存储
  backend: "memory" | "file" | "vector" | "hybrid";
  path?: string;

  // 向量存储
  vectorDb?: {
    type: "lancedb" | "qdrant" | "pinecone";
    url?: string;
    apiKey?: string;
    collection?: string;
  };

  // 嵌入
  embedding?: {
    provider: "openai" | "anthropic" | "local";
    model?: string;
    dimension?: number;
  };

  // 行为
  maxMemory?: number;
  compactThreshold?: number;
  retentionDays?: number;
}

const config: MemoryConfig = {
  backend: "hybrid",
  path: "./memory",
  vectorDb: {
    type: "lancedb",
    collection: "openclaw-memory",
  },
  embedding: {
    provider: "openai",
    model: "text-embedding-3-small",
    dimension: 1536,
  },
  maxMemory: 10000,
  compactThreshold: 0.8,
  retentionDays: 30,
};
```

## 搜索集成

### 搜索选项

```typescript
interface SearchOptions {
  limit?: number;
  threshold?: number;
  types?: MemoryType[];
  tags?: string[];
  dateRange?: {
    start?: Date;
    end?: Date;
  };
  includeArchived?: boolean;
}

interface MemoryResult {
  entry: MemoryEntry;
  score: number;
  snippet?: string;
  highlights?: string[];
}
```

### 搜索实现

```typescript
class VectorSearchEngine implements SearchEngine {
  async search(
    query: string,
    options?: SearchOptions
  ): Promise<MemoryResult[]> {
    // Generate query embedding
    const queryEmbedding = await this.embeddingService.embed(query);

    // Search vector store
    const results = await this.vectorStore.search(queryEmbedding, {
      limit: options?.limit ?? 10,
      threshold: options?.threshold ?? 0.7,
      filter: this.buildFilter(options),
    });

    // Transform to MemoryResult
    return results.map((r) => ({
      entry: r.entry,
      score: r.score,
      snippet: this.extractSnippet(r.entry.content, query),
      highlights: this.extractHighlights(r.entry.content, query),
    }));
  }

  private extractSnippet(content: string, query: string): string {
    const index = content.toLowerCase().indexOf(query.toLowerCase());
    if (index === -1) return content.slice(0, 200);

    const start = Math.max(0, index - 50);
    const end = Math.min(content.length, index + 150);
    return "..." + content.slice(start, end) + "...";
  }
}
```

## 上下文构建

### 上下文组装

```typescript
interface MemoryContext {
  messages: Message[];
  memories: MemoryEntry[];
  recentFacts: Fact[];
  activeTasks: Task[];
}

async buildContext(
  sessionId: string,
  prompt: string
): Promise<MemoryContext> {
  const [session, recentMemories, facts, tasks] = await Promise.all([
    this.getSession(sessionId),
    this.search(prompt, { limit: 5, types: ["context"] }),
    this.getFacts(sessionId),
    this.getActiveTasks(sessionId),
  ]);

  return {
    messages: session.messages,
    memories: recentMemories.map((r) => r.entry),
    recentFacts: facts,
    activeTasks: tasks,
  };
}
```

### 提示词集成

```typescript
function injectMemoryIntoPrompt(
  prompt: string,
  context: MemoryContext
): string {
  const sections: string[] = [];

  if (context.recentFacts.length > 0) {
    sections.push(`
## Known Facts
${context.recentFacts.map((f) => `- ${f.content}`).join("\n")}
    `.trim());
  }

  if (context.activeTasks.length > 0) {
    sections.push(`
## Active Tasks
${context.activeTasks.map((t) => `- [${t.status}] ${t.content}`).join("\n")}
    `.trim());
  }

  if (context.memories.length > 0) {
    sections.push(`
## Recent Context
${context.memories.map((m) => m.content).join("\n")}
    `.trim());
  }

  if (sections.length === 0) return prompt;

  return `${prompt}

${sections.join("\n\n")}
  `.trim();
}
```

## 事实提取

### 事实推断

```typescript
interface FactExtractor {
  extract(conversation: Message[]): Promise<Fact[]>;
  merge(existing: Fact[], newFacts: Fact[]): Fact[];
}

class FactExtractorImpl implements FactExtractor {
  async extract(conversation: Message[]): Promise<Fact[]> {
    const text = conversation.map((m) => m.content).join("\n");

    const response = await this.model.complete({
      prompt: `
Extract key facts from this conversation:
${text}

Format each fact as: { "type": "fact", "content": "...", "confidence": 0.0-1.0 }
      `,
    });

    return JSON.parse(response).facts;
  }

  merge(existing: Fact[], newFacts: Fact[]): Fact[] {
    const merged = [...existing];

    for (const newFact of newFacts) {
      const existingIndex = merged.findIndex(
        (f) => this.similarity(f.content, newFact.content) > 0.8
      );

      if (existingIndex >= 0) {
        // Update existing fact with higher confidence
        if (newFact.confidence > merged[existingIndex].confidence) {
          merged[existingIndex] = newFact;
        }
      } else {
        merged.push(newFact);
      }
    }

    return merged;
  }
}
```

## 压缩

### 压缩策略

```typescript
interface CompactionStrategy {
  shouldCompact(session: Session): boolean;
  compact(session: Session): Promise<CompactedSession>;
}

class SummaryCompaction implements CompactionStrategy {
  shouldCompact(session: Session): boolean {
    return (
      session.messageCount > 100 ||
      session.tokenCount > 8000
    );
  }

  async compact(session: Session): Promise<CompactedSession> {
    // Generate summary
    const summary = await this.generateSummary(session);

    // Keep recent messages
    const recentMessages = session.messages.slice(-10);

    return {
      sessionId: session.id,
      summary,
      recentMessages,
      compactedAt: new Date(),
      originalMessageCount: session.messageCount,
    };
  }
}
```

## DREAMS 系统

### 反思生成

```typescript
interface DreamsGenerator {
  generateReflection(session: Session): Promise<string>;
  mergeWithExisting(dreams: string, newReflection: string): string;
}

class DreamsGeneratorImpl implements DreamsGenerator {
  async generateReflection(session: Session): Promise<string> {
    const response = await this.model.complete({
      prompt: `
Analyze this conversation session and generate insights:

Recent messages:
${session.recentMessages.map((m) => `${m.role}: ${m.content}`).join("\n")}

Generate a reflection that includes:
1. Key topics discussed
2. User preferences observed
3. Tasks or commitments made
4. Important context to remember

Format as a markdown list.
      `,
    });

    const date = new Date().toISOString().split("T")[0];
    return `## ${date}\n\n${response}`;
  }

  mergeWithExisting(dreams: string, newReflection: string): string {
    // Prepend new reflection
    return `${newReflection}\n\n---\n\n${dreams}`;
  }
}
```

## 完整示例

```typescript
import { createMemoryHost } from "@openclaw/memory-host-sdk";

const host = createMemoryHost({
  backend: "hybrid",
  path: "./memory",
  vectorDb: { type: "lancedb", collection: "openclaw" },
  embedding: { provider: "openai", model: "text-embedding-3-small" },
});

await host.initialize();

// Store a memory
await host.store({
  id: generateId(),
  key: "user:preference:dark-mode",
  type: "preference",
  content: "User prefers dark mode interface",
  metadata: {
    source: "user",
    importance: 0.8,
    confidence: 0.95,
  },
  createdAt: new Date(),
  updatedAt: new Date(),
});

// Search memories
const results = await host.search("What does the user prefer?", {
  limit: 5,
  threshold: 0.7,
});

console.log("Found:", results.length, "memories");

// Build context for agent
const context = await host.buildContext("session-123", "What is user's name?");
console.log("Context:", context);

// Compact old memories
await host.compact("session-123");

await host.shutdown();
```

## 相关内容

- [内存系统](../part-8-session-memory/00-session-memory-overview) - 内存架构
- [上下文引擎](../part-8-session-memory/03-context-engine) - 上下文组装
- [压缩](../part-8-session-memory/04-compaction) - 内存压缩
