# 09 AI Agent 工具调用深度解析 / AI Agent Tool Calling — Deep Dive

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `modules/vcpLoop/toolCallParser.js`, `modules/vcpLoop/toolExecutor.js`, `modules/handlers/streamHandler.js`, `modules/handlers/nonStreamHandler.js`, `modules/chatCompletionHandler.js`  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [VCP 调用协议设计哲学 / VCP Protocol Design Philosophy](#1-vcp-调用协议设计哲学--vcp-protocol-design-philosophy)
2. [调用语法完整规范 / Complete Call Syntax Spec](#2-调用语法完整规范--complete-call-syntax-spec)
3. [VCP 调用循环 / VCP Loop Architecture](#3-vcp-调用循环--vcp-loop-architecture)
4. [ToolCallParser 解析器实现 / ToolCallParser Implementation](#4-toolcallparser-解析器实现--toolcallparser-implementation)
5. [ToolExecutor 执行器实现 / ToolExecutor Implementation](#5-toolexecutor-执行器实现--toolexecutor-implementation)
6. [Archery 模式（后台触发）/ Archery Mode (Background Trigger)](#6-archery-模式后台触发--archery-mode-background-trigger)
7. [流式 vs 非流式处理 / Streaming vs Non-Streaming](#7-流式-vs-非流式处理--streaming-vs-non-streaming)
8. [并发工具调用 / Concurrent Tool Calls](#8-并发工具调用--concurrent-tool-calls)
9. [错误处理与重试 / Error Handling & Retry](#9-错误处理与重试--error-handling--retry)
10. [与主流框架对比 / Comparison with Mainstream Frameworks](#10-与主流框架对比--comparison-with-mainstream-frameworks)

---

## 1. VCP 调用协议设计哲学 / VCP Protocol Design Philosophy

### 中文

VCP 协议的核心出发点是：**为 AI 设计符合其认知工学的调用语言**，而不是强迫 AI 适应为机器设计的 JSON 格式。

**三项关键设计原则**:
1. **串语法保护心流** — 工具调用嵌入在 AI 的自然语言输出中，AI 无需打断思路切换模式
2. **容错设计允许小错** — 参数值包裹在 `「始」...「末」` 中而非严格 JSON，截断或缺失逗号不会整体解析失败
3. **前端零侵入** — 任何标准 OpenAI 兼容前端无需修改即可使用，工具调用完全在服务端处理

### English

VCP Protocol's core idea: **design a call language that fits AI's cognitive ergonomics**, rather than forcing AI to adapt to JSON designed for machines.

**Three key design principles**:
1. **Serial syntax protects flow state** — Tool calls are embedded in AI's natural language output; AI doesn't need to break thought to switch modes
2. **Fault-tolerant design allows minor mistakes** — Parameter values wrapped in `「始」...「末」` rather than strict JSON; truncation or missing commas don't cause total parse failure
3. **Zero frontend intrusion** — Any standard OpenAI-compatible frontend works unmodified; tool calls handled entirely server-side

---

## 2. 调用语法完整规范 / Complete Call Syntax Spec

### 基础格式 / Base Format

```
<<<[TOOL_REQUEST]>>>
tool_name:「始」PluginName「末」,
param1:「始」value1「末」,
param2:「始」multiline
value is fine too「末」
<<<[END_TOOL_REQUEST]>>>
```

### 特殊控制参数 / Special Control Parameters

| 参数 / Parameter | 值 / Values | 说明 / Description |
|---|---|---|
| `tool_name` | 插件名 / Plugin name | **必需** — 对应 manifest `name` 字段 |
| `archery` | `true` \| `no_reply` | 后台触发，不等待结果/不回复给 AI |
| `ink` | `mark_history` | 标记本次调用写入历史记录 |
| `tool_password` | 验证码 | 启用 UserAuth 时的调用鉴权码 |

### 多命令变体 / Multi-Command Variant

支持同一插件多子命令（如 ScheduleManager）:
```
<<<[TOOL_REQUEST]>>>
tool_name:「始」ScheduleManager「末」,
command:「始」AddSchedule「末」,
time:「始」2026-03-01 10:00「末」,
content:「始」晨会「末」
<<<[END_TOOL_REQUEST]>>>
```

### 并发调用（单次输出多块）/ Parallel Calls

```
<<<[TOOL_REQUEST]>>>
tool_name:「始」GoogleSearch「末」,
query:「始」VCPToolBox architecture「末」
<<<[END_TOOL_REQUEST]>>>
<<<[TOOL_REQUEST]>>>
tool_name:「始」UrlFetch「末」,
url:「始」https://github.com/camsiabor/VCPToolBox「末」
<<<[END_TOOL_REQUEST]>>>
```

两个请求**并发执行**，合并后返回给 AI。

### `<think>` 标签保护 / `<think>` Tag Protection

```
<think>
这里我可以用工具 <<<[TOOL_REQUEST]>>> 但不会被解析
</think>

实际调用 / Actual call:
<<<[TOOL_REQUEST]>>>
tool_name:「始」SearchTool「末」
<<<[END_TOOL_REQUEST]>>>
```

✅ `<think>...</think>` 块内的工具调用块被自动剥离，不触发执行。  
📁 `modules/vcpLoop/toolCallParser.js:14`  
```javascript
const contentWithoutThink = content.replace(/<think>[\s\S]*?<\/think>/g, '');
```

---

## 3. VCP 调用循环 / VCP Loop Architecture

### 循环架构概览 / Loop Architecture Overview

```
客户端请求 POST /v1/chat/completions
         │
         ▼
[ChatCompletionHandler]
  preprocessors 执行 (messagePreprocessor 类插件)
         │
         ▼
[第1轮 AI 请求 / Round 1 AI Request]
  → 上游 LLM API
  → 流式读取响应
         │
         ▼ 检测到 <<<[TOOL_REQUEST]>>> ?
  ├── 否 → 响应直接透传给客户端 (结束)
  └── 是 →
         ▼
[ToolCallParser.parse(content)]
  解析出所有工具调用块
         │
         ▼
[ToolExecutor.executeAll(toolCalls)]
  并发执行所有普通(non-archery)工具
         │
         ▼
[构建下一轮消息 / Build next-round messages]
  将工具结果注入 messages 数组
  refreshRagBlocksIfNeeded()  ← RAG 区块刷新
         │
         ▼
[第 N 轮 AI 请求 / Round N AI Request]
  maxVCPLoopStream 控制最大循环次数 (默认 5)
  若无新工具调用 → 结束并回传客户端
  若有新工具调用 → 继续循环
         │
[Archery 工具] ← 并行触发，不等待，不注入上下文
```

### 循环深度控制 / Loop Depth Control

```javascript
// modules/handlers/streamHandler.js
const maxRecursion = maxVCPLoopStream || 5;  // 最大 VCP 循环次数
let recursionDepth = 0;

// 每轮递增，达到上限时强制结束
if (recursionDepth >= maxRecursion) {
    // 停止循环，直接透传最后一轮 AI 输出
}
```

| 配置 / Config | 说明 / Description |
|---|---|
| `MaxVCPLoopStream` | 流式模式最大循环次数（config.env）|
| `MaxVCPLoopNonStream` | 非流式模式最大循环次数 |

---

## 4. ToolCallParser 解析器实现 / ToolCallParser Implementation

> 📁 `modules/vcpLoop/toolCallParser.js`

### 解析流程 / Parse Flow

```javascript
// 1. 剥离 <think> 块
const contentWithoutThink = content.replace(/<think>[\s\S]*?<\/think>/g, '');

// 2. 循环查找所有 <<<[TOOL_REQUEST]>>> ... <<<[END_TOOL_REQUEST]>>> 块
while (searchOffset < content.length) {
    const startIndex = content.indexOf('<<<[TOOL_REQUEST]>>>', searchOffset);
    const endIndex = content.indexOf('<<<[END_TOOL_REQUEST]>>>', startIndex);
    const blockContent = content.substring(startIndex + ..., endIndex).trim();
    const parsed = this._parseBlock(blockContent);
    toolCalls.push(parsed);
}

// 3. 解析单个块内的参数
// 正则: /([\w_]+)\s*:\s*「始」([\s\S]*?)「末」\s*(?:,)?/g
// 提取: key = 参数名, value = 「始」...「末」之间的内容（支持多行）
```

### 返回结构 / Return Structure

```javascript
[
  {
    name: "GoogleSearch",       // 工具名 (tool_name 字段)
    args: {                     // 所有其他参数
      query: "search term",
      maxResults: "5"
    },
    archery: false,             // 是否后台触发
    markHistory: false          // 是否标记历史
  },
  // ...
]
```

### 容错性 / Fault Tolerance

| 异常场景 / Edge Case | 处理方式 / Handling |
|---|---|
| 缺少 `<<<[END_TOOL_REQUEST]>>>` | 跳过该不完整块，继续解析后续块 |
| 参数值包含换行符 | `[\s\S]*?` 匹配多行，正常解析 |
| 参数后缺少逗号 | 正则末尾 `(?:,)?` 可选，不影响解析 |
| `<think>` 内有工具调用块 | 全部剥离，不触发执行 |
| 空内容 / null | 直接返回空数组 |

---

## 5. ToolExecutor 执行器实现 / ToolExecutor Implementation

> 📁 `modules/vcpLoop/toolExecutor.js`

### 执行流程 / Execution Flow

```javascript
async execute(toolCall, clientIp) {
  // 1. 验证码校验 (启用 UserAuth 时)
  if (this.vcpToolCode) {
    const authResult = await this._verifyAuth(args);
    if (!authResult.valid) return this._createErrorResult(...);
  }

  // 2. 检查插件是否存在
  if (!this.pluginManager.getPlugin(name)) {
    return this._createErrorResult(name, `未找到名为 "${name}" 的插件`);
  }

  // 3. 执行插件 (委托给 PluginManager.processToolCall)
  const result = await this.pluginManager.processToolCall(name, args, clientIp);

  // 4. 处理返回结果 (支持富内容/文本/JSON)
  return this._processResult(name, result);
}
```

### 富内容格式支持 / Rich Content Support

工具可返回多模态内容：

```javascript
// 工具返回富内容示例 / Tool rich content return example
{
  "data": {
    "content": [
      { "type": "text", "text": "图片已生成" },
      { "type": "image_url", "image_url": { "url": "data:image/png;base64,..." } }
    ]
  }
}
```

### WebSocket 广播 / WS Broadcast

每次工具执行完成，自动广播到 VCPLog 客户端：

```javascript
this.webSocketServer.broadcast({
  type: 'vcp_log',
  data: { tool_name: toolName, status: 'success'|'error', content }
}, 'VCPLog');
```

---

## 6. Archery 模式（后台触发）/ Archery Mode (Background Trigger)

### 定义 / Definition

`archery: true` 或 `archery: no_reply` — 工具在**后台触发**，主流程不等待其完成，不将结果注入 AI 上下文。

### 使用场景 / Use Cases

```
场景1: AI 触发后台任务，继续回复用户
  <<<[TOOL_REQUEST]>>>
  tool_name:「始」TencentCOSBackup「末」,
  archery:「始」true「末」
  <<<[END_TOOL_REQUEST]>>>
  好的，我已在后台启动备份，请继续告诉我需要什么帮助...

场景2: AI 发送通知而无需等待结果
  <<<[TOOL_REQUEST]>>>
  tool_name:「始」AgentMessage「末」,
  archery:「始」no_reply「末」,
  target:「始」VCPChat「末」,
  message:「始」任务完成通知「末」
  <<<[END_TOOL_REQUEST]>>>
```

### 实现原理 / Implementation

```javascript
// ToolCallParser.separate() 分离普通调用和 archery 调用
const { normal, archery } = ToolCallParser.separate(toolCalls);

// 普通调用：await 等待，结果注入上下文
const normalResults = await toolExecutor.executeAll(normal, clientIp);

// Archery 调用：fire-and-forget，不等待
archery.forEach(tc => toolExecutor.execute(tc, clientIp).catch(err => 
  console.error('[Archery] Background tool error:', err)
));
```

---

## 7. 流式 vs 非流式处理 / Streaming vs Non-Streaming

| 特性 / Feature | 流式 (StreamHandler) | 非流式 (NonStreamHandler) |
|---|---|---|
| 上游响应方式 | SSE text/event-stream | JSON application/json |
| 客户端感知 | 字符逐步出现 | 一次性返回 |
| VCP 循环 | 每轮收集完整内容再判断 | 同上 |
| KeepAlive | ✅ 5s SSE 心跳注释 | N/A |
| 中断支持 | ✅ AbortController | ✅ AbortController |
| 配置 | `MaxVCPLoopStream` | `MaxVCPLoopNonStream` |

### SSE 保活心跳 / SSE KeepAlive

```javascript
// streamHandler.js — 防止上游卡顿时浏览器假死
keepAliveTimer = setInterval(() => {
  res.write(': vcp-keepalive\n\n');  // SSE 注释行，不影响内容
}, 5000);
```

### 上游错误流代理 / Upstream Error Stream Proxy

```javascript
// 即使上游返回 4xx/5xx，也不中断 SSE 连接
// 而是将错误作为 SSE chunk 发送给客户端
if (isOriginalRequestStreaming && upstreamStatus !== 200) {
  res.setHeader('Content-Type', 'text/event-stream');
  // 构造包含错误信息的 SSE chunk
  const errorContent = `[UPSTREAM_ERROR] 上游API返回状态码 ${upstreamStatus}`;
  // ... 发送并正常结束流
}
```

---

## 8. 并发工具调用 / Concurrent Tool Calls

VCP 并发调用的效率优势对比：

```
MCP / JSON Function Calling（串行）:
  调用1 ─── 等待 ─── 返回
                         调用2 ─── 等待 ─── 返回
                                                  调用3 ─── 等待 ─── 返回
  总时间 = T1 + T2 + T3

VCP（并发）:
  调用1 ─┐
  调用2 ─┤─── 并发执行 ─── 合并结果
  调用3 ─┘
  总时间 = max(T1, T2, T3)
```

实现：`Promise.all()` 并发执行所有 non-archery 工具：

```javascript
// toolExecutor.js
async executeAll(toolCalls, clientIp) {
  return Promise.all(
    toolCalls.map(tc => this.execute(tc, clientIp))
  );
}
```

---

## 9. 错误处理与重试 / Error Handling & Retry

### 上游 API 重试 / Upstream API Retry

```javascript
// chatCompletionHandler.js — fetchWithRetry
async function fetchWithRetry(url, options, {
  retries = 3,
  delay = 1000,
  connectionTimeout = 120000  // 连接超时安全网：防止上游 API 静默挂起
}) {
  // 重试触发条件: HTTP 429, 500, 503 或网络错误
  // 超时中止: 每次请求独立 AbortController + setTimeout
  // 用户中止: 外部 AbortController 信号转发，不重试
}
```

### 工具执行错误格式 / Tool Execution Error Format

```javascript
// 统一错误返回格式
{
  success: false,
  error: "错误描述",
  content: [{ type: 'text', text: '[错误] 错误描述' }]
}
```

错误检测逻辑 (`isToolResultError`)：支持多种错误格式：
- 对象中含 `error: true`, `success: false`, `status: 'error'`
- 字符串以 `[error]`, `[错误]`, `error:` 等前缀开头

---

## 10. 与主流框架对比 / Comparison with Mainstream Frameworks

### 横向对比表 / Cross-Framework Comparison

| 特性 / Feature | VCP (VCPToolBox) | LangChain Tools | AutoGen | CrewAI | Semantic Kernel |
|---|---|---|---|---|---|
| 调用语法 / Call syntax | 文本标记 / Text markers | JSON Function Calling | JSON / Python | Python decorator | JSON / YAML |
| 模型依赖 / Model dependency | ❌ 无（任意 API） | ✅ 需要 Function Calling | ✅ 需要 Function Calling | ✅ 需要 Function Calling | ✅ 需要 Function Calling |
| 前端侵入 / Frontend intrusion | ❌ 零侵入 | ✅ 需要集成 SDK | ✅ 需要集成 SDK | ✅ 需要集成 SDK | ✅ 需要集成 SDK |
| 并发调用 / Parallel calls | ✅ 单轮多块 | 🔍 部分支持 | ✅ | ✅ | 🔍 |
| 后台触发 / Background trigger | ✅ Archery 模式 | ❌ | 🔍 | ❌ | ❌ |
| 流式支持 / Streaming support | ✅ SSE KeepAlive | ✅ | ❌ | ❌ | ✅ |
| 最大循环深度 / Max loop depth | 可配置 (默认 5) | 可配置 | 可配置 | 可配置 | 可配置 |
| 进程模型 / Process model | 即用即销 / Ephemeral | 常驻 SDK / Resident SDK | 常驻 Agent | 常驻 Agent | 常驻 Kernel |
| 富内容返回 / Rich content return | ✅ 图文混合 | ✅ | 🔍 | 🔍 | ✅ |

### LangChain Tools 对比详解 / LangChain Tools Detailed Comparison

**LangChain Tool 定义方式**:
```python
from langchain.tools import tool

@tool
def search(query: str) -> str:
    """Search the web for information."""
    return search_api(query)

# 依赖 OpenAI Function Calling JSON schema
# 需要模型支持 functions 参数
```

**VCP 等价定义（plugin-manifest.json）**:
```json
{
  "name": "Search",
  "capabilities": {
    "tools": [{
      "name": "Search",
      "description": "Search the web for information.",
      "parameters": { "query": { "type": "string" } }
    }]
  }
}
```

**核心差异**: VCP 无需模型支持 `functions` 参数，适用于所有文本生成模型。

### AutoGen 对比详解 / AutoGen Detailed Comparison

**AutoGen** 的 Agent 使用 Python 函数注册工具，调用通过 Function Calling 路由：
```python
# AutoGen — 需要 OpenAI API + function_call 支持
@user_proxy.register_for_execution()
@assistant.register_for_llm(description="Search tool")
def search(query: str) -> str: ...
```

**VCP 差异**:
- AutoGen 依赖 Python 运行时和 Function Calling
- VCP 支持任意 LLM API，工具可以是 Node.js/Python/Rust/Shell
- AutoGen Agent 是常驻进程；VCP 工具是即用即销

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ➡️ 下一节 / Next: [10_agent_memory](../10_agent_memory/README.md)
