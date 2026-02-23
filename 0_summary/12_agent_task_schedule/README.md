# 12 AI Agent 任务调度系统深度解析 / AI Agent Task Scheduling — Deep Dive

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `Plugin.js` (PluginManager), `Plugin/ScheduleManager/ScheduleManager.js`, `Plugin/ScheduleBriefing/ScheduleBriefing.js`, `Plugin/DailyNoteManager/`, `Plugin/TimelineGenerator/`  
> **依赖库 / Dependencies**: `node-schedule` (cron jobs), `chokidar` (file watch)  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [VCP 任务调度设计哲学 / Scheduling Design Philosophy](#1-vcp-任务调度设计哲学--scheduling-design-philosophy)
2. [调度机制分类 / Scheduling Mechanisms](#2-调度机制分类--scheduling-mechanisms)
3. [静态插件 Cron 调度 / Static Plugin Cron Scheduling](#3-静态插件-cron-调度--static-plugin-cron-scheduling)
4. [ScheduleManager 插件（用户日程）/ ScheduleManager Plugin](#4-schedulemanager-插件用户日程--schedulemanager-plugin)
5. [ScheduleBriefing 插件（日程简报）/ ScheduleBriefing Plugin](#5-schedulebriefing-插件日程简报--schedulebriefing-plugin)
6. [异步任务插件 / Asynchronous Task Plugins](#6-异步任务插件--asynchronous-task-plugins)
7. [Archery 后台调度 / Archery Background Scheduling](#7-archery-后台调度--archery-background-scheduling)
8. [文件监听触发器 / File Watcher Triggers](#8-文件监听触发器--file-watcher-triggers)
9. [WebSocket 心跳调度 / WebSocket Heartbeat Scheduling](#9-websocket-心跳调度--websocket-heartbeat-scheduling)
10. [调度系统完整架构图 / Complete Scheduling Architecture](#10-调度系统完整架构图--complete-scheduling-architecture)
11. [与主流框架对比 / Comparison with Mainstream Frameworks](#11-与主流框架对比--comparison-with-mainstream-frameworks)

---

## 1. VCP 任务调度设计哲学 / Scheduling Design Philosophy

### 中文

VCP 的任务调度遵循**"即用即销，无状态轻量"**原则：
- 不维护常驻任务进程，避免 MCP 式的进程积压
- 定时任务由 `node-schedule` cron 触发，轻量执行后释放资源
- AI 的"主动性任务"（Archery 模式）实现了 AI 驱动的异步调度
- 人类日程由 ScheduleManager 管理，AI 日程由日记系统隐式承载

### English

VCP task scheduling follows the **"ephemeral, stateless, lightweight"** principle:
- No resident task processes — avoids MCP-style process accumulation
- Scheduled tasks triggered by `node-schedule` cron, resources released after execution
- AI's "proactive tasks" (Archery mode) enable AI-driven async scheduling
- Human schedules managed by ScheduleManager; AI schedules implicitly carried by diary system

---

## 2. 调度机制分类 / Scheduling Mechanisms

VCPToolBox 中存在 **5 种独立的调度机制**：

| 调度类型 / Type | 触发方式 / Trigger | 代码位置 / Location | 典型场景 / Typical Use |
|---|---|---|---|
| Cron 定时调度 | `node-schedule` cron 表达式 | `Plugin.js: _scheduleStaticPlugin()` | 周期性数据刷新 (天气/日历) |
| AI 主动调用 | 对话中 VCP 工具调用块 | `ToolExecutor.execute()` | AI 添加/查看日程 |
| Archery 后台触发 | `archery: true` 工具调用 | `ToolCallParser.separate()` | AI 发起无等待后台任务 |
| 文件变更触发 | `chokidar` 文件监听 | `KnowledgeBaseManager._startWatcher()` | 日记文件 → 自动索引 |
| 异步回调触发 | `POST /plugin-callback/:id` | `server.js` 内联路由 | 长耗时任务（视频生成）完成回调 |

---

## 3. 静态插件 Cron 调度 / Static Plugin Cron Scheduling

### 3.1 Cron 调度实现 / Cron Implementation

> 📁 `Plugin.js — _scheduleStaticPlugin()`  
> 依赖 / Dependency: `node-schedule` npm 包

```javascript
// Plugin.js (推断实现 / Inferred implementation)
_scheduleStaticPlugin(plugin) {
    const cronExpression = plugin.refreshIntervalCron || plugin.cron || '*/30 * * * *';
    
    const job = schedule.scheduleJob(cronExpression, async () => {
        try {
            await this._updateStaticPluginValue(plugin);
        } catch (err) {
            console.error(`[PluginManager] Cron job for ${plugin.name} failed:`, err.message);
        }
    });
    
    this.scheduledJobs.set(plugin.name, job);
    
    // 启动时立即执行一次（不等待第一个 cron 触发）
    this._updateStaticPluginValue(plugin).catch(err => 
        console.error(`[PluginManager] Initial update for ${plugin.name} failed:`, err.message)
    );
}
```

### 3.2 插件 Cron 配置示例 / Plugin Cron Config Examples

```json
// DailyNote 插件 — 每30分钟刷新日记内容
{
  "pluginType": "static",
  "refreshIntervalCron": "*/30 * * * *"
}

// ScheduleBriefing 插件 — 每小时刷新
{
  "pluginType": "static",
  "refreshIntervalCron": "0 * * * *"
}

// WeatherReporter 插件 — 每小时刷新天气
{
  "pluginType": "static",
  "refreshIntervalCron": "0 * * * *"
}

// DailyHot 插件 — 每小时刷新热榜
{
  "pluginType": "static",
  "refreshIntervalCron": "0 * * * *"
}
```

### 3.3 静态插件执行保护机制 / Execution Safety Mechanisms

```javascript
// _executeStaticPluginCommand() 中的保护机制

// 1. 超时控制 — 防止静态插件无限运行
const timeoutDuration = plugin.communication?.timeout || 60000;  // 默认 60s
const timeoutId = setTimeout(() => {
    console.log(`[PluginManager] Static plugin "${plugin.name}" completed work cycle.`);
    pluginProcess.kill('SIGKILL');
    resolve(output.trim());  // 超时不报错，返回已收集的输出
}, timeoutDuration);

// 2. 错误隔离 — 单个插件失败不影响其他插件
job.on('error', err => console.error(...));

// 3. 即用即销 — spawn 子进程，执行完毕释放
```

### 3.4 静态值更新与占位符存储 / Static Value Update & Placeholder Storage

```javascript
// Plugin.js — _updateStaticPluginValue()
// 执行插件脚本 → 解析输出 → 存入 staticPlaceholderValues Map

if (plugin.capabilities?.systemPromptPlaceholders) {
    plugin.capabilities.systemPromptPlaceholders.forEach(ph => {
        const placeholderKey = ph.placeholder;
        let parsedValue = newValue;
        
        // 尝试 JSON 解析（如果输出是 JSON 格式）
        try {
            parsedValue = JSON.parse(newValue);
        } catch (e) {
            // 保持原始字符串
        }
        
        this.staticPlaceholderValues.set(placeholderKey, {
            value: parsedValue,
            updatedAt: Date.now()
        });
    });
}
```

---

## 4. ScheduleManager 插件（用户日程）/ ScheduleManager Plugin

> 📁 `Plugin/ScheduleManager/`  
> 类型 / Type: `synchronous`

### 4.1 存储格式 / Storage Format

```json
// Plugin/ScheduleManager/schedules.json
[
  {
    "id": "1706083200000",         // Unix 时间戳（毫秒）作为 ID
    "time": "2026-03-01 10:00",    // YYYY-MM-DD HH:mm 格式
    "content": "准备季度汇报"
  },
  // ... 按时间升序排列
]
```

### 4.2 支持的命令 / Supported Commands

| 命令 / Command | 参数 / Parameters | 示例 / Example |
|---|---|---|
| `AddSchedule` | `time`, `content` | 添加新日程 |
| `DeleteSchedule` | `id` | 删除指定日程 |
| `ListSchedules` | 无 / None | 列出所有日程 |

### 4.3 AI 调用示例 / AI Call Examples

**添加日程 / Add schedule**:
```
<<<[TOOL_REQUEST]>>>
tool_name:「始」ScheduleManager「末」,
command:「始」AddSchedule「末」,
time:「始」2026-03-15 14:30「末」,
content:「始」与设计团队讨论新版本界面方案「末」
<<<[END_TOOL_REQUEST]>>>
```

**查看日程 / List schedules**:
```
<<<[TOOL_REQUEST]>>>
tool_name:「始」ScheduleManager「末」,
command:「始」ListSchedules「末」
<<<[END_TOOL_REQUEST]>>>
```

### 4.4 时间验证 / Time Validation

```javascript
// ScheduleManager.js
case 'AddSchedule':
    // 时间格式正则校验
    if (!request.time || !/^\d{4}-\d{1,2}-\d{1,2}/.test(request.time)) {
        return { status: 'error', error: `无效的时间格式: ${request.time}` };
    }
    
    // 生成 ID = 时间戳（毫秒），便于排序
    const newSchedule = { id: Date.now().toString(), ... };
    
    // 写入后按时间升序重排
    schedules.sort((a, b) => new Date(a.time) - new Date(b.time));
```

---

## 5. ScheduleBriefing 插件（日程简报）/ ScheduleBriefing Plugin

> 📁 `Plugin/ScheduleBriefing/`  
> 类型 / Type: `static` (每小时 cron)

### 功能 / Function

每小时自动：
1. **清理过期日程** — 删除时间已过去的日程条目
2. **提取下一个日程** — 找出距现在最近的未来日程
3. **更新占位符** — `{{VCPNextSchedule}}` → "下午 2:30: 与设计团队讨论..."

### 注入示例 / Injection Example

AI 的 system prompt 中:
```
当前时间: 2026-03-15 13:00
下一个日程: {{VCPNextSchedule}}
```

运行时替换为:
```
当前时间: 2026-03-15 13:00
下一个日程: 14:30 - 与设计团队讨论新版本界面方案 (还有1.5小时)
```

---

## 6. 异步任务插件 / Asynchronous Task Plugins

`asynchronous` 类型插件用于**长耗时任务**，实现 fire-and-forget + 回调模式：

### 6.1 执行流程 / Execution Flow

```
AI 发起调用:
  <<<[TOOL_REQUEST]>>>
  tool_name:「始」VideoGenerator「末」,
  prompt:「始」日落时分的海边「末」
  <<<[END_TOOL_REQUEST]>>>

           │
           ▼ (毫秒级)
PluginManager 生成 callbackId = crypto.randomUUID()
立即返回给 AI: "任务已受理，ID: xxx"
           │
           ▼ (后台异步执行，可能耗时数分钟)
VideoGenerator 子进程执行 → 生成视频文件
           │
           ▼ (任务完成)
POST /plugin-callback/xxx
{
  "result": "/pw=key/files/video_output.mp4",
  "success": true
}
           │
           ▼
chatCompletionHandler 收到回调
将结果注入到等待中的对话上下文
重新触发 AI 继续响应 (携带视频链接)
```

### 6.2 典型异步插件 / Typical Async Plugins

| 插件 / Plugin | 预估耗时 / Est. Time | 回调格式 / Callback |
|---|---|---|
| `VideoGenerator` | 1-30分钟 | 视频文件 URL |
| `SunoGen` | 30秒-5分钟 | 音乐文件 URL |
| `TencentCOSBackup` | 10秒-5分钟 | 备份状态报告 |

---

## 7. Archery 后台调度 / Archery Background Scheduling

Archery 模式是 VCP 特有的**AI 主动发起的后台异步调度**机制。

### 7.1 与异步插件的区别 / vs Asynchronous Plugins

| 特性 / Feature | `asynchronous` 插件 | Archery 模式 |
|---|---|---|
| 定义位置 / Defined in | manifest `pluginType` | 调用参数 `archery: true` |
| AI 等待结果 / AI waits | 是（通过回调）| 否（完全 fire-and-forget）|
| 结果注入上下文 / Result injected | ✅ 回调后注入 | ❌ 不注入 |
| 典型场景 / Use case | 视频生成、长计算 | 发通知、后台备份 |

### 7.2 使用示例 / Usage Examples

```
// 场景: AI 在回复用户的同时，后台发送消息通知
<<<[TOOL_REQUEST]>>>
tool_name:「始」AgentMessage「末」,
archery:「始」true「末」,
target:「始」VCPChat「末」,
message:「始」系统检测到重要事件，请主人注意「末」
<<<[END_TOOL_REQUEST]>>>

好的，我已经帮您分析完报告，主要发现如下...
(AI 同步继续回复，通知在后台发送)
```

---

## 8. 文件监听触发器 / File Watcher Triggers

`chokidar` 文件监听在 VCPToolBox 中承担了**反应式调度**的角色：

### 8.1 日记文件监听 / Diary File Watcher

```javascript
// KnowledgeBaseManager._startWatcher()
this.watcher = chokidar.watch(this.config.rootPath, {
    ignoreInitial: false,          // 启动时处理已有文件
    persistent: true,
    awaitWriteFinish: { stabilityThreshold: 500 }  // 等待文件写入稳定
});

this.watcher
    .on('add', filePath => this._onFileAdded(filePath))
    .on('change', filePath => this._onFileChanged(filePath))
    .on('unlink', filePath => this._onFileDeleted(filePath));
```

### 8.2 RAG 参数热调控监听 / RAG Params Hot-Reload Watcher

```javascript
// KnowledgeBaseManager._startRagParamsWatcher()
this.ragParamsWatcher = chokidar.watch('rag_params.json');
this.ragParamsWatcher.on('change', async () => {
    await this.loadRagParams();  // 立即重新加载，无需重启
});

// RAGDiaryPlugin 同样有独立的 rag_params.json 监听
```

### 8.3 批量处理窗口 / Batch Processing Window

文件变更不立即处理，而是积攒后批量处理：

```javascript
// 防止频繁写入时触发过多 Embedding API 调用
this._onFileChanged(filePath) {
    this.pendingFiles.add(filePath);
    
    if (this.batchTimer) clearTimeout(this.batchTimer);
    this.batchTimer = setTimeout(
        () => this._processBatch(),
        this.config.batchWindow  // 默认 2000ms
    );
}
```

---

## 9. WebSocket 心跳调度 / WebSocket Heartbeat Scheduling

WebSocket 连接使用 `setInterval` 实现心跳调度：

```javascript
// modules/handlers/streamHandler.js — SSE 保活
keepAliveTimer = setInterval(() => {
    res.write(': vcp-keepalive\n\n');  // 每 5 秒发送 SSE 心跳
}, 5000);

// WebSocketServer.js — WS 心跳 (推断)
setInterval(() => {
    clients.forEach(ws => {
        if (ws.readyState === WebSocket.OPEN) {
            ws.ping();  // 或发送 {type: 'ping'}
        }
    });
}, 30000);  // 每 30 秒
```

---

## 10. 调度系统完整架构图 / Complete Scheduling Architecture

```
VCPToolBox 任务调度全景 / Full Scheduling Landscape
══════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│  层级 1: 系统级定时器 / System-Level Timers                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ node-schedule cron jobs                                    ││
│  │ ├── DailyNote        "*/30 * * * *"  → 刷新日记占位符     ││
│  │ ├── ScheduleBriefing "0 * * * *"     → 刷新下一日程      ││
│  │ ├── WeatherReporter  "0 * * * *"     → 刷新天气          ││
│  │ ├── DailyHot         "0 * * * *"     → 刷新热榜          ││
│  │ └── ArxivDailyPapers "0 8 * * *"     → 刷新每日论文      ││
│  └────────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────────┤
│  层级 2: 文件监听触发 / File Watcher Triggers                    │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ chokidar watchers                                          ││
│  │ ├── dailynote/**/*.md   → 批量索引更新 (2s 窗口)           ││
│  │ ├── rag_params.json     → 热调控参数重载 (即时)            ││
│  │ └── agent_map.json      → Agent 映射热重载 (即时)          ││
│  └────────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────────┤
│  层级 3: AI 主动调度 / AI-Initiated Scheduling                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ VCP 工具调用                                               ││
│  │ ├── ScheduleManager (AddSchedule)  → 用户日程管理          ││
│  │ ├── DailyNoteWrite                 → 写入长期记忆          ││
│  │ ├── archery: true 模式             → 后台 fire-and-forget  ││
│  │ └── asynchronous 插件              → 长任务 + 回调         ││
│  └────────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────────┤
│  层级 4: 连接保活调度 / Connection Keepalive Scheduling          │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ setInterval                                                ││
│  │ ├── SSE keepalive comment   5s  → 防止流式断连            ││
│  │ └── WS ping/pong           30s  → 检测分布式节点活性       ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. 与主流框架对比 / Comparison with Mainstream Frameworks

### 横向对比表 / Cross-Framework Comparison

| 特性 / Feature | VCP | Celery (Python) | BullMQ (Node.js) | n8n | Temporal |
|---|---|---|---|---|---|
| 定时任务 / Cron jobs | ✅ node-schedule | ✅ Celery Beat | ✅ | ✅ | ✅ |
| 异步任务队列 / Async queue | 🔍 简单回调 | ✅ Redis 队列 | ✅ Redis 队列 | ✅ | ✅ |
| AI 驱动调度 / AI-driven | ✅ Archery 模式 | ❌ | ❌ | 🔍 | ❌ |
| 任务重试 / Task retry | ✅ (fetchWithRetry) | ✅ 自动重试 | ✅ 指数退避 | ✅ | ✅ |
| 任务持久化 / Task persistence | ❌ (内存) | ✅ Redis | ✅ Redis | ✅ DB | ✅ DB |
| 分布式工作节点 / Distributed workers | ✅ VCPDistributed | ✅ | ✅ | ✅ | ✅ |
| 工作流编排 / Workflow orchestration | 🔍 VCP Loop | 🔍 Chain | 🔍 | ✅ 可视化 | ✅ 代码定义 |
| 可视化 UI | ✅ AdminPanel | ✅ Flower | ✅ Bull Board | ✅ | ✅ |
| 外部依赖 / External deps | ❌ 无 | ✅ Redis/RabbitMQ | ✅ Redis | ✅ DB | ✅ DB |
| 进程模型 / Process model | 即用即销 | 常驻 Worker | 常驻 Worker | 常驻 | 常驻 |

### BullMQ 对比详解 / BullMQ Detailed Comparison

**BullMQ** 是 Node.js 最成熟的任务队列框架，基于 Redis：

```javascript
// BullMQ — 基于 Redis 的持久化任务队列
const { Queue, Worker } = require('bullmq');

// 生产者
const queue = new Queue('image-generation', { connection: redis });
await queue.add('generate', { prompt: '...' }, {
    attempts: 3,           // 自动重试 3 次
    backoff: { type: 'exponential', delay: 1000 }  // 指数退避
});

// 消费者
const worker = new Worker('image-generation', async job => {
    return await generateImage(job.data.prompt);
}, { connection: redis });
```

**BullMQ 优势**:
- Redis 持久化 — 任务不丢失（VCP 内存任务重启后丢失）
- 丰富的任务生命周期事件（active, completed, failed, stalled）
- Bull Board UI 可视化
- 速率限制 (rate limiting)、并发控制

**与 VCP 的差异**:
```
BullMQ:
  + 企业级任务队列，Redis 持久化，生产可靠
  + 丰富的队列管理功能（优先级、延迟、批量）
  - 需要 Redis 外部依赖
  - 不了解 AI 对话上下文，无法 AI 驱动调度

VCP:
  + 零外部依赖，集成在 AI 对话流程中
  + Archery 模式允许 AI 主动发起后台任务
  - 无 Redis 持久化，服务重启后异步任务回调可能丢失
  - 适合轻量任务，不适合高并发生产任务队列
```

### Temporal 对比详解 / Temporal Detailed Comparison

**Temporal** 是目前最先进的**持久化工作流引擎**（Cadence 的继任者）：

```go
// Temporal Workflow — 代码即编排
func VideoProcessingWorkflow(ctx workflow.Context, input VideoInput) error {
    // 即使服务重启，工作流状态也持久化恢复
    err := workflow.ExecuteActivity(ctx, DownloadVideo, input.URL).Get(ctx, nil)
    err = workflow.ExecuteActivity(ctx, TranscodeVideo, ...).Get(ctx, nil)
    err = workflow.ExecuteActivity(ctx, UploadToCloud, ...).Get(ctx, nil)
    return err
}
```

**Temporal 优势**:
- 持久化工作流 — 代码崩溃/重启后从断点恢复
- 无限持续时间工作流（秒级到年级）
- 内置 Saga 模式（分布式事务补偿）
- 完善的测试框架

**与 VCP 的差异**:
```
Temporal:
  + 适合需要持久保证的企业关键工作流
  + 秒级到年级的任意时长任务
  - 需要独立 Temporal Server 部署（Go/Java）
  - 学习曲线较高

VCP:
  + 零基础设施 — 作为 AI 中间层开箱即用
  + AI 可以用自然语言规划和调度任务
  - 不适合严格可靠性要求的生产工作流
  - 重启后 async 回调可能丢失
```

### n8n 对比详解 / n8n Detailed Comparison

**n8n** 是开源的低代码工作流自动化平台，类似 Zapier：

```
可视化拖拽构建工作流:
  Webhook触发 → 调用API → 数据转换 → 发送邮件 → 存储数据库
```

**与 VCP 的核心区别**:
```
n8n:
  + 可视化界面，非技术人员可构建工作流
  + 丰富的第三方集成（500+ 节点）
  - AI 只是作为"一个节点"，不是调度主体

VCP:
  + AI 是主体（驱动整个流程）
  + AI 可以根据对话上下文动态调整调度
  - 无可视化工作流编辑器
```

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ⬅️ 上一节 / Prev: [11_agent_rag](../11_agent_rag/README.md)
