# OpenClaw Architecture Book

📚 深入理解 OpenClaw 系统架构与实现细节的开源教材

[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-Deploy-brightgreen)](https://a576378368.github.io/openclaw_docs)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

## 在线阅读

**https://a576378368.github.io/openclaw_docs**

## 关于本书

OpenClaw 是一个开源的多通道 AI 网关，用于将聊天平台与 AI Agent 连接起来。本书深入介绍其设计理念、架构模式和实现细节。

## 本书结构

本书分为九个部分，共 49 篇文档：

| 章节 | 内容 |
|------|------|
| 基础 | 简介、系统概览、核心概念、目录结构 |
| 核心模块 | 网关、Agent、会话、内存、工具、MCP、Flow、任务 |
| 插件系统 | 插件架构、SDK、契约、运行时、开发指南 |
| 网关协议 | 协议概述、WebSocket 传输、消息流、事件和 RPC、认证和安全 |
| 通道系统 | 通道架构、抽象、入站事件、消息处理、传输层 |
| SDK 和 API | App SDK、插件 SDK、Memory Host SDK、API 参考 |
| 配置系统 | 配置架构、Provider 配置、通道配置、Agent 配置、插件配置 |
| 会话内存 | 会话内存概览、上下文引擎、压缩、多 Agent |
| 开发者指南 | 开发工作流、构建系统、测试、调试、部署 |

## 面向读者

- **贡献者** - 想要了解 OpenClaw 内部原理
- **插件开发者** - 想要构建扩展
- **集成者** - 想要连接自定义平台
- **运维者** - 想要了解系统如何工作

## 前置知识

- TypeScript/Node.js 熟练度
- 熟悉 AI/LLM 概念
- 理解 WebSocket 通信
- 基本了解插件/扩展模式

## 本地开发

```bash
# 安装依赖
pip install mkdocs mkdocs-material

# 启动开发服务器
mkdocs serve --dev-addr 0.0.0.0:8000

# 构建静态站点
mkdocs build
```

## 相关链接

- [OpenClaw 官网](https://openclaw.ai)
- [GitHub 仓库](https://github.com/openclaw/openclaw)
- [在线文档](https://docs.openclaw.ai)

## 许可

MIT License