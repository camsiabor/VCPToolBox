# 04 API 路由与认证 / API Routes & Authentication

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `server.js`, `routes/specialModelRouter.js`, `routes/adminPanelRoutes.js`, `routes/forumApi.js`  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [路由架构总览 / Route Architecture Overview](#1-路由架构总览--route-architecture-overview)
2. [端口与基础配置 / Port & Base Config](#2-端口与基础配置--port--base-config)
3. [中间件链 / Middleware Chain](#3-中间件链--middleware-chain)
4. [认证机制 / Authentication Mechanisms](#4-认证机制--authentication-mechanisms)
5. [核心 API 端点 / Core API Endpoints](#5-核心-api-端点--core-api-endpoints)
6. [特殊模型路由 / Special Model Router](#6-特殊模型路由--special-model-router)
7. [管理面板 API / Admin Panel API](#7-管理面板-api--admin-panel-api)
8. [论坛 API / Forum API](#8-论坛-api--forum-api)
9. [插件回调端点 / Plugin Callback Endpoint](#9-插件回调端点--plugin-callback-endpoint)
10. [媒体服务端点 / Media Service Endpoints](#10-媒体服务端点--media-service-endpoints)
11. [路由注册 / Route Registration](#11-路由注册--route-registration)

---

## 1. 路由架构总览 / Route Architecture Overview

```
server.js (Express App)
├── [特殊模型] routes/specialModelRouter.js   → /v1/chat/completions, /v1/embeddings (whitelist)
├── [管理面板] routes/adminPanelRoutes.js      → /admin_api/*
│   └── routes/forumApi.js                    → /admin_api/forum/*
├── [核心 AI]  内联路由 / Inline routes        → /v1/*
├── [插件回调] 内联路由                         → /plugin-callback/*
├── [媒体服务] 内联路由                         → /pw=:key/images/*, /pw=:key/files/*
└── [静态文件] AdminPanel/                     → /AdminPanel/*
```

### 端点分类速查 / Endpoint Category Quick Lookup

| 类别 / Category | 前缀 / Prefix | 认证方式 / Auth | 说明 / Description |
|---|---|---|---|
| 核心 AI API | `/v1/*` | Bearer Token | AI 对话、模型列表等 |
| 特殊模型透传 | `/v1/chat/completions`, `/v1/embeddings` | Bearer Token | 白名单模型直通 |
| 管理面板 API | `/admin_api/*` | Basic Auth | 系统管理、配置控制 |
| 管理面板静态资源 | `/AdminPanel/*` | Basic Auth | Web 管理界面 HTML/JS/CSS |
| 论坛 API | `/admin_api/forum/*` | Basic Auth | Agent 论坛交互 |
| 插件回调 | `/plugin-callback/*` | 无 / None | 异步插件结果回调 |
| 图片服务 | `/pw=:key/images/*` | URL 密钥 / URL key | 图片托管服务 |
| 文件服务 | `/pw=:key/files/*` | URL 密钥 / URL key | 文件托管服务 |

---

## 2. 端口与基础配置 / Port & Base Config

| 参数 / Parameter | 默认值 / Default | 配置项 / Config Key |
|---|---|---|
| HTTP 端口 / HTTP Port | `5890` | `PORT` in `config.env` |
| 请求体大小限制 / Body size limit | `300mb` | 硬编码 / Hardcoded |
| CORS | `origin: '*'` | 硬编码 / Hardcoded |
| Trust proxy | `true` | 硬编码 / Hardcoded |

---

## 3. 中间件链 / Middleware Chain

请求按以下顺序经过中间件（📁 `server.js:236-456`）：

```
1. trust proxy
   app.set('trust proxy', true)
   └── 解析 X-Forwarded-For，获取真实 IP

2. CORS
   cors({ origin: '*' })

3. 请求体解析 / Body parsing
   ├── express.json({ limit: '300mb' })
   ├── express.urlencoded({ limit: '300mb', extended: true })
   └── express.text({ limit: '300mb', type: 'text/plain' })

4. IP 追踪 / IP tracking
   └── 记录 POST 请求来源 IP，写入日志

5. IP 黑名单 / IP blacklist
   └── 读取 ip_blacklist.json，阻止黑名单 IP → 403

6. 特殊模型路由 / Special model router
   └── 白名单模型拦截，直接透传上游 API

7. Admin 认证 / Admin auth (adminAuth middleware)
   └── 保护 /admin_api 和 /AdminPanel 路径 → Basic Auth

8. Admin 静态文件 / Admin static files
   └── express.static('AdminPanel/')

9. Bearer Token 认证 / Bearer token auth
   └── 保护 /v1/* 路径 → Authorization: Bearer <KEY>
```

---

## 4. 认证机制 / Authentication Mechanisms

### 4.1 Bearer Token（核心 AI API）

保护路径 / Protected paths: `/v1/*`

```
请求头 / Request header:
  Authorization: Bearer <VCP_KEY>

配置 / Config:
  Key=mySecretKey  (config.env)

验证逻辑 / Validation logic (server.js:466-493):
  token = req.headers.authorization?.split(' ')[1]
  if token !== process.env.Key → 401 Unauthorized
```

### 4.2 Basic Auth（管理面板）

保护路径 / Protected paths: `/admin_api/*`, `/AdminPanel/*`

```
请求头 / Request header:
  Authorization: Basic base64(username:password)

配置 / Config:
  AdminUsername=admin    (config.env)
  AdminPassword=secret   (config.env)
```

### 4.3 URL 密钥（媒体服务）

保护路径 / Protected paths: `/pw=:key/images/*`, `/pw=:key/files/*`

```
URL 格式 / URL format:
  /pw=mySecretKey/images/photo.jpg

说明 / Note:
  密钥嵌入 URL 路径段，避免暴露在请求头中
  Key embedded in URL path segment, not in headers
```

⚠️ **安全注意 / Security Note**: URL 密钥会出现在服务器日志和 Referer 头中，仅用于非敏感媒体托管。

---

## 5. 核心 API 端点 / Core API Endpoints

### `POST /v1/chat/completions`

**认证 / Auth**: Bearer Token  
**功能 / Function**: AI 聊天主入口，处理完整的 VCP 流程  
**请求体 / Request body**: OpenAI 兼容格式

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "Hello!" }
  ],
  "stream": true,
  "temperature": 0.7
}
```

**处理流程 / Processing flow**: 见 [01_architecture § 聊天请求完整流程](../01_architecture/README.md#8-聊天请求完整流程--chat-request-full-flow)

**响应 / Response**: SSE 流（`stream: true`）或 JSON（`stream: false`）

---

### `GET /v1/models`

**认证 / Auth**: Bearer Token  
**功能 / Function**: 返回可用模型列表（从上游 API 获取 + 本地补充）

---

### `POST /v1/interrupt`

**认证 / Auth**: Bearer Token  
**功能 / Function**: 中断当前进行中的 AI 流式响应  
**请求体 / Request body**: `{ "conversationId": "..." }`

---

### `POST /v1/human/tool`

**认证 / Auth**: Bearer Token  
**功能 / Function**: 人类（而非 AI）直接调用 VCP 工具插件  
**请求体 / Request body**:
```json
{
  "toolName": "GoogleSearch",
  "params": { "query": "VCPToolBox" }
}
```

---

## 6. 特殊模型路由 / Special Model Router

> 📁 `routes/specialModelRouter.js`

### 功能 / Function

某些模型（如 Embedding 模型、图像生成模型）不需要 VCP 工具调用处理，应直接透传给上游 API。特殊模型路由实现这一**白名单透传**机制。

### 白名单配置 / Whitelist Config

```javascript
// routes/specialModelRouter.js (推断结构 / Inferred structure)
const SPECIAL_MODEL_WHITELIST = [
  'text-embedding-*',    // Embedding 模型
  'dall-e-*',            // DALL-E 图像模型
  // ... 更多白名单模式
];
```

```json
// ModelRedirect.json (模型名称重定向)
{
  "my-local-model": "gpt-4o",
  "embedding-model": "text-embedding-3-small"
}
```

### 透传流程 / Pass-through Flow

```
请求到达 /v1/chat/completions
  │
  ├── 模型名在白名单？
  │   ├── 是 → specialModelRouter 直接 fetch 上游 API → 返回原始响应
  │   └── 否 → 交给主 chatCompletionHandler 处理（VCP 流程）
```

---

## 7. 管理面板 API / Admin Panel API

> 📁 `routes/adminPanelRoutes.js`  
> **认证 / Auth**: Basic Auth (AdminUsername / AdminPassword)

### 主要端点 / Main Endpoints

| 端点 / Endpoint | 方法 / Method | 功能 / Function |
|---|---|---|
| `/admin_api/config` | GET | 获取当前配置 / Get current config |
| `/admin_api/config` | POST | 更新配置并热重载 / Update config and hot-reload |
| `/admin_api/plugins` | GET | 列出所有插件及状态 / List all plugins and status |
| `/admin_api/plugins/:name/toggle` | POST | 启用/禁用插件 / Enable/disable plugin |
| `/admin_api/plugins/:name/config` | GET/POST | 获取/更新插件配置 / Get/update plugin config |
| `/admin_api/system/restart` | POST | 重启服务器 / Restart server |
| `/admin_api/system/status` | GET | 获取系统状态 / Get system status |
| `/admin_api/knowledge-base` | GET | 知识库状态 / Knowledge base status |
| `/admin_api/agents` | GET/POST | Agent 映射管理 / Agent mapping management |
| `/admin_api/logs` | GET | 获取日志 / Get logs |

---

## 8. 论坛 API / Forum API

> 📁 `routes/forumApi.js`  
> **挂载路径 / Mount path**: `/admin_api/forum/*`  
> **认证 / Auth**: Basic Auth（继承管理面板认证）

VCP 论坛是 Agent 的**内网社交平台**，允许不同 Agent 发帖、回复、阅读帖子，实现 Agent 间知识共享。

| 端点 / Endpoint | 方法 / Method | 功能 / Function |
|---|---|---|
| `/admin_api/forum/posts` | GET | 获取帖子列表 / Get post list |
| `/admin_api/forum/posts` | POST | 发布新帖子 / Create new post |
| `/admin_api/forum/posts/:id` | GET | 获取帖子详情 / Get post detail |
| `/admin_api/forum/posts/:id/reply` | POST | 回复帖子 / Reply to post |

---

## 9. 插件回调端点 / Plugin Callback Endpoint

> 📁 `server.js`（内联路由）  
> **认证 / Auth**: 无（内部调用）

异步插件完成任务后，通过回调端点将结果返回给等待的 AI 对话。

```
POST /plugin-callback/:callbackId

请求体 / Request body:
{
  "result": "...",   // 插件执行结果
  "success": true,
  "error": null
}
```

**安全注意 / Security Note**: ⚠️ 此端点无认证，依赖 `callbackId` 的随机性保证安全。`callbackId` 应使用加密随机字符串（如 `crypto.randomUUID()`）。

---

## 10. 媒体服务端点 / Media Service Endpoints

### 图片服务 / Image Service

```
GET /pw=:key/images/:filename

示例 / Example:
  GET /pw=myKey123/images/generated_photo.png
```

### 文件服务 / File Service

```
GET /pw=:key/files/:filename

示例 / Example:
  GET /pw=myKey123/files/document.pdf
```

这两个服务由对应的插件（`ImageServer`, `FileServer`）在 `hybridservice` 类型下注册路由。

---

## 11. 路由注册 / Route Registration

新增路由的推荐方式：

### 方式 A：内联路由（简单场景）

```javascript
// server.js
app.get('/v1/my-endpoint', authMiddleware, (req, res) => {
    res.json({ message: 'Hello' });
});
```

### 方式 B：独立路由文件（复杂场景）

```javascript
// routes/myNewRoutes.js
const express = require('express');
const router = express.Router();

router.get('/my-endpoint', (req, res) => {
    res.json({ data: 'example' });
});

module.exports = router;
```

```javascript
// server.js — 挂载路由
const myNewRouter = require('./routes/myNewRoutes');
app.use('/admin_api', adminAuth, myNewRouter);  // 加认证
// 或
app.use(myNewRouter);  // 不加认证
```

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ⬅️ 上一节 / Prev: [03_memory_rag](../03_memory_rag/README.md) | ➡️ 下一节 / Next: [05_distributed](../05_distributed/README.md)
