# 01 系统整体架构 / Overall System Architecture

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `server.js`, `Plugin.js`, `WebSocketServer.js`, `KnowledgeBaseManager.js`, `modules/`  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [项目定位 / Project Identity](#1-项目定位--project-identity)
2. [技术栈 / Tech Stack](#2-技术栈--tech-stack)
3. [目录结构 / Directory Structure](#3-目录结构--directory-structure)
4. [核心三角架构 / Core Triangle](#4-核心三角架构--core-triangle)
5. [架构全景图 / Architecture Panorama](#5-架构全景图--architecture-panorama)
6. [启动序列 / Boot Sequence](#6-启动序列--boot-sequence)
7. [核心模块详解 / Core Module Reference](#7-核心模块详解--core-module-reference)
8. [聊天请求完整流程 / Chat Request Full Flow](#8-聊天请求完整流程--chat-request-full-flow)
9. [设计决策 / Design Decisions](#9-设计决策--design-decisions)

---

## 1. 项目定位 / Project Identity

### 中文

VCPToolBox（**Variable & Command Protocol ToolBox**）是一个 **Node.js 核心的 AI 能力增强中间层**。它部署在 AI 客户端（SillyTavern / VCPChat / OpenWebUI 等）与上游 LLM API（OpenAI / Gemini / 自托管模型等）之间，实现：

- 🛠️ **工具执行** — 通过插件让 AI 调用外部工具（搜索、代码执行、图像生成等）
- 💾 **持久化记忆** — 基于向量 RAG 的长期记忆系统
- 🌐 **分布式调度** — 跨节点工具执行与文件传输
- 🔄 **模型无关** — 支持任意 OpenAI 兼容 API，不依赖 Function Calling 特性

### English

VCPToolBox (**Variable & Command Protocol ToolBox**) is a **Node.js-based AI capability enhancement middleware**. It sits between AI frontends (SillyTavern / VCPChat / OpenWebUI, etc.) and upstream LLM APIs (OpenAI / Gemini / self-hosted models, etc.), providing:

- 🛠️ **Tool Execution** — Plugins allow the AI to call external tools (search, code execution, image generation, etc.)
- 💾 **Persistent Memory** — Long-term memory system based on vector RAG
- 🌐 **Distributed Dispatch** — Cross-node tool execution and file transfer
- 🔄 **Model-Agnostic** — Supports any OpenAI-compatible API; no Function Calling required

📁 `README.md`, `VCP.md` — 项目哲学与背景 / Project philosophy & background

---

## 2. 技术栈 / Tech Stack

| 层级 / Layer | 技术 / Technology | 文件 / File |
|---|---|---|
| 运行时 / Runtime | Node.js (CommonJS) | `server.js:1` |
| Web 框架 / Web Framework | Express.js | `server.js:231` |
| 向量索引 / Vector Index | Rust N-API (USearch/Vexus) | `rust-vexus-lite/` |
| 数据库 / Database | SQLite (better-sqlite3, WAL mode) | `KnowledgeBaseManager.js:83` |
| WebSocket | `ws` library | `WebSocketServer.js:46` |
| 文件监听 / File Watch | chokidar | `KnowledgeBaseManager.js:901` |
| 进程管理 / Process Mgmt | pm2-runtime | `Dockerfile` |
| Python 桥 / Python Bridge | child_process spawn | `Plugin.js` (Python plugins) |
| 日志 / Logging | 自定义模块 / Custom module | `modules/logger.js` |

---

## 3. 目录结构 / Directory Structure

```
VCPToolBox/                        ← 扁平化根目录 (no src/ layer)
├── server.js                      ← ✅ HTTP/SSE 主入口 / Main HTTP entry
├── Plugin.js                      ← ✅ 插件生命周期总控 / Plugin lifecycle manager
├── WebSocketServer.js             ← ✅ 分布式 WS 骨架 / Distributed WS backbone
├── KnowledgeBaseManager.js        ← ✅ RAG/向量库总控 / RAG/vector DB manager
├── EPAModule.js                   ← ✅ 语义空间投影 / Semantic space projection
├── ResidualPyramid.js             ← ✅ 残差金字塔 / Residual pyramid
├── ResultDeduplicator.js          ← ✅ SVD 结果去重 / SVD result deduplication
├── FileFetcherServer.js           ← ✅ 跨节点文件获取 / Cross-node file fetch
├── WorkerPool.js                  ← ✅ 工作池管理 / Worker pool management
├── TextChunker.js                 ← ✅ 文本分块 / Text chunking
├── EmbeddingUtils.js              ← ✅ Embedding 工具函数 / Embedding utilities
├── vcpInfoHandler.js              ← ✅ VCP 信息推送 / VCP info broadcasting
├── modelRedirectHandler.js        ← ✅ 模型重定向 / Model redirect
│
├── modules/                       ← 复用后端模块 / Reusable backend modules
│   ├── chatCompletionHandler.js   ← 对话主流程编排 / Chat completion orchestrator
│   ├── messageProcessor.js        ← 变量替换管线 / Variable substitution pipeline
│   ├── agentManager.js            ← Agent 别名映射 / Agent alias mapping
│   ├── logger.js                  ← 日志系统 / Logging system
│   └── ...
│
├── routes/                        ← Express 路由层 / Express routing layer
│   ├── specialModelRouter.js      ← 特殊模型白名单透传 / Special model whitelist
│   ├── adminPanelRoutes.js        ← 管理面板 API / Admin panel API
│   └── forumApi.js                ← 论坛 API / Forum API
│
├── Plugin/                        ← 插件目录 / Plugin directory (79 active + 8 disabled)
│   └── <PluginName>/
│       ├── plugin-manifest.json   ← 启用态契约 / Active plugin contract
│       └── plugin-manifest.json.block ← 禁用标记 / Disabled marker
│
├── AdminPanel/                    ← 内嵌管理前端 / Embedded admin frontend (static)
├── rust-vexus-lite/               ← Rust N-API 向量引擎 / Rust N-API vector engine
├── VCPChrome/                     ← Chrome 扩展 / Chrome extension
├── OpenWebUISub/                  ← OpenWebUI 用户脚本 / OpenWebUI user script
├── SillyTavernSub/                ← SillyTavern 插件 / SillyTavern plugin
├── Agent/                         ← Agent 角色卡目录 / Agent character card directory
├── TVStxt/                        ← 外部变量文件 / External variable files
├── dailynote/                     ← 运行数据目录 / Runtime diary data (not source)
├── image/                         ← 运行媒体资源 / Runtime media assets (not source)
│
├── config.env.example             ← 主配置模板 / Main config template
├── docker-compose.yml             ← Docker Compose 部署 / Docker Compose deployment
└── Dockerfile                     ← Docker 镜像构建 / Docker image build
```

---

## 4. 核心三角架构 / Core Triangle

VCPToolBox 的核心是 **"AI–工具–记忆"** 铁三角：

```
              ┌──────────────────┐
              │   AI 推理引擎     │
              │ (上游 LLM API)    │
              │  AI Reasoning    │
              └────────┬─────────┘
                       │  VCP 协议 / VCP Protocol
              ┌────────┴─────────┐
              │   VCPToolBox     │  ← 中间层 / Middleware
              │  (server.js)     │
              └────────┬─────────┘
             ╱         │          ╲
            ╱          │           ╲
   ┌────────┴───┐  ┌───┴──────┐  ┌─┴──────────────┐
   │  工具执行   │  │  持久记忆  │  │  分布式调度     │
   │ Plugin.js  │  │  KBMgr   │  │ WebSocketServer │
   │ Tool Exec  │  │  Memory  │  │  Distributed    │
   └────────────┘  └──────────┘  └─────────────────┘
```

**三角关系说明 / Triangle Description**:

| 角 / Corner | 职责 / Role | 文件 / File |
|---|---|---|
| AI 推理 / AI Reasoning | 语言理解与生成 / Language understanding & generation | 上游 API / upstream LLM API |
| 工具执行 / Tool Execution | 79+ 插件，覆盖搜索/图像/代码等 / 79+ plugins: search, image, code... | `Plugin.js`, `Plugin/` |
| 持久记忆 / Persistent Memory | TagMemo RAG、向量检索、日记系统 / TagMemo RAG, vector search, diary | `KnowledgeBaseManager.js` |

---

## 5. 架构全景图 / Architecture Panorama

```
┌──────────────────────────────────────────────────────────────────┐
│                         客户端层 / Client Layer                   │
│   VCPChat  │  SillyTavern  │  OpenWebUI  │  自定义前端 / Custom  │
└───────────────────────────┬──────────────────────────────────────┘
                            │  HTTP / SSE / WebSocket
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                    server.js  (端口 5890 / Port 5890)             │
│  ┌────────────┐ ┌───────────────┐ ┌──────────────────────────┐  │
│  │ Express App│ │ 中间件链       │ │  路由层 / Routing Layer   │  │
│  │            │ │ Middleware    │ │  /v1/chat/completions     │  │
│  │ :5890      │ │ (Auth/CORS)   │ │  /v1/models               │  │
│  └────────────┘ └───────────────┘ │  /admin_api/*             │  │
│                                   │  /plugin-callback/*       │  │
│                                   └──────────────────────────┘  │
└──────────────┬──────────────────────────────────┬────────────────┘
               │                                  │
               ▼                                  ▼
┌──────────────────────────┐       ┌──────────────────────────────┐
│  modules/                │       │  WebSocketServer.js          │
│  chatCompletionHandler   │       │  (分布式通信 / Distributed    │
│  messageProcessor        │       │   Communication)             │
│  agentManager            │       └──────────┬───────────────────┘
└──────────┬───────────────┘                  │
           │                         ┌────────┴──────────┐
           ▼                         │  分布式节点集群    │
┌──────────────────┐                 │  Distributed Node │
│  Plugin.js       │◄───────────────►│  Cluster          │
│  (插件调度)       │                 └───────────────────┘
│  Plugin Dispatch │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────────────┐
│  Plugin/  (79 活跃 / active + 8 禁用/disabled)│
│  Node.js / Python / Rust plugins             │
└──────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────┐
│  KnowledgeBaseManager.js                     │
│  ┌──────────────┐ ┌───────────┐ ┌─────────┐ │
│  │ diaryIndices │ │ tagIndex  │ │ SQLite  │ │
│  │ (向量索引Map)│ │(全局Tag索引)│ │  WAL   │ │
│  └──────────────┘ └───────────┘ └─────────┘ │
│  EPA ─ ResidualPyramid ─ ResultDeduplicator  │
└──────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────┐
│  rust-vexus-lite/  (Rust N-API 向量引擎)      │
│  USearch-based KNN search                    │
└──────────────────────────────────────────────┘
```

---

## 6. 启动序列 / Boot Sequence

以下是 `node server.js` 的完整 **11 步启动序列**：

| 步骤 / Step | 描述 / Description | 阻塞 / Blocking | 📁 位置 / Location |
|---|---|---|---|
| 1 | 加载 `config.env` / Load config | ✅ 同步阻塞 | `server.js:4` |
| 2 | 初始化日志系统 / Init logger | ✅ 同步阻塞 | `server.js:20-22` |
| 3 | 解析 Agent 目录路径 / Resolve Agent dir | ⬜ 同步计算 | `server.js:27-66` |
| 4 | 创建 Express 应用 / Create Express app | ⬜ | `server.js:231` |
| 5 | 挂载中间件链 / Mount middleware chain | ⬜ | `server.js:236-456` |
| 6 | 注册路由层 / Register routes | ⬜ | `server.js:306-813` |
| 7 | 实例化 ChatCompletionHandler | ⬜ | `server.js:749-785` |
| 8 | 初始化 PluginManager / Init PluginManager | ✅ 异步等待 | `server.js:787-800` |
| 9 | 初始化 KnowledgeBaseManager | ✅ 异步等待 | `server.js:76-84` |
| 10 | 初始化 WebSocketServer | ✅ 异步等待 | `server.js:802-808` |
| 11 | `app.listen(:5890)` / Start listening | ✅ 最终门控 | `server.js:813-825` |

⚠️ **风险点 / Risk**: 步骤 8–10 均为异步阻塞，若任一初始化失败，`app.listen` 不会执行。

---

## 7. 核心模块详解 / Core Module Reference

### 7.1 `server.js` — HTTP/SSE 主入口

- **职责 / Role**: Express 应用创建、中间件挂载、路由注册、全局启动编排
- **端口 / Port**: `5890`（可通过 `PORT` 环境变量覆盖）
- **关键变量 / Key Vars**: `app`, `chatCompletionHandler`, `pluginManager`, `knowledgeBaseManager`
- **中断控制 / Interrupt**: `POST /v1/interrupt` → 终止进行中的 AI 流

### 7.2 `Plugin.js` — 插件生命周期总控

- **职责 / Role**: manifest 发现与加载、六类插件执行分发、服务路由注册、热重载清理
- **关键类 / Key Class**: `PluginManager`
- **执行入口 / Execution Entry**: `executePlugin(name, args, context)`
- 📁 `Plugin.js:388-564` (加载), `Plugin.js:632-720` (执行), `Plugin.js:805-1030` (服务路由)

### 7.3 `WebSocketServer.js` — 分布式 WS 骨架

- **职责 / Role**: 6 种 WS 客户端类型管理、消息路由、分布式工具调用闭环、文件获取
- **关键函数 / Key Functions**: `initialize()`, `broadcastVCPInfo()`, `dispatchToolToNode()`
- 📁 `WebSocketServer.js:52-57` (客户端分型), `WebSocketServer.js:400-465` (消息分发)

### 7.4 `KnowledgeBaseManager.js` — RAG/向量库总控

- **职责 / Role**: SQLite + 全局 Tag 索引 + diaryIndices + EPA + 残差金字塔 + 去重器
- **关键类 / Key Class**: `KnowledgeBaseManager` (单例 / Singleton)
- **索引结构 / Index**: `diaryIndices: Map<name, VexusIndex>`, `tagIndex: VexusIndex`
- 📁 `KnowledgeBaseManager.js:76-135` (init), `KnowledgeBaseManager.js:446-621` (search)

### 7.5 `modules/chatCompletionHandler.js` — 对话主流程

- **职责 / Role**: 接收 `/v1/chat/completions` 请求 → 调用 messageProcessor → 执行插件 → 转发上游 → 流式返回
- **关键方法 / Key Methods**: `handleChatCompletion()`, `processToolCalls()`

### 7.6 `modules/messageProcessor.js` — 变量替换管线

- **职责 / Role**: 解析并替换 `{{Agent*}}`, `{{Tar*}}`, `{{Var*}}`, `{{Sar*}}` 等占位符
- **外部文件加载 / File Loading**: 从 `TVStxt/*.txt` 加载变量值

### 7.7 `modules/agentManager.js` — Agent 别名映射

- **职责 / Role**: 加载 `agent_map.json`，维护 Agent 别名 → 角色卡文件路径映射，支持 chokidar 热更新

---

## 8. 聊天请求完整流程 / Chat Request Full Flow

```
客户端 POST /v1/chat/completions
         │
         ▼
[中间件] Bearer Token 验证 / IP 黑名单检查
         │
         ▼
[specialModelRouter] 是否为白名单特殊模型？
    ├── 是 → 直接透传上游 API
    └── 否 →
         ▼
[chatCompletionHandler.handleChatCompletion()]
         │
         ▼
[messageProcessor] 变量替换 / 占位符注入 / Agent 角色卡加载
         │
         ▼
[PluginManager] 执行 messagePreprocessor 类型插件
         │
         ▼
[static 插件占位符注入] {{AllCharacterDiariesData}} 等
         │
         ▼
[RAG 检索] KnowledgeBaseManager.search() (若 Agent 启用)
         │
         ▼
[转发上游 LLM API] fetch(apiUrl, {stream: true})
         │
         ▼
[流式读取响应] 检测 VCP 工具调用块
  <<<[TOOL_REQUEST]>>>
  toolName:「始」xxx「末」
  param:「始」value「末」
  <<<[END_TOOL_REQUEST]>>>
         │ 检测到工具调用
         ▼
[PluginManager.executePlugin()] 并行执行所有工具
         │
         ▼
[聚合工具结果] 注入回 AI 上下文
         │
         ▼
[继续流式输出] 或 [再次请求 LLM]
         │
         ▼
客户端 SSE 流 / Client SSE stream
```

---

## 9. 设计决策 / Design Decisions

| 决策 / Decision | 原因 / Reason | 影响 / Impact |
|---|---|---|
| 扁平根目录，无 `src/` | 快速定位文件，降低路径复杂度 / Fast file location, reduce path complexity | 大文件聚集在根目录 / Large files at root |
| VCP 自定义工具调用语法（非 JSON Function Calling）| 模型无关，支持流式容错 / Model-agnostic, supports streaming fault tolerance | 前端零侵入 / Zero frontend intrusion |
| SQLite + Rust 向量索引（非云 DB）| 离线可用，低延迟 / Offline capable, low latency | 需本地构建 Rust 组件 / Requires local Rust build |
| 插件即用即销（非常驻进程）| 零进程积压，按需资源分配 / Zero process accumulation, on-demand allocation | 启动延迟略高 / Slightly higher startup latency |
| CommonJS（非 ESM）| 历史兼容性 / Historical compatibility | 无法直接 `import` ESM 模块 / Cannot directly import ESM modules |

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ➡️ 下一节 / Next: [02_plugin_system/README.md](../02_plugin_system/README.md)
