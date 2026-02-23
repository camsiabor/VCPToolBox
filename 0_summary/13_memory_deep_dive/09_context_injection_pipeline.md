# 09 记忆注入上下文全链路详解 — "AI 如何记得？"
# 09 Memory-to-Context Injection Pipeline — "How Does the AI Remember?"

> **所属 / Parent**: [README.md](./README.md)  
> **核心源文件 / Key Source Files**:  
> - `modules/chatCompletionHandler.js` — 主请求处理器 / Main Request Handler  
> - `modules/messageProcessor.js` — 变量替换引擎 / Variable Replacement Engine  
> - `modules/contextManager.js` — Token 预算裁剪 / Token Budget Pruning  
> - `modules/handlers/streamHandler.js` — VCP 工具循环 / VCP Tool Loop  
> - `modules/handlers/nonStreamHandler.js` — 非流式工具循环  
> - `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js` — RAG 区块注入 / RAG Block Injection  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [核心问题与答案 / Core Question & Answer](#1-核心问题与答案--core-question--answer)
2. [全链路鸟瞰图 / Full Pipeline Bird's-Eye View](#2-全链路鸟瞰图--full-pipeline-birds-eye-view)
3. [阶段 0：请求入口 / Stage 0 — Request Entry](#3-阶段-0请求入口--stage-0--request-entry)
4. [阶段 1：上下文预算裁剪 / Stage 1 — Context Budget Pruning](#4-阶段-1上下文预算裁剪--stage-1--context-budget-pruning)
5. [阶段 2：角色分割 (RoleDivider) / Stage 2 — Role Divider](#5-阶段-2角色分割-roledivider--stage-2--role-divider)
6. [阶段 3：变量替换引擎 / Stage 3 — Variable Replacement Engine](#6-阶段-3变量替换引擎--stage-3--variable-replacement-engine)
   - 6.1 [Agent 展开 / Agent Expansion](#61-agent-展开--agent-expansion)
   - 6.2 [静态插件占位符 / Static Plugin Placeholders](#62-静态插件占位符--static-plugin-placeholders)
   - 6.3 [动态上下文折叠 / Dynamic Context Folding](#63-动态上下文折叠--dynamic-context-folding)
   - 6.4 [时间变量 / Time Variables](#64-时间变量--time-variables)
   - 6.5 [环境变量注入 / Env Variable Injection](#65-环境变量注入--env-variable-injection)
7. [阶段 4：消息预处理器 / Stage 4 — Message Preprocessors](#7-阶段-4消息预处理器--stage-4--message-preprocessors)
   - 7.1 [RAGDiaryPlugin 记忆注入 / RAGDiaryPlugin Memory Injection](#71-ragdiaryplugin-记忆注入--ragdiaryplugin-memory-injection)
   - 7.2 [RAG 区块格式详解 / RAG Block Format](#72-rag-区块格式详解--rag-block-format)
8. [阶段 5：LLM API 调用 / Stage 5 — LLM API Call](#8-阶段-5llm-api-调用--stage-5--llm-api-call)
9. [阶段 6：VCP 工具执行循环 / Stage 6 — VCP Tool Execution Loop](#9-阶段-6vcp-工具执行循环--stage-6--vcp-tool-execution-loop)
   - 9.1 [工具结果注入 / Tool Result Injection](#91-工具结果注入--tool-result-injection)
   - 9.2 [RAG 区块对话中刷新 / In-Loop RAG Refresh](#92-rag-区块对话中刷新--in-loop-rag-refresh)
10. [最终上下文结构示意图 / Final Context Structure Diagram](#10-最终上下文结构示意图--final-context-structure-diagram)
11. [记忆 Token 占用分析 / Memory Token Budget Analysis](#11-记忆-token-占用分析--memory-token-budget-analysis)
12. [调试完整上下文 / Debugging the Full Context](#12-调试完整上下文--debugging-the-full-context)
13. [设计对比：与 LangChain / LangGraph 的区别 / Design Comparison](#13-设计对比与-langchain--langgraph-的区别--design-comparison)

---

## 1. 核心问题与答案 / Core Question & Answer

> **问 / Q**: VCP 是如何让 AI "记得"、"知道"那些记忆的？
> **Q**: How does VCP make the AI "remember" and "know" those memories?

**一句话回答**:  
VCP 将检索到的记忆以**格式化 HTML 注释块**的形式**直接嵌入 system 消息的文本内容**，在每次 API 调用前将这些块作为整个 `messages` 数组的一部分发送给 LLM，让 LLM 在单次推理的上下文窗口中"看到"记忆。

**One-sentence answer**:  
VCP embeds retrieved memories as **formatted HTML comment blocks directly inside the system message text**, then sends those blocks as part of the full `messages` array on every API call, so the LLM "sees" the memories in its single-inference context window.

**没有外部持久上下文 / No external persistent context**:  
LLM 本身没有任何持久状态。"记忆"完全来自 VCP 在每次调用前**从磁盘检索、注入到 system/user 消息文本中**的内容。

---

## 2. 全链路鸟瞰图 / Full Pipeline Bird's-Eye View

```
客户端 (SillyTavern / VCPChat)
  POST /v1/chat/completions
  {
    messages: [
      { role: "system", content: "你是小可...[[小可]][[{{VCPDailyTools}}]]" },
      { role: "user",   content: "今天天气怎么样？" }
    ]
  }
        │
        ▼
ChatCompletionHandler.handle()
  │
  ├─ 阶段 1: contextManager.pruneMessages()       裁剪超 Token 的历史
  │          [保护: system 消息 + 最后2条消息]
  │
  ├─ 阶段 2: roleDivider.process()                角色分割（多角色支持）
  │
  ├─ 阶段 3: messageProcessor.replaceAgentVariables()  变量替换大管道
  │   ├─ Agent 展开:    {{小可}} → 读取 Agent/小可.txt 并递归展开
  │   ├─ 静态插件占位符: {{VCPDailyTools}} → 插件描述列表
  │   ├─ 时间变量:      {{Date}} {{Time}} {{Today}} {{Festival}}
  │   ├─ 环境变量:      {{TarXxx}} {{VarXxx}} (→ TVStxt/*.txt)
  │   ├─ SarPrompt:    模型特定提示词注入
  │   └─ 动态上下文折叠: vcp_dynamic_fold 协议
  │
  ├─ 阶段 4: pluginManager 消息预处理器链
  │   ├─ VCPTavern      (优先级最高，预设注入)
  │   ├─ MultiModal/ImageProcessor (媒体处理)
  │   └─ RAGDiaryPlugin ← ★ 核心记忆注入
  │       processMessages(messages):
  │         ① getSingleEmbeddingCached(query) → 查询向量
  │         ② TagMemo + Shotgun + Dedup + Rerank → 候选记忆片段
  │         ③ 替换系统消息中的 [[小可]] 占位符:
  │            原文: "...[[小可]]..."
  │            替换后: "...<!-- VCP_RAG_BLOCK_START {"dbName":"小可","k":5} -->
  │                    [--- 从"小可日记本"中检索到的相关记忆片段 ---]
  │                    * [2026-02-20] 今天学了量子力学...
  │                    * [2026-02-18] 关于代码重构...
  │                    [--- 记忆片段结束 ---]
  │                    <!-- VCP_RAG_BLOCK_END -->..."
  │
  ├─ 阶段 5: fetchWithRetry() → LLM API
  │   POST apiUrl/v1/chat/completions
  │   body.messages = [
  │     { role: "system", content: "你是小可...<RAG_BLOCK>记忆内容</RAG_BLOCK>...工具描述..." },
  │     { role: "user",   content: "今天天气怎么样？" }
  │   ]
  │       ← AI 在这里"看到"记忆，开始推理
  │
  └─ 阶段 6: VCP 工具循环 (如 AI 使用了工具)
      ├─ 工具执行 → 结果追加到 messages
      │  messages.push({ role: "user", content: "<!-- VCP_TOOL_PAYLOAD -->\n{results}" })
      ├─ RAG 刷新 (RAGMemoRefresh=true)
      │  _refreshRagBlocksIfNeeded() → 基于新上下文重新检索
      │  → 更新 messages 中的 VCP_RAG_BLOCK 内容
      └─ 再次调用 LLM API (新上下文 = 旧 messages + 工具结果 + 刷新记忆)
```

---

## 3. 阶段 0：请求入口 / Stage 0 — Request Entry

**📁 位置**: `modules/chatCompletionHandler.js:304` — `handle(req, res)`

```javascript
// 客户端 POST body 格式
{
  "model": "gemini-2.0-flash",
  "messages": [
    {
      "role": "system",
      "content": "你是小可，一个AI助手。[[小可]]{{VCPAllTools}}{{Date}}"
    },
    {
      "role": "user",
      "content": "上周发生了什么？"
    }
  ],
  "stream": true,
  "contextTokenLimit": 100000     // VCP 特有参数（可选）
}
```

**关键注意点**: 
- `[[小可]]` 是 RAGDiaryPlugin 的占位符语法，代表"检索小可日记本"
- `{{VCPAllTools}}` 是 messageProcessor 的静态插件占位符，注入工具描述列表
- `{{Date}}` 是时间变量
- `contextTokenLimit` 是 VCP 专有参数，LLM API 不识别，需在发送前剥离

---

## 4. 阶段 1：上下文预算裁剪 / Stage 1 — Context Budget Pruning

**📁 位置**: `modules/contextManager.js:34` — `pruneMessages()`

```
触发条件: originalBody.contextTokenLimit !== undefined

pruneMessages(messages, limit):
  
  必须保留 (mustKeepIndices):
    ① 所有 role=system 的消息
    ② role=user 且以 '[系统提示:]' 开头的消息（VCPTavern 注入）
    ③ 最后 2 条消息（确保最新对话不被截断）
  
  裁剪策略 (从前往后删除非必须保留):
    从索引 0 开始遍历
    if (not in mustKeepIndices):
      删除该消息
      重新估算 token 数
      if (currentTokens <= limit): 停止删除
  
  Token 估算:
    仅统计文本字符数（Base64 图像数据被排除）
    粗略公式: total_chars ≈ tokens
```

**为什么要裁剪?**  
记忆注入（RAG 区块）会增加 system 消息的长度，加上对话历史可能超出 LLM 上下文窗口。裁剪保证总体在 token 预算内。

**裁剪优先级顺序**（从最先删到最后删）:
```
1. 最早的 assistant 消息
2. 最早的 user 普通消息
3. （永不删除）system 消息
4. （永不删除）最后 2 条消息
```

---

## 5. 阶段 2：角色分割 (RoleDivider) / Stage 2 — Role Divider

**📁 位置**: `modules/roleDivider.js`  
**触发条件**: `config.enableRoleDivider = true` (config.env 中 `EnableRoleDivider`)

```
功能: 将包含多角色标记的单条消息拆分成多条消息

示例:
  输入消息:
    { role: "user", content: "[小可]今天学了量子力学\n[用户]这很有趣" }
  
  分割后:
    { role: "user",      content: "[小可]今天学了量子力学" }
    { role: "assistant", content: "[用户]这很有趣" }

配置:
  config.env 中的 RoleDividerIgnoreList: 跳过指定角色的分割
  config.env 中的 RoleDividerSwitches:  按角色名控制开关
  skipCount=1: 跳过第一条系统消息（避免破坏 system 提示词）
```

**对记忆注入的影响**:  
RAGDiaryPlugin 在分割之后运行，因此能看到正确的历史楼层结构，确保 ContextVectorManager 获得真实的角色序列。

---

## 6. 阶段 3：变量替换引擎 / Stage 3 — Variable Replacement Engine

**📁 位置**: `modules/messageProcessor.js:14` — `resolveAllVariables()`

对每条消息的 `content` 字段执行递归变量展开：

### 6.1 Agent 展开 / Agent Expansion

```
正则: /\{\{([a-zA-Z0-9_:]+)\}\}/g → 提取所有占位符名称

for (alias of allAliases):
  if (agentManager.isAgent(alias)):  ← 检查 Agent/ 目录下是否存在
    agentContent = readFile(Agent/<alias>.txt)
    resolvedContent = resolveAllVariables(agentContent, ...)  ← 递归!
    text = text.replaceAll(`{{${alias}}}`, resolvedContent)

防循环引用: processingStack Set 检测 A→B→A 模式
```

**示例 / Example**:
```
system 消息中: "{{小可}}" (占位符)
  → agentManager.isAgent("小可") = true
  → 读取 Agent/小可.txt:
    "你是小可，温柔善良的AI助手。{{共同记忆}}"  ← 可嵌套
  → 递归展开 {{共同记忆}}
  → 最终: "你是小可，温柔善良的AI助手。[共同记忆内容...]"
```

**大文件内容**: `Agent/<alias>.txt` 中可以包含角色人设、世界观、性格描述等，展开后直接成为 system 消息的一部分。

### 6.2 静态插件占位符 / Static Plugin Placeholders

```javascript
// 来自 Plugin.js: plugin.capabilities.systemPromptPlaceholders
// 每个插件可在启动时注册自己的占位符

staticPlaceholderValues = pluginManager.getAllPlaceholderValues()
// Map<placeholder, content>

// 示例注册:
// {{VCPDailyTools}} → 天气+新闻工具的完整 VCP 协议描述
// {{VCPWebSearch}} → 网页搜索工具的 VCP 协议描述

// 注入: text.replace({{VCPDailyTools}}, toolDescription)
```

**效果**: AI 在 system 消息中读到完整的工具调用规范，知道有哪些工具可用以及如何调用。

### 6.3 动态上下文折叠 / Dynamic Context Folding

```
触发条件: 某个静态插件占位符的值是包含 vcp_dynamic_fold 字段的 JSON 对象

vcp_dynamic_fold 协议:
{
  "vcp_dynamic_fold": true,
  "plugin_description": "天气查询工具（当前天气、预报）",
  "fold_blocks": [
    {
      "threshold": 0.7,    // 相似度 >= 0.7: 展开完整版（含示例）
      "content": "完整工具描述含多个调用示例..."
    },
    {
      "threshold": 0.3,    // 相似度 >= 0.3: 展开简化版
      "content": "简化工具描述..."
    },
    {
      "threshold": 0.0,    // 兜底: 展开最精简版（节省 token）
      "content": "天气工具（仅名称）"
    }
  ]
}

执行逻辑:
  userVector = getSingleEmbeddingCached(最后一条 user 消息)
  descVector = kv_store 中缓存的 plugin_description 向量
  sim = cosineSimilarity(userVector, descVector)
  
  选择第一个 sim >= threshold 的 fold_block
  → 将该 block.content 注入到 system 消息中
```

**优化效果**: 与用户话题无关的工具的描述自动压缩 → 节省 token → 为记忆留出更多空间。

### 6.4 时间变量 / Time Variables

```javascript
// 注入到 system 消息中（仅 role=system 时生效）
{{Date}}     → "2026年2月23日"        (REPORT_TIMEZONE)
{{Time}}     → "下午3:00:00"
{{Today}}    → "星期一"
{{Festival}} → "丙午马年正月廿六"     (农历信息)
```

### 6.5 环境变量注入 / Env Variable Injection

```bash
# config.env 中以 Tar / Var 开头的键值对
TarUserName=张三
VarUserPreferences=我喜欢量子物理

# 在 system 消息中:
{{TarUserName}}        → "张三"
{{VarUserPreferences}} → "我喜欢量子物理"

# 如果值以 .txt 结尾，则读取文件内容
VarPersonality=xiaoke_personality.txt
{{VarPersonality}} → TVStxt/xiaoke_personality.txt 的文件内容
```

---

## 7. 阶段 4：消息预处理器 / Stage 4 — Message Preprocessors

**📁 位置**: `modules/chatCompletionHandler.js:518-529`

```javascript
// 执行顺序（顺序固定）:
for (const name of pluginManager.messagePreprocessors.keys()):
  // 跳过已在上一步特殊处理的
  if (name in ['ImageProcessor', 'MultiModalProcessor', 'VCPTavern']): continue
  
  processedMessages = await pluginManager.executeMessagePreprocessor(name, processedMessages)
```

**当前注册的 messagePreprocessor 插件**（按字母顺序执行）:
1. `RAGDiaryPlugin` — **★ 核心记忆注入**
2. 其他 messagePreprocessor 类型插件（如 `VCPTavernOC` 等）

### 7.1 RAGDiaryPlugin 记忆注入 / RAGDiaryPlugin Memory Injection

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:857` — `processMessages()`

这是记忆进入 LLM 上下文的**唯一通道**。完整流程见 [02_rag_diary_plugin.md](./02_rag_diary_plugin.md)，此处聚焦**注入机制**：

```
processMessages(messages):

  输入: messages 数组（已经过 Agent 展开 + 变量替换的版本）
  输出: 同一个 messages 数组，其中 system 消息的 content 被修改

  for (systemMsg of messages where role='system'):
    content = systemMsg.content
    
    扫描占位符:
    content = content.replace(/\[\[小可\]\]/g, async (match) => {
      return await _processRAGPlaceholder({ dbName: "小可", ... })
    })
    // 其他格式: <<小可>> 《《小可》》 [[VCP元思考:主题]]
    
    systemMsg.content = content  ← 直接修改，非副本！
  
  return messages  ← 返回的是同一个引用
```

### 7.2 RAG 区块格式详解 / RAG Block Format

RAG 区块在 system 消息的**内嵌位置**替换了原来的占位符：

**注入前 / Before injection**:
```
你是小可。我们的共同记忆：
[[小可]]

你可以使用以下工具：
{{VCPAllTools}}
```

**注入后 / After injection**:
```
你是小可。我们的共同记忆：
<!-- VCP_RAG_BLOCK_START {"dbName":"小可","modifiers":"::TagMemo","k":5} -->

[--- 从"小可日记本"中检索到的相关记忆片段 ---]
* [2026-02-20] - 小可
  今天学了量子力学导论，薛定谔的猫好有趣...
  Tag: 量子力学, 物理, 学习

* [2026-02-18] - 小可
  今天重构了代码，把三个函数合并成一个...
  Tag: 编程, Python, 重构

* [2026-02-15] - 小可
  上周的量子纠缠实验读到了一半...
  Tag: 量子力学, 实验

[--- 记忆片段结束 ---]
<!-- VCP_RAG_BLOCK_END -->

你可以使用以下工具：
[工具描述...]
```

**HTML 注释格式的选择理由 / Why HTML comment format**:
1. LLM 能读取 HTML 注释内容（它们在 LLM 眼中就是普通文本）
2. 注释语法对 LLM 有隐含的"元信息"语义暗示
3. VCP 后端可以通过精确的 `VCP_RAG_BLOCK_START/END` 正则查找并更新区块
4. 不干扰其他 Markdown 格式

**元数据嵌入 / Metadata embedding**:
```json
// VCP_RAG_BLOCK_START 后的 JSON 元数据：
{
  "dbName": "小可",           // 日记本名称（用于刷新）
  "modifiers": "::TagMemo",  // 修饰符（记录检索策略）
  "k": 5                     // 实际返回条数
}
```
这些元数据在**对话中 RAG 刷新**时被解析重用。

---

## 8. 阶段 5：LLM API 调用 / Stage 5 — LLM API Call

**📁 位置**: `modules/chatCompletionHandler.js:539` — `fetchWithRetry()`

经过所有预处理后，最终的 `messages` 数组被发送给 LLM：

```javascript
POST apiUrl + '/v1/chat/completions'
{
  "model": "gemini-2.0-flash",
  "messages": [
    {
      "role": "system",
      "content": "你是小可，温柔善良的AI助手...\n\n<!-- VCP_RAG_BLOCK_START {\"dbName\":\"小可\"} -->\n[从\"小可日记本\"中检索到的相关记忆片段]\n* [2026-02-20]...\n<!-- VCP_RAG_BLOCK_END -->\n\n你可以使用以下工具:\n[工具描述...]\n\n今天日期: 2026年2月23日星期一"
    },
    {
      "role": "user",
      "content": "上周发生了什么？"
    }
  ],
  "stream": true
}
```

**LLM 视角 / LLM's perspective**:  
LLM 的"记忆"就是 system 消息中的文本。它通过**注意力机制 (Attention Mechanism)** 在单次推理中关注到 RAG 区块中的记忆片段，并根据这些信息生成响应。

**LLM 不知道 / What the LLM doesn't know**:
- 这些记忆是从 SQLite 数据库检索来的
- 存在向量搜索过程
- 这只是所有日记的一个子集

---

## 9. 阶段 6：VCP 工具执行循环 / Stage 6 — VCP Tool Execution Loop

**📁 位置**: `modules/handlers/streamHandler.js:195`

当 AI 的输出包含 VCP 工具调用（`<<<[TOOL_REQUEST]>>>...<<<[END_TOOL_REQUEST]>>>`），进入递归循环：

### 9.1 工具结果注入 / Tool Result Injection

```
循环每轮 (recursionDepth < maxVCPLoopStream):

  步骤 A: 将 AI 这轮输出追加到 messages
    currentMessagesForLoop.push({
        role: 'assistant',
        content: currentAIContentForLoop  // AI 这轮生成的文本（含工具调用语法）
    })

  步骤 B: 解析工具调用 (ToolCallParser.parse)
    toolCalls = [...VCP工具调用列表...]
    if (toolCalls.length === 0): break  // 没有工具调用，循环结束

  步骤 C: 执行工具
    results = await toolExecutor.executeAll(normalCalls, clientIp)
    // 例: DailyNoteWrite, WebSearch, GetWeather 等

  步骤 D: ★ 将工具结果注入到 messages 中（以 user 角色）
    payload = `<!-- VCP_TOOL_PAYLOAD -->\n${JSON.stringify(toolResults)}`
    currentMessagesForLoop.push({
        role: 'user',
        content: payload
    })
    // LLM 下一轮会读到这个 user 消息，知道工具执行结果

  步骤 E: ★ RAG 刷新（可选）
    见 §9.2

  步骤 F: 再次调用 LLM API（携带更新后的 messages）
    POST apiUrl/v1/chat/completions { messages: currentMessagesForLoop, ... }
```

**消息结构在工具循环后的变化**:
```
第0轮:
  [system: 角色+记忆], [user: "搜索量子力学"]

第1轮后（AI 调用了 WebSearch）:
  [system: 角色+记忆], [user: "搜索量子力学"],
  [assistant: "我来搜索...<<<[TOOL_REQUEST]>>>WebSearch..."],
  [user: "<!-- VCP_TOOL_PAYLOAD -->\n{"results": "...量子纠缠搜索结果..."}"]

第2轮后（AI 调用了 DailyNoteWrite）:
  ...前面的消息...,
  [assistant: "根据搜索结果...<<<[TOOL_REQUEST]>>>DailyNoteWrite..."],
  [user: "<!-- VCP_TOOL_PAYLOAD -->\n{"status": "success", ...}"]
```

### 9.2 RAG 区块对话中刷新 / In-Loop RAG Refresh

**📁 位置**: `modules/handlers/streamHandler.js:338`  
**触发条件**: `config.RAGMemoRefresh = true`

```
在每次工具执行完成后、下一轮 LLM 调用前执行:

_refreshRagBlocksIfNeeded(currentMessagesForLoop, newContext, pluginManager):

  newContext = {
    lastAiMessage: currentAIContentForLoop,   // AI 这轮输出
    toolResultsText: JSON.stringify(toolResults) // 工具执行结果
  }

  扫描 messages 中所有包含 VCP_RAG_BLOCK_START 的消息:
    (包括 system, assistant, user 角色的消息)
  
  for (每个 RAG_BLOCK):
    解析 metadata = { dbName, modifiers, k }
    
    // 寻找最近的"真实"用户问题（排除系统消息格式）
    originalUserQuery = last user message before this block
    
    newBlock = await ragPlugin.refreshRagBlock(
      metadata, newContext, originalUserQuery
    )
    // refreshRagBlock 使用 "AI这轮输出 + 工具结果" 作为新查询
    // 重新检索 → 返回新的 VCP_RAG_BLOCK 字符串
    
    message.content = message.content.replace(oldBlock, () => newBlock)
    // ★ 使用箭头函数避免 $ 符号被正则特殊解析的 Bug
```

**刷新的意义**:  
工具调用产生了新的上下文信息（如搜索结果、写入的日记）。刷新确保 LLM 在后续轮次中能看到基于**最新对话状态**检索的记忆，而不是第一轮时基于用户原始问题的记忆。

---

## 10. 最终上下文结构示意图 / Final Context Structure Diagram

这是 LLM 在第 N 轮（有工具调用）收到的完整 `messages` 结构：

```
messages = [

  ① system (唯一, 始终存在):
    "[角色人设：小可的人格...N字]
     
     <!-- VCP_RAG_BLOCK_START {"dbName":"小可","k":5} -->
     [从"小可日记本"中检索到的相关记忆片段]
     * [2026-02-20] 今天学了量子力学...
     * [2026-02-18] 关于代码重构...
     <!-- VCP_RAG_BLOCK_END -->
     
     <!-- VCP_RAG_BLOCK_START {"dbName":"知识库","k":3} -->
     [从"知识库日记本"中检索到的相关记忆片段]
     * [量子力学基础知识...]
     <!-- VCP_RAG_BLOCK_END -->
     
     [VCP 工具调用协议说明...]
     [工具1: DailyNoteWrite...]
     [工具2: WebSearch...]
     
     [当前时间: 2026年2月23日 星期一 15:00]
     [用户名: 张三]"

  ② user (可能是 VCPTavern 注入的系统提示):
    "[系统提示:] 请注意以下事项..."
    (如果没有, 则无此条)

  ③ user: "上周发生了什么？"       ← 用户的原始问题

  ④ assistant: "我来查看一下...    ← AI 第1轮输出
     <<<[TOOL_REQUEST]>>>           （含工具调用）
     tool_name:「始」DailyNoteWrite「末」
     ...「末」
     <<<[END_TOOL_REQUEST]>>>"

  ⑤ user:                          ← 工具结果
     "<!-- VCP_TOOL_PAYLOAD -->
      {"status": "success", "message": "Diary saved to..."}"

  ⑥ assistant: "我已经记录下来了..." ← AI 第2轮输出
     <<<[TOOL_REQUEST]>>>
     tool_name:「始」WebSearch「末」
     ...
     <<<[END_TOOL_REQUEST]>>>

  ⑦ user:                          ← 工具结果
     "<!-- VCP_TOOL_PAYLOAD -->
      {"results": "量子纠缠搜索结果..."}"
]
```

**各消息类型的"记忆作用" / Memory role of each message type**:

| 消息 | 角色 | 记忆来源 |
|---|---|---|
| ① system | 长期记忆 + 角色人设 | RAG 检索 + Agent/*.txt |
| ④⑥ assistant | 当前轮次 AI 的推理链 | AI 自身输出（工具调用可见性）|
| ⑤⑦ user (TOOL_PAYLOAD) | 工具执行结果（短期记忆）| 工具插件实时执行 |
| ③ user | 用户的输入 | 对话本身 |

---

## 11. 记忆 Token 占用分析 / Memory Token Budget Analysis

以典型配置为例估算：

```
System 消息结构:
  角色人设 (Agent/小可.txt)    : ~500 tokens
  RAG 区块 (日记检索 5条×100字) : ~800 tokens  ← 记忆
  工具描述 (VCPAllTools)        : ~300 tokens
  时间/变量                      : ~50 tokens
  ────────────────────────────────────────────
  小计                           : ~1650 tokens

User 消息历史:
  对话历史 (10轮×100字)          : ~600 tokens
  ────────────────────────────────────────────
  小计                           : ~600 tokens

LLM 最终收到的 tokens          : ~2250 tokens

在 1M token 上下文窗口中占比   : ~0.2%
(充裕，但注意: RAG 区块越多，占用越大)
```

**记忆 token 控制建议**:
```bash
# config.env 或 rag_params.json

# 每个占位符召回条数 (默认动态计算，约 3~8)
# 减少 k → 节省 token
# 增加 k → 更多记忆但可能超 context

# 多个日记本时
[[小可:0.5]][[知识库:0.5]]    # k 乘以 0.5，各减半
```

---

## 12. 调试完整上下文 / Debugging the Full Context

### 12.1 输出调试日志

```bash
# config.env
DEBUG_MODE=true               # 启用详细日志

# 日志文件位置: logs/<timestamp>.json (由 writeDebugLog 写入)
```

**各阶段日志文件**:

| 阶段 | 日志文件名 | 内容 |
|---|---|---|
| 入口 | `LogInput` | 原始请求 body |
| 角色分割后 | `LogAfterInitialRoleDivider` | 分割后的 messages |
| 变量替换后 | `LogAfterVariableProcessing` | Agent 展开 + 变量替换完成 |
| 预处理器后 | `LogAfterPreprocessors` | RAG 注入完成 |
| 发送给 LLM 前 | `LogOutputAfterProcessing` | **完整最终 messages** |

```bash
# 查看记忆注入后的系统提示
cat logs/LogAfterPreprocessors.json | jq '.messages[0].content' | grep -A 20 "VCP_RAG_BLOCK_START"
```

### 12.2 验证 RAG 区块存在

```bash
# 检查 RAG 注入成功的标志
grep "VCP_RAG_BLOCK_START" logs/LogAfterPreprocessors.json

# 检查注入了多少条记忆
grep -c "\* \[2026-" logs/LogAfterPreprocessors.json

# 检查工具执行循环次数
grep "\[VCP Stream Loop\]" server.log | head -20
```

### 12.3 验证 RAG 刷新

```bash
# RAG 刷新触发
grep "\[VCP Refresh\]" server.log
# 输出: [VCP Refresh] 消息[0]中发现 2 个 RAG 区块，准备刷新...
# 输出: [VCP Refresh] ✅ 对话历史中的 RAG 记忆区块已根据新上下文成功刷新。

# 如果没看到刷新日志，检查
grep "RAGMemoRefresh" .env   # 确认配置已启用
```

### 12.4 常见问题排查

| 症状 | 可能原因 | 修复方法 |
|---|---|---|
| AI 不知道记忆内容 | RAG 区块未被注入 | 检查占位符语法 `[[日记本名]]` 是否在 system 消息中 |
| AI 记忆过时（工具调用后）| RAGMemoRefresh=false | 在 config.env 中设置 `RAGMemoRefresh=true` |
| 系统消息超长 | RAG 区块太多或 Agent 文件太大 | 减小 k 值或设置 `contextTokenLimit` |
| AI 看不到最近日记 | 文件未被索引 | 检查 `VectorStore/` 是否有新的 `.usearch` 文件 |
| RAG 刷新后记忆没变 | 缓存命中 | 清空 `queryResultCache` 或等待 1 小时过期 |

---

## 13. 设计对比：与 LangChain / LangGraph 的区别 / Design Comparison

| 特性 | VCP | LangChain ConversationBufferMemory | LangGraph Checkpointer |
|---|---|---|---|
| 记忆注入位置 | system 消息文本 (HTML 注释区块) | system/user 消息末尾追加 | 通过 state 注入 |
| 持久化存储 | SQLite + VexusIndex 文件 | 内存 (Redis/外部可选) | SQLite/Redis/PostgreSQL |
| 检索方式 | 向量KNN + TagMemo + BM25 混合 | 简单截断或 VectorStore | 依赖 retriever |
| 检索粒度 | 日记 chunk 级 (~100-500字) | 完整消息级 | 可配置 |
| 工具循环中刷新 | ✅ (RAGMemoRefresh) | ❌ 通常不刷新 | 可手动实现 |
| 多层记忆 | ✅ (5层: 即时→梦境) | ❌ 单层 | ❌ 单层 |
| 记忆来源 | 本地 .md 文件 | 对话历史 | graph state |
| AI 写入记忆 | ✅ (DailyNoteWrite 工具) | ❌ | ❌ (需自定义) |
| Token 预算控制 | ✅ contextTokenLimit + k 动态调整 | 有限 | 需手动 |
| 调试工具 | 多阶段日志文件 + AdminPanel VCP Info | LangSmith | LangGraph Studio |

**VCP 的独特优势 / VCP's unique strengths**:
1. **AI 是记忆的主动生产者** — AI 可以调用 DailyNoteWrite 主动写入日记，形成记忆的正反馈循环
2. **多粒度记忆** — 从即时对话到 AgentDream 梦境整合，5个时间尺度
3. **本地化** — 所有记忆存储在本地 `.md` 文件中，完全可读可编辑
4. **对话中动态刷新** — 工具调用后自动重检索，记忆始终与当前上下文对齐

---

> ⬆️ [README.md](./README.md) | ⬅️ [08_write_plugins.md](./08_write_plugins.md)

---

## Security Summary / 安全摘要

本文档仅涉及文档变更，无代码修改，不引入新的安全风险。  
This document only contains documentation changes; no code was modified; no new security risks are introduced.
