---
title: OpenClaw Architecture Book
hide:
  - navigation
  - toc
  - footer
---

# OpenClaw Architecture Book

<div class="hero-section">
<div class="hero-content">

## 深入理解 OpenClaw 系统架构

开源的多通道 AI 网关，用于将聊天平台与 AI Agent 连接起来。

[:material-arrow-right: English Version](en/architecture-book/index.md){ .md-button .md-button--primary }
[:material-arrow-right: 中文版本](zh/architecture-book/index.md){ .md-button }

</div>
</div>

## 本书特点

[:material-book-open-variant: 九大章节] 涵盖从基础概念到高级功能的完整知识体系

[:material-code-braces: 深入源码] 结合实际代码解析架构设计和实现细节

[:material-wrench: 开发指南] 包含测试、调试和部署的最佳实践

[:material-translate: 双语支持] 英文和中文版本同步更新

## 快速导航

=== "📚 Foundations 基础"

    - [Introduction 简介](en/architecture-book/part-1-foundations/01-introduction.md)
    - [系统概览](zh/architecture-book/part-1-foundations/02-system-overview.md)
    - [核心概念](en/architecture-book/part-1-foundations/03-core-concepts.md)
    - [目录结构](en/architecture-book/part-1-foundations/04-directory-structure.md)

=== "⚙️ Core Modules 核心模块"

    - [Gateway](en/architecture-book/part-2-core-modules/01-gateway.md)
    - [Agents](en/architecture-book/part-2-core-modules/02-agents.md)
    - [Sessions](en/architecture-book/part-2-core-modules/03-sessions.md)
    - [Memory](en/architecture-book/part-2-core-modules/04-memory.md)

=== "🔌 Plugin System 插件系统"

    - [Plugin Architecture](en/architecture-book/part-3-plugin-system/01-plugin-architecture.md)
    - [Plugin SDK](en/architecture-book/part-3-plugin-system/02-plugin-sdk.md)
    - [Writing Plugins](en/architecture-book/part-3-plugin-system/05-writing-plugins.md)

=== "🌐 Gateway Protocol 网关协议"

    - [Protocol Overview](en/architecture-book/part-4-gateway-protocol/01-protocol-overview.md)
    - [WebSocket Transport](en/architecture-book/part-4-gateway-protocol/02-ws-transport.md)
    - [Events and RPC](en/architecture-book/part-4-gateway-protocol/04-events-and-rpc.md)

## 相关链接

- [OpenClaw 官网](https://openclaw.ai)
- [GitHub 仓库](https://github.com/openclaw/openclaw)
- [在线文档](https://docs.openclaw.ai)

<style>
.hero-section {
    background: linear-gradient(135deg, #5c6bc0 0%, #3949ab 100%);
    border-radius: 16px;
    padding: 3rem 2rem;
    margin: 2rem 0;
    color: white;
}

.hero-content {
    text-align: center;
}

.hero-content h1 {
    color: white;
    margin-bottom: 1rem;
}

.hero-content p {
    color: rgba(255,255,255,0.9);
    font-size: 1.2rem;
    margin-bottom: 2rem;
}

.hero-content .md-button {
    margin: 0.5rem;
}

.hero-content .md-button--primary {
    background: white;
    color: #3949ab;
}
</style>