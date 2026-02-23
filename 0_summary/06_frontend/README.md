# 06 前端组件 / Frontend Components

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心目录 / Core Dirs**: `AdminPanel/`, `VCPChrome/`, `OpenWebUISub/`, `SillyTavernSub/`  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [前端生态概览 / Frontend Ecosystem Overview](#1-前端生态概览--frontend-ecosystem-overview)
2. [AdminPanel 管理面板](#2-adminpanel-管理面板)
3. [VCPChrome 浏览器扩展](#3-vcpchrome-浏览器扩展)
4. [OpenWebUISub 用户脚本](#4-openwebuisub-用户脚本)
5. [SillyTavernSub 插件](#5-sillytavernsub-插件)
6. [前后端通信协议 / Frontend-Backend Communication](#6-前后端通信协议--frontend-backend-communication)

---

## 1. 前端生态概览 / Frontend Ecosystem Overview

VCPToolBox 前端由**四个独立组件**构成，覆盖不同使用场景：

| 组件 / Component | 类型 / Type | 技术栈 / Tech Stack | 主要功能 / Main Function |
|---|---|---|---|
| `AdminPanel` | 内嵌静态前端 / Embedded static | 原生 JS/CSS, EasyMDE | 服务器管理、配置、监控 |
| `VCPChrome` | Chrome 扩展 / Chrome extension | Manifest V3, Service Worker | 浏览器控制、页面采集 |
| `OpenWebUISub` | 用户脚本 / Userscript | Tampermonkey/Greasemonkey | VCP 协议渲染、OpenWebUI 增强 |
| `SillyTavernSub` | ST 扩展插件 / ST extension | JavaScript | SillyTavern UI 增强、VCP 通知 |

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户界面层 / UI Layer                    │
├────────────────┬─────────────────┬──────────────────────────────┤
│  AdminPanel    │   VCPChrome     │  OpenWebUISub / SillyTavern  │
│  :5890/Admin.. │  Chrome ext     │  Userscript injection        │
│                │                 │                              │
│ - 系统监控      │ - 页面监控      │ - ToolCall 渲染               │
│ - 配置管理      │ - 远程控制      │ - DailyNote 渲染              │
│ - 插件管理      │ - 信息采集      │ - 图片渲染增强                │
│ - 知识库编辑    │                 │ - 侧边栏面板                  │
└───────┬────────┴────────┬────────┴─────────────┬────────────────┘
        │                 │                       │
        │ HTTP/WS         │ WebSocket             │ GM_xmlhttpRequest
        │ /admin_api      │ /vcp-chrome-*         │ 跨域代理
        ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│               VCPToolBox 后端服务 / Backend Service             │
│                   (Node.js + Express)                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. AdminPanel 管理面板

> 📁 `AdminPanel/`  
> 访问地址 / Access URL: `http://host:5890/AdminPanel/`  
> 认证 / Auth: Basic Auth (AdminUsername / AdminPassword)

### 2.1 架构特点 / Architecture Characteristics

AdminPanel 是**内嵌静态前端**，直接由 Express 以 `express.static('AdminPanel/')` 托管：

- ✅ 无需独立前端服务 / No separate frontend server required
- ✅ 无打包工具依赖（Webpack/Vite）/ No bundler dependency
- ✅ 原生 JavaScript 模块化设计 / Native JS modular design
- ⚠️ 不是 SPA，路由由服务端控制 / Not SPA, routing server-controlled

### 2.2 目录结构 / Directory Structure

```
AdminPanel/
├── index.html                     ← 主页面入口 / Main page entry
├── login.html                     ← 登录页面 / Login page
├── script.js                      ← 前端路由与分区切换 / Frontend routing
├── style.css                      ← 主样式（CSS 变量驱动）/ Main styles
├── woff.css                       ← 字体样式 / Font styles
│
├── js/                            ← 前端业务模块 / Frontend modules
│   ├── utils.js                   ← apiFetch、showMessage 等工具函数
│   ├── config.js                  ← 配置解析与表单构建
│   ├── dashboard.js               ← 仪表盘（系统监控）
│   ├── plugins.js                 ← 插件列表与配置管理
│   ├── notes-manager.js           ← 日记知识库管理
│   ├── agent-manager.js           ← Agent 映射管理
│   ├── agent-assistant-config.js  ← AgentAssistant 配置
│   ├── tvs-editor.js              ← 高级变量编辑器（TVStxt）
│   ├── forum.js                   ← VCP 论坛界面
│   ├── model-redirect.js          ← 模型重定向配置
│   └── log-viewer.js              ← 日志查看器
│
└── lib/                           ← 第三方库（本地化）
    ├── easymde.min.js             ← Markdown 编辑器
    └── ...
```

### 2.3 主要功能模块 / Main Feature Modules

| 模块 / Module | 文件 / File | 功能 / Function |
|---|---|---|
| 仪表盘 / Dashboard | `dashboard.js` | CPU/内存/运行时间监控，实时 VCP 日志流 |
| 配置管理 / Config | `config.js` | 读取/修改 `config.env`，热重载 |
| 插件管理 / Plugins | `plugins.js` | 插件列表、启用/禁用、配置编辑 |
| 日记管理 / Notes | `notes-manager.js` | 知识库 CRUD、Tag 管理、向量索引重建 |
| Agent 管理 / Agents | `agent-manager.js` | `agent_map.json` 编辑，角色卡管理 |
| 变量编辑器 / TVS | `tvs-editor.js` | `TVStxt/*.txt` 外部变量文件编辑 |
| 论坛 / Forum | `forum.js` | 查看/发布/回复帖子 |
| 日志查看器 / Logs | `log-viewer.js` | 实时日志流，历史日志查询 |

---

## 3. VCPChrome 浏览器扩展

> 📁 `VCPChrome/`  
> 标准: Manifest V3 Chrome Extension

### 3.1 核心能力 / Core Capabilities

| 能力 / Capability | 说明 / Description |
|---|---|
| 页面内容采集 / Page content capture | 将当前页面转换为 AI 可读的 Markdown 格式 |
| 截图 / Screenshot | 当前页面截图并传输给服务器 |
| 链接提取 / Link extraction | 提取页面所有超链接 |
| 页面控制 / Page control | AI 远程指令打开 URL、点击元素 |
| 信息注入 / Info injection | 向 VCPToolBox 实时推送浏览器状态 |

### 3.2 架构 / Architecture

```
Chrome 浏览器 / Chrome Browser
├── background.js (Service Worker)  ← 持久化 WS 连接管理
├── content.js                       ← 页面内容采集与 DOM 操作
├── popup.js / popup.html            ← 扩展图标弹出界面
└── manifest.json                    ← Manifest V3 配置

         │ WebSocket 长连接
         ▼
VCPToolBox WebSocketServer
  /vcp-chrome-observer/VCP_Key=<key>   ← 数据上报通道
  /vcp-chrome-control/VCP_Key=<key>    ← 远程控制指令通道
```

### 3.3 使用场景 / Use Cases

```
场景 / Scenario:
AI 需要读取当前浏览器页面内容

流程 / Flow:
1. AI 调用 ChromeBridge 工具
2. VCPToolBox 通过 WS 向扩展发送 "capture_page" 命令
3. content.js 采集页面文本、图片、链接
4. 内容通过 WS 回传 VCPToolBox
5. VCPToolBox 将内容格式化后注入 AI 上下文
```

---

## 4. OpenWebUISub 用户脚本

> 📁 `OpenWebUISub/`  
> 运行方式: Tampermonkey / Greasemonkey 用户脚本

### 4.1 功能 / Function

在 OpenWebUI 等前端界面注入 VCP 专属渲染逻辑，增强用户体验：

| 增强项 / Enhancement | 说明 / Description |
|---|---|
| VCP ToolCall 渲染 | 将 `<<<[TOOL_REQUEST]>>>` 块渲染为可折叠的 UI 卡片 |
| DailyNote 渲染 | 将日记数据渲染为时间线格式 |
| 图片渲染增强 | 自动识别图片 URL 并内联展示 |
| 侧边栏面板 | 注入 VCP 功能侧边栏（记忆可视化等） |

### 4.2 跨域通信 / Cross-Origin Communication

```javascript
// GM_xmlhttpRequest 绕过浏览器同源策略
GM_xmlhttpRequest({
    method: 'POST',
    url: 'http://vcptoolbox-server:5890/v1/chat/completions',
    headers: { 'Authorization': 'Bearer key' },
    data: JSON.stringify(payload),
    onload: (response) => { /* handle */ }
});
```

---

## 5. SillyTavernSub 插件

> 📁 `SillyTavernSub/`  
> 运行方式: SillyTavern 扩展插件

### 5.1 功能 / Function

| 功能 / Function | 说明 / Description |
|---|---|
| VCP 通知接收 | 在 ST 通知栏显示工具调用日志 |
| DailyNote 渲染 | ST 内的日记内容可视化 |
| VCP 工具调用可视化 | 在对话中显示工具调用详情卡片 |
| AgentList 支持 | ST 内 Agent 列表管理 |

---

## 6. 前后端通信协议 / Frontend-Backend Communication

### AdminPanel 通信

```javascript
// AdminPanel/js/utils.js — apiFetch 封装
async function apiFetch(endpoint, options = {}) {
    const response = await fetch(`/admin_api/${endpoint}`, {
        headers: {
            'Authorization': 'Basic ' + btoa(`${username}:${password}`),
            'Content-Type': 'application/json',
            ...options.headers
        },
        ...options
    });
    return response.json();
}
```

### 实时日志推送 / Real-time Log Push

```javascript
// AdminPanel 通过 WebSocket 接收实时日志
const ws = new WebSocket(`ws://host:5890/VCPlog/VCP_Key=${key}`);
ws.onmessage = (event) => {
    const log = JSON.parse(event.data);
    // 渲染到日志面板
};
```

### OpenWebUI 跨域代理 / OpenWebUI Cross-Origin Proxy

```javascript
// OpenWebUISub 用户脚本通过 GM_xmlhttpRequest 突破同源限制
GM_xmlhttpRequest({
    method: 'GET',
    url: `${VCP_SERVER}/v1/models`,
    headers: { 'Authorization': `Bearer ${apiKey}` },
    onload: (res) => handleModels(JSON.parse(res.responseText))
});
```

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ⬅️ 上一节 / Prev: [05_distributed](../05_distributed/README.md) | ➡️ 下一节 / Next: [07_config_deploy](../07_config_deploy/README.md)
