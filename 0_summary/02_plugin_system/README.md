# 02 插件生态系统 / Plugin Ecosystem

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `Plugin.js`, `Plugin/<PluginName>/plugin-manifest.json`  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [概述 / Overview](#1-概述--overview)
2. [VCP 工具调用语法 / VCP Tool-Call Syntax](#2-vcp-工具调用语法--vcp-tool-call-syntax)
3. [六类插件类型 / Six Plugin Types](#3-六类插件类型--six-plugin-types)
4. [Manifest Schema 规范 / Manifest Schema Spec](#4-manifest-schema-规范--manifest-schema-spec)
5. [插件生命周期 / Plugin Lifecycle](#5-插件生命周期--plugin-lifecycle)
6. [配置级联机制 / Config Cascading](#6-配置级联机制--config-cascading)
7. [通信协议 / Communication Protocols](#7-通信协议--communication-protocols)
8. [插件分类索引 / Plugin Category Index](#8-插件分类索引--plugin-category-index)
9. [快速开发指南 / Quick Dev Guide](#9-快速开发指南--quick-dev-guide)

---

## 1. 概述 / Overview

### 中文

插件系统是 VCPToolBox 的**核心能力扩展层**。每个插件通过一个 `plugin-manifest.json` 文件声明自身的类型、执行方式与能力。`PluginManager`（`Plugin.js`）负责在启动时发现、加载全部插件，并在运行时按需调度。

- **总量统计**: 79 个活跃插件 + 8 个禁用插件（`.block` 后缀）
- **多语言**: Node.js / Python / Rust 混合架构
- **热加载**: stdio 协议插件支持文件变更自动重载

### English

The plugin system is the **core capability extension layer** of VCPToolBox. Each plugin declares its type, execution method, and capabilities via a `plugin-manifest.json` file. `PluginManager` (`Plugin.js`) discovers and loads all plugins at startup and dispatches them on demand.

- **Total**: 79 active plugins + 8 disabled plugins (`.block` suffix)
- **Multi-runtime**: Node.js / Python / Rust mixed architecture
- **Hot-reload**: stdio-protocol plugins support automatic file-change reloading

---

## 2. VCP 工具调用语法 / VCP Tool-Call Syntax

VCPToolBox 使用**自定义文本标记语法**（非 JSON Function Calling）调用工具，设计目标是对 AI 模型友好、模型无关、流式容错。

```
<<<[TOOL_REQUEST]>>>
toolName:「始」SearchTool「末」
query:「始」Node.js async patterns「末」
maxResults:「始」5「末」
<<<[END_TOOL_REQUEST]>>>
```

### 语法规则 / Syntax Rules

| 元素 / Element | 格式 / Format | 说明 / Description |
|---|---|---|
| 块开始 / Block start | `<<<[TOOL_REQUEST]>>>` | 工具调用开始标记 / Tool call start marker |
| 块结束 / Block end | `<<<[END_TOOL_REQUEST]>>>` | 工具调用结束标记 / Tool call end marker |
| 参数值包裹 / Param wrap | `key:「始」value「末」` | 使用中文书名号包裹，支持多行值 / Chinese delimiters support multiline values |
| 工具名 / Tool name | `toolName:「始」PluginName「末」` | 必须与 manifest 中的 `name` 字段匹配 / Must match `name` in manifest |
| 并发调用 / Parallel calls | 多个块连续出现 / Multiple blocks in sequence | 服务器并发执行 / Server executes concurrently |

### 优势对比 MCP / Advantages Over MCP

| 特性 / Feature | VCP | JSON Function Calling / MCP |
|---|---|---|
| 模型依赖 / Model dependency | ❌ 无需 / None | ✅ 需要 FunctionTool 字段 / Requires FunctionTool field |
| 流式容错 / Stream fault tolerance | ✅ 文本截断仍可解析 | ❌ JSON 截断导致解析失败 |
| 并行调用 / Parallel calls | ✅ 单次输出多块 | ❌ 单步串行 |
| 前端侵入 / Frontend intrusion | ❌ 零侵入 | ✅ 前端需深度参与 |

---

## 3. 六类插件类型 / Six Plugin Types

### 类型总览 / Type Overview

| 类型 / Type | 数量 / Count | 触发方式 / Trigger | 典型用途 / Typical Use |
|---|---|---|---|
| `static` | ~10 | cron 定时 / cron schedule | 注入全局占位符 / Inject global placeholders |
| `messagePreprocessor` | ~5 | 每次对话前 / Before each chat | 预处理消息体 / Preprocess message body |
| `synchronous` | ~45 | AI 显式调用 / AI explicit call | 执行工具并立即返回 / Execute tool, return immediately |
| `asynchronous` | ~3 | AI 调用后台运行 / AI call, background run | 长耗时任务 / Long-running tasks |
| `service` | ~8 | 启动时常驻 / Resident at startup | HTTP 服务、定时任务 / HTTP services, scheduled tasks |
| `hybridservice` | ~8 | 常驻 + 可调用 / Resident + callable | 复合型服务 / Composite services |

### 3.1 `static` — 静态插件

**触发 / Trigger**: cron 表达式定时刷新  
**输出 / Output**: 存储在 `staticPlaceholderValues` Map，通过 `{{Placeholder}}` 注入 system prompt  
**示例 / Example**: `DailyNoteGet` — 周期读取日记内容注入 `{{AllCharacterDiariesData}}`

```json
{
  "pluginType": "static",
  "capabilities": {
    "systemPromptPlaceholders": [
      { "placeholder": "{{AllCharacterDiariesData}}", "description": "所有角色日记 JSON" }
    ]
  },
  "cron": "*/30 * * * *"
}
```

### 3.2 `messagePreprocessor` — 消息预处理插件

**触发 / Trigger**: 每次 `/v1/chat/completions` 请求前  
**能力 / Capability**: 修改 `messages` 数组（追加、删除、替换）  
**示例 / Example**: `WorkspaceInjector` — 注入工作区上下文信息

### 3.3 `synchronous` — 同步工具插件

**触发 / Trigger**: AI 输出 VCP 工具调用块  
**流程 / Flow**:  
`AI 调用 → spawn 子进程 → stdout 读取 → 结果注入上下文 → AI 继续`  
**超时 / Timeout**: manifest `communication.timeout` 字段（默认 30000ms）  
**示例 / Example**: `GoogleSearch`, `UrlFetch`, `LinuxShellExecutor`

### 3.4 `asynchronous` — 异步任务插件

**触发 / Trigger**: AI 调用后立即返回 `ack`，任务后台执行  
**回调 / Callback**: 完成后调用 `POST /plugin-callback/<callbackId>`  
**示例 / Example**: `VideoGenerator`, `SunoGen`

### 3.5 `service` — 常驻服务插件

**触发 / Trigger**: 服务器启动时 `pm2-like` 方式启动，持续运行  
**通信 / Communication**: HTTP / WebSocket / stdio  
**示例 / Example**: `VCPForum`, `DailyHot`

### 3.6 `hybridservice` — 混合服务插件

**触发 / Trigger**: 常驻 + 可被 AI 显式调用  
**典型场景 / Scenario**: 既提供常驻 HTTP 接口，又响应 AI 工具调用  
**示例 / Example**: `ImageServer`, `FileServer`

---

## 4. Manifest Schema 规范 / Manifest Schema Spec

完整的 `plugin-manifest.json` 字段说明：

```jsonc
{
  // ── 元数据 / Metadata ──────────────────────────────────────────
  "manifestVersion": "1.0.0",      // manifest 格式版本
  "name": "MyPlugin",              // 插件唯一标识符（必需）/ Unique plugin ID (required)
  "version": "1.0.0",              // 插件版本号
  "displayName": "我的插件",       // 显示名称（管理面板用）
  "description": "...",            // 功能描述（注入 AI system prompt）
  "author": "YourName",

  // ── 类型 / Type ────────────────────────────────────────────────
  "pluginType": "synchronous",     // static|messagePreprocessor|synchronous|asynchronous|service|hybridservice

  // ── 执行入口 / Entry Point ────────────────────────────────────
  "entryPoint": {
    "type": "nodejs",              // nodejs | python | shell | rust
    "command": "node index.js"     // 启动命令
  },

  // ── 通信协议 / Communication Protocol ────────────────────────
  "communication": {
    "protocol": "stdio",           // stdio | direct | distributed
    "timeout": 30000               // 超时毫秒数（synchronous 插件）
  },

  // ── 能力声明 / Capabilities ───────────────────────────────────
  "capabilities": {
    // 静态占位符（static 插件）
    "systemPromptPlaceholders": [
      { "placeholder": "{{MyData}}", "description": "描述" }
    ],
    // 工具描述（synchronous 插件，注入 AI 的 tool list）
    "tools": [
      {
        "name": "MyTool",
        "description": "工具功能描述（AI 看到的说明）",
        "parameters": {
          "param1": { "type": "string", "description": "参数说明" }
        }
      }
    ]
  },

  // ── 定时刷新（static 插件）────────────────────────────────────
  "cron": "*/30 * * * *",

  // ── 配置 Schema（驱动管理面板表单）────────────────────────────
  "configSchema": {
    "API_KEY": { "type": "string", "secret": true, "description": "API 密钥" },
    "MAX_RESULTS": { "type": "number", "default": 5 }
  }
}
```

### 禁用插件 / Disabled Plugins

- 启用: `plugin-manifest.json` 存在
- 禁用: 重命名为 `plugin-manifest.json.block`

当前 **8 个禁用插件** (`.block` 标记):  
`PowerShellExecutor`, `PyCameraCapture`, 及其他 6 个

---

## 5. 插件生命周期 / Plugin Lifecycle

```
服务器启动 / Server Startup
         │
         ▼
PluginManager.loadPlugins()
  ├── 遍历 Plugin/ 目录 / Scan Plugin/ dir
  ├── 过滤 plugin-manifest.json.block / Skip .block files
  ├── 解析 plugin-manifest.json / Parse manifest
  ├── 合并配置（全局 + 插件 config.env）/ Merge config
  └── 按类型注册插件 / Register by type
         │
         ▼
插件就绪 / Plugins Ready
  ├── static 插件: 启动 cron 定时器 / Start cron timer
  ├── service/hybridservice: spawn 常驻进程 / Spawn resident process
  └── synchronous/asynchronous: 等待调用 / Wait for invocation
         │
         ▼
运行时调用 / Runtime Invocation
  ├── AI 输出 VCP 块 / AI outputs VCP block
  ├── 解析工具名与参数 / Parse tool name & params
  ├── PluginManager.executePlugin(name, params)
  └── 根据 protocol 分发 / Dispatch by protocol:
       ├── stdio: spawn 子进程 / Spawn subprocess
       ├── direct: 直接调用模块函数 / Call module function directly
       └── distributed: 通过 WebSocket 路由到远程节点 / Route to remote node via WS
         │
         ▼
结果返回 / Result Return
  ├── 同步: stdout 捕获 → JSON 解析 → 注入上下文
  └── 异步: 立即返回 ack → 后台回调 /plugin-callback/<id>
```

---

## 6. 配置级联机制 / Config Cascading

```
优先级（高→低）/ Priority (high→low):
  插件 config.env  >  全局 config.env  >  manifest configSchema 默认值
  Plugin config.env > Global config.env > Manifest configSchema defaults
```

### 配置加载流程 / Config Loading Flow

```javascript
// Plugin.js (推断 / Inferred)
const globalConfig = loadDotenv('config.env');
const pluginConfig = loadDotenv(`Plugin/${name}/config.env`);
const merged = { ...globalConfig, ...pluginConfig };  // 插件覆盖全局
```

### 敏感配置 / Sensitive Config

- `Plugin/*/config.env` — 包含 API 密钥，**不提交到 git**
- `Plugin/*/config.env.example` — 模板文件，提交到 git

---

## 7. 通信协议 / Communication Protocols

### 7.1 `stdio` — 标准 I/O 协议（最常用）

```
PluginManager ─── spawn('node index.js') ───► 插件进程
                                               │ stdout (JSON)
PluginManager ◄──────────────────────────────┘
```

- 插件从 `stdin` 读取参数（JSON 格式）
- 插件向 `stdout` 写入结果（JSON 格式）
- 支持热重载（文件变更 → 下次调用时重新 spawn）

### 7.2 `direct` — 直接模块调用

```
PluginManager ─── require('./Plugin/MyPlugin/index.js') ───► module.exports.execute(params)
```

- 在同一 Node.js 进程中执行，无子进程开销
- ⚠️ 不支持热重载（需重启服务器）

### 7.3 `distributed` — 分布式远程调用

```
PluginManager ──► WebSocketServer ──► 分布式节点 / Distributed Node
                                      └── 本地 spawn 子进程
                                      └── 结果通过 WS 回传
```

- 用于将工具路由到特定硬件节点（如 GPU 服务器）
- 通过 `WebSocketServer.js` 实现透明路由

---

## 8. 插件分类索引 / Plugin Category Index

### 搜索与信息 / Search & Information

| 插件 / Plugin | 类型 / Type | 说明 / Description |
|---|---|---|
| `GoogleSearch` | synchronous | Google 搜索 / Google search |
| `TavilySearch` | synchronous | Tavily AI 搜索 / Tavily AI search |
| `SerpSearch` | synchronous | SerpAPI 搜索 |
| `UrlFetch` | synchronous | 网页内容抓取 / Webpage content fetch |
| `ArxivDailyPapers` | static | ArXiv 每日论文 / Daily ArXiv papers |
| `PubMedSearch` | synchronous | PubMed 医学搜索 |
| `KEGGSearch` | synchronous | KEGG 生物信息搜索 |
| `DailyHot` | service | 56+ 热点源聚合 / 56+ hot topic sources |

### 图像与多媒体 / Image & Media

| 插件 / Plugin | 类型 / Type | 说明 / Description |
|---|---|---|
| `ComfyUIGen` | synchronous | ComfyUI 工作流图像生成 |
| `FluxGen` | synchronous | Flux 模型图像生成 |
| `NovelAIGen` | synchronous | NovelAI 图像生成 |
| `GeminiImageGen` | synchronous | Gemini 图像生成 |
| `QwenImageGen` | synchronous | Qwen 图像生成 |
| `VideoGenerator` | asynchronous | 视频生成 |
| `SunoGen` | asynchronous | AI 音乐生成 |
| `ImageProcessor` | synchronous | 图像处理 |
| `MIDITranslator` | synchronous | MIDI 音乐翻译 |

### 文件系统 / File System

| 插件 / Plugin | 类型 / Type | 说明 / Description |
|---|---|---|
| `FileOperator` | synchronous | 文件读写操作 |
| `FileServer` | hybridservice | 文件托管服务 |
| `ImageServer` | hybridservice | 图片托管服务 |
| `FileListGenerator` | synchronous | 文件列表生成 |
| `FileTreeGenerator` | synchronous | 文件树生成 |
| `TencentCOSBackup` | synchronous | 腾讯云对象存储备份 |

### 系统执行 / System Execution

| 插件 / Plugin | 类型 / Type | 说明 / Description |
|---|---|---|
| `LinuxShellExecutor` | synchronous | ⚠️ Linux shell 命令执行（安全敏感）|
| `PowerShellExecutor` | synchronous | ⚠️ PowerShell 执行（禁用）|
| `PyCameraCapture` | synchronous | ⚠️ 摄像头捕获（禁用）|
| `PyScreenshot` | synchronous | 截图功能 |
| `LinuxLogMonitor` | service | 系统日志监控 |

### AI Agent 协作 / AI Agent Collaboration

| 插件 / Plugin | 类型 / Type | 说明 / Description |
|---|---|---|
| `AgentAssistant` | synchronous | Agent 间通信与任务委派 |
| `AgentMessage` | synchronous | Agent 向前端发送 WS 消息 |
| `MagiAgent` | synchronous | 多 Agent 协作决策 |
| `ProjectAnalyst` | synchronous | 项目分析 Agent |

### 日记与记忆 / Diary & Memory

| 插件 / Plugin | 类型 / Type | 说明 / Description |
|---|---|---|
| `DailyNote` | static | 日记读取与注入 |
| `DailyNoteWrite` | synchronous | AI 写入日记 |
| `DailyNoteManager` | hybridservice | 日记管理服务 |
| `RAGDiaryPlugin` | hybridservice | 语义分组、向量管理、元思考系统 |
| `LightMemo` | synchronous | 轻量级备忘录 |
| `ThoughtClusterManager` | synchronous | 思维簇管理 |

### 时间与调度 / Time & Scheduling

| 插件 / Plugin | 类型 / Type | 说明 / Description |
|---|---|---|
| `ScheduleManager` | hybridservice | 日程管理 |
| `ScheduleBriefing` | static | 日程简报注入 |
| `TimelineGenerator` | synchronous | 时间线生成 |

---

## 9. 快速开发指南 / Quick Dev Guide

### 创建一个最小同步插件 / Create a Minimal Synchronous Plugin

**Step 1**: 创建目录 / Create directory
```bash
mkdir Plugin/MyNewPlugin
```

**Step 2**: 创建 manifest / Create manifest
```json
// Plugin/MyNewPlugin/plugin-manifest.json
{
  "manifestVersion": "1.0.0",
  "name": "MyNewPlugin",
  "version": "1.0.0",
  "displayName": "我的新插件 / My New Plugin",
  "description": "这个插件做XXX。This plugin does XXX.",
  "author": "YourName",
  "pluginType": "synchronous",
  "entryPoint": { "type": "nodejs", "command": "node index.js" },
  "communication": { "protocol": "stdio", "timeout": 30000 },
  "capabilities": {
    "tools": [{
      "name": "MyNewPlugin",
      "description": "执行XXX功能 / Performs XXX function",
      "parameters": {
        "input": { "type": "string", "description": "输入参数 / Input parameter" }
      }
    }]
  }
}
```

**Step 3**: 实现插件逻辑 / Implement plugin logic
```javascript
// Plugin/MyNewPlugin/index.js
process.stdin.resume();
process.stdin.setEncoding('utf8');
let inputData = '';
process.stdin.on('data', chunk => inputData += chunk);
process.stdin.on('end', async () => {
  const params = JSON.parse(inputData);
  const { input } = params;
  // ... 执行逻辑 / Execute logic
  const result = `处理完成: ${input}`;
  process.stdout.write(JSON.stringify({ success: true, result }));
  process.exit(0);
});
```

**Step 4**: 重启服务器 / Restart server — 插件自动被发现和加载

### 调试技巧 / Debug Tips

- 查看插件加载日志 / Check plugin load logs: `DebugLog/archive/YYYY-MM-DD/Debug/`
- 手动测试插件 / Manually test plugin: `echo '{"input":"test"}' | node Plugin/MyNewPlugin/index.js`
- 检查 manifest 语法 / Check manifest syntax: `node -e "JSON.parse(require('fs').readFileSync('Plugin/MyNewPlugin/plugin-manifest.json','utf8'))"`

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ⬅️ 上一节 / Prev: [01_architecture](../01_architecture/README.md) | ➡️ 下一节 / Next: [03_memory_rag](../03_memory_rag/README.md)
