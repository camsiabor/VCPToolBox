# 02 RAGDiaryPlugin — RAG 日记插件全流程详解
# 02 RAGDiaryPlugin — RAG Diary Plugin Full Pipeline

> **所属 / Parent**: [README.md](./README.md)  
> **源文件 / Source**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js` (2948 行 / lines)  
> **类型 / Type**: `messagePreprocessor` 插件  
> **依赖 / Deps**: `KnowledgeBaseManager`, `ContextVectorManager`, `AIMemoHandler`, `MetaThinkingManager`, `SemanticGroupManager`, `TimeExpressionParser`

---

## 目录 / Table of Contents

1. [插件定位 / Plugin Role](#1-插件定位--plugin-role)
2. [初始化流程 / Initialization Flow](#2-初始化流程--initialization-flow)
3. [processMessages — 主入口详解 / Main Entry](#3-processmessages--主入口详解--main-entry)
4. [占位符语法大全 / Placeholder Syntax Reference](#4-占位符语法大全--placeholder-syntax-reference)
5. [_processSingleSystemMessage — 系统消息处理器](#5-_processsinglesystemmessage--系统消息处理器)
6. [_processRAGPlaceholder — 核心 RAG 检索](#6-_processragplaceholder--核心-rag-检索)
7. [聚合检索 _processAggregateRetrieval](#7-聚合检索-_processaggregateretrieval)
8. [时间感知双路召回 / Time-Aware Dual-Path Retrieval](#8-时间感知双路召回--time-aware-dual-path-retrieval)
9. [语义组增强 / Semantic Group Enhancement](#9-语义组增强--semantic-group-enhancement)
10. [对话中 RAG 刷新 / In-Conversation RAG Refresh](#10-对话中-rag-刷新--in-conversation-rag-refresh)
11. [Rerank 集成 / Rerank Integration](#11-rerank-集成--rerank-integration)
12. [三级缓存系统 / Three-Level Cache System](#12-三级缓存系统--three-level-cache-system)
13. [VCP Info 广播 / VCP Info Broadcast](#13-vcp-info-广播--vcp-info-broadcast)
14. [结果格式化 / Result Formatting](#14-结果格式化--result-formatting)
15. [工具方法参考 / Utility Method Reference](#15-工具方法参考--utility-method-reference)

---

## 1. 插件定位 / Plugin Role

RAGDiaryPlugin 是 VCP 记忆系统的**入口大脑**。它在每次对话请求时运行，负责：

1. **理解意图** — 分析查询向量、提取时间意图、计算动态参数
2. **调度检索** — 根据占位符语法路由到不同的检索策略
3. **注入记忆** — 将检索结果格式化为 HTML 注释块注入 system prompt
4. **广播状态** — 通过 VCP Info WebSocket 实时推送检索结果到管理面板

---

## 2. 初始化流程 / Initialization Flow

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:277` — `initialize()`

```javascript
// Plugin.js 调用 initialize(config, dependencies)
// dependencies.vectorDBManager = KnowledgeBaseManager 实例

initialize(config, dependencies):
  ① this.vectorDBManager = dependencies.vectorDBManager
  ② await loadConfig()          读取 config.env (RAG 参数)
  ③ await loadRagParams()       读取 rag_params.json (热调控参数)
  ④ 初始化子模块:
     - this.timeParser           = new TimeExpressionParser()
     - this.semanticGroups       = new SemanticGroupManager(this)
     - this.contextVectorManager = new ContextVectorManager(this)
     - this.metaThinkingManager  = new MetaThinkingManager(this)
     - this.aiMemoHandler        = new AIMemoHandler(this, aiMemoCache)
  ⑤ await aiMemoHandler.loadConfig()
  ⑥ await metaThinkingManager.loadConfig()
  ⑦ 启动缓存清理定时器 (30分钟间隔)
  ⑧ 启动 rag_params.json 热调控监听
```

---

## 3. processMessages — 主入口详解 / Main Entry

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:857`

这是 Plugin.js 调用的标准 `messagePreprocessor` 接口。

```
processMessages(messages, pluginConfig)
  │
  ├─① 检测全局 AIMemo 许可证              [line 864-873]
  │   system 消息包含 "[[AIMemo=True]]" → isAIMemoLicensed = true
  │
  ├─② 扫描需要处理的 system 消息          [line 875-880]
  │   正则: /\[\[.*日记本.*\]\]|<<.*日记本.*>>|《《.*日记本.*》》
  │         |\{\{.*日记本\}\}|\[\[VCP元思考.*\]\]|\[\[AIMemo=True\]\]/
  │
  ├─③ 提取查询文本                        [line 940-960]
  │   userContent = 最近一条 user 消息
  │   aiContent   = 最近一条 assistant 消息（截取前 500 字符）
  │   combinedQuery = userContent + ' ' + aiContent
  │
  ├─④ 获取查询向量                        [line 963]
  │   queryVector = await getSingleEmbeddingCached(combinedQuery)
  │
  ├─⑤ 上下文向量管理 (V4)                [line 974-978]
  │   contextVectorManager.updateContext(messages, {allowApi: false})
  │   historySegments = contextVectorManager.segmentContext(messages)
  │
  ├─⑥ 时间意图解析                        [line 967-970]
  │   timeRanges = timeParser.extractTimeRanges(userContent)
  │
  ├─⑦ 动态参数计算                        [line 972]
  │   { k, tagWeight, tagTruncationRatio, metrics } =
  │     await _calculateDynamicParams(queryVector, userContent, aiContent)
  │
  └─⑧ 逐个处理每条 system 消息           [line 985-1036]
      for (systemMsg of systemMessages):
        processedContent = await _processSingleSystemMessage(
          systemMsg.content, queryVector, ...
        )
        systemMsg.content = processedContent
```

---

## 4. 占位符语法大全 / Placeholder Syntax Reference

| 语法 / Syntax | 触发效果 / Effect |
|---|---|
| `[[日记本名]]` | 标准向量 RAG 检索 |
| `[[A\|B\|C]]` | 聚合多库检索 (A、B、C 三个日记本) |
| `[[日记本名::TagMemo0.3]]` | 指定 TagWeight=0.3 |
| `[[日记本名::TagMemo]]` | 使用动态 TagWeight |
| `[[日记本名::Time]]` | 时间感知双路召回 |
| `[[日记本名::Group]]` | 语义组增强检索 |
| `[[日记本名::Rerank]]` | 启用外部 Rerank |
| `[[日记本名::Meta]]` | 仅元思考，不做向量检索 |
| `[[日记本名::AIMemo]]` | AI 驱动全量召回 (需 `[[AIMemo=True]]`) |
| `[[日记本名:1.5]]` | K 倍率乘以 1.5 |
| `[[VCP元思考:主题名]]` | 元思考递归推理链 |
| `<<日记本名>>` | 标准检索（双尖括号格式） |
| `《《日记本名》》` | 混合模式（中文双书名号格式） |
| `[[AIMemo=True]]` | 全局 AIMemo 许可证（不产生输出） |

**修饰符可以组合 / Modifiers can combine**:
```
[[小可::TagMemo::Group::Rerank:2.0]]
```

---

## 5. _processSingleSystemMessage — 系统消息处理器

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:1037`

```
输入: 一条 system 消息的 content 字符串
输出: 替换了所有占位符的 content 字符串

处理步骤:
  1. 移除 [[AIMemo=True]] (仅作许可证，不应出现在输出中)

  2. 扫描 [[...]] 格式的占位符 (单日记本 + 聚合)
     → 解析 _parseAggregateSyntax(rawName, modifiers)
     → 路由到对应处理器:
       a. isAggregate + AIMemo → aiMemoHandler.processAIMemoAggregated(dbNames)
       b. isAggregate + 标准   → _processAggregateRetrieval(options)
       c. 单库 + AIMemo         → aiMemoHandler.processAIMemo(dbName)
       d. 单库 + 标准           → _processRAGPlaceholder(options)

  3. 扫描 <<...>> 格式的占位符 (同上逻辑)

  4. 扫描 《《...》》 格式的占位符 (同上逻辑)

  5. 扫描 [[VCP元思考:主题]] 占位符
     → metaThinkingManager.processMetaThinking(theme, queryVector)

  6. await Promise.all(processingPromises)  ← 并行执行所有检索
  
  7. 将结果替换回 content 中对应的占位符位置
```

---

## 6. _processRAGPlaceholder — 核心 RAG 检索

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:1840`

这是标准向量检索的核心方法，也是最复杂的流程：

```
_processRAGPlaceholder(options):
  {dbName, modifiers, queryVector, dynamicK, timeRanges,
   defaultTagWeight, tagTruncationRatio, historySegments,
   contextDiaryPrefixes}

  ① 生成缓存键 + 查询缓存               [line 1859-1882]
    cacheKey = MD5(userContent + aiContent + dbName + modifiers + dynamicK)
    if (hit) → 直接返回缓存内容 + 广播缓存 VCP Info

  ② 解析修饰符                          [line 1887-1906]
    kMultiplier   = _extractKMultiplier(modifiers)  // ":1.5" 等
    useTime       = modifiers.includes('::Time')
    useGroup      = modifiers.includes('::Group')
    useRerank     = modifiers.includes('::Rerank')
    tagWeight     = 从 modifiers 提取或使用 defaultTagWeight
    finalK        = round(dynamicK × kMultiplier)
    kForSearch    = useRerank ? finalK × rerankMultiplier : finalK
                    + dedupBuffer (contextDiaryPrefixes.size)

  ③ 语义组增强 (if useGroup)             [line 1921-1926]
    activatedGroups = semanticGroups.detectAndActivateGroups(userContent)
    if (groups found):
      finalQueryVector = semanticGroups.getEnhancedVector(userContent, groups)

  ④ 原子级 Tag 感应                     [line 1933-1946]
    // 先用 tagBoost=0 调用一次 applyTagBoost 感应出 Tag
    // 再将 Tag 作为 coreTags 传入正式搜索 (双重增强)
    boostResult = vectorDBManager.applyTagBoost(queryVec, tagWeight, [])
    coreTagsForSearch = _truncateCoreTags(
      boostResult.info.matchedTags, tagTruncationRatio, metrics
    )

  ⑤ 路由到具体检索模式:

    [A] 时间感知模式 (useTime=true):  见 §8
    [B] Shotgun 模式 (V4, historySegments 非空):  见 §6.1
    [C] 标准单向量模式:
        results = vectorDBManager.search(
          dbName, finalQueryVector, kForSearch, tagWeight, coreTagsForSearch
        )
        results = _filterContextDuplicates(results, contextDiaryPrefixes)
        results = await vectorDBManager.deduplicateResults(results, queryVec)
        if (useRerank): results = _rerankDocuments(userContent, results, finalK)
        else: results = results.slice(0, finalK)

  ⑥ 格式化结果:
    if (useGroup): formatGroupRAGResults(results, displayName, activatedGroups)
    else:          formatStandardResults(results, displayName)
    → <!-- VCP_RAG_BLOCK_START {metadata} -->...<!-- VCP_RAG_BLOCK_END -->

  ⑦ 广播 VCP Info:
    pushVcpInfo({ type: 'RAG_RETRIEVAL_DETAILS', dbName, results, ... })

  ⑧ 写入缓存 (含 VCP Info 副本)
```

### 6.1 Shotgun V4 路径

```
当 historySegments 非空时:

searchVectors = [
  { vector: finalQueryVector,  type: 'current',   k: finalK    },
  { vector: historySegments[n-3], type: 'history_0', k: ceil(finalK/2) },
  { vector: historySegments[n-2], type: 'history_1', k: ceil(finalK/2) },
  { vector: historySegments[n-1], type: 'history_2', k: ceil(finalK/2) },
]

// 并行搜索
results = await Promise.all(
  searchVectors.map(qv => vectorDBManager.search(dbName, qv.vector, qv.k, ...))
)

// 合并 + 去重
flattenedResults = flatten(results)
flattenedResults = _filterContextDuplicates(flattenedResults)
uniqueResults = await vectorDBManager.deduplicateResults(flattenedResults, queryVec)
```

---

## 7. 聚合检索 _processAggregateRetrieval

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:1604`

当占位符语法为 `[[A|B|C]]` 时触发，按语义相似度动态分配 K 值：

```
_processAggregateRetrieval({ diaryNames, kMultiplier, queryVector, ... }):

  ① 计算每个日记本与查询的余弦相似度
    for (name of diaryNames):
      diaryVec = vectorDBManager.getDiaryNameVector(name)
      sim = cosineSimilarity(queryVector, diaryVec)
      diaryScores.push({ name, similarity: sim })

  ② Softmax 温度分配 K (temperature = 3.0)
    expScores = diaryScores.map(d => exp(d.similarity × temperature))
    totalExpScore = sum(expScores)
    totalK = round(dynamicK × kMultiplier)
    reservedK = minKPerDiary × diaryNames.length  // 保底每库至少 1 条

    for (i):
      k_i = reservedK/n + round((totalK - reservedK) × expScores[i] / totalExpScore)
    // 高相似度的日记本获得更多 K 配额

  ③ 并行对每个日记本执行 _processRAGPlaceholder()
    await Promise.all(kAllocations.map(alloc => ...))

  ④ 按 score 排序合并，截取 totalK 条
```

---

## 8. 时间感知双路召回 / Time-Aware Dual-Path Retrieval

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:1951`

当修饰符包含 `::Time` 且查询中有时间表达式（如"今天"、"上周"）时触发：

```
V5 平衡双路召回 (Balanced Dual-Path Retrieval):

路径 A — 语义召回 (60%):
  kSemantic = ceil(finalK × 0.6)
  ragResults = vectorDBManager.search(dbName, queryVec, kSemantic, ...)
  → 标注 source: 'rag'

路径 B — 时间路召回 (40%):
  kTime = finalK - kSemantic
  timeFilePaths = []
  for (timeRange of timeRanges):
    paths = _getTimeRangeFilePaths(dbName, timeRange)
    timeFilePaths.push(...paths)
  
  // 对时间路文件进行相关性排序
  timeResults = [] 
  for (filePath of unique timeFilePaths):
    content = fs.readFile(filePath)
    chunks = SQL: SELECT content FROM chunks WHERE file_id = fileId
    for (chunk of chunks):
      sim = cosineSimilarity(queryVec, chunkVec)  // 取 chunk 向量
      timeResults.push({ text, date, source: 'time', sim })
  sort by sim DESC, take kTime

合并: ragResults + timeResults
去重: _filterContextDuplicates()
格式化: formatCombinedTimeAwareResults()
```

---

## 9. 语义组增强 / Semantic Group Enhancement

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:1921`  
→ 详见 [06_semantic_group_aimemo.md](./06_semantic_group_aimemo.md)

```
// 激活检测
activatedGroups = semanticGroups.detectAndActivateGroups(userContent)
// 关键词大小写不敏感包含匹配

// 向量增强
if (activatedGroups.size > 0):
  enhancedQueryVec = semanticGroups.getEnhancedVector(
    userContent, activatedGroups, queryVector
  )
  // 加权平均: queryVec(weight=1.0) + groupVecs(weight=groupWeight×strength)
```

---

## 10. 对话中 RAG 刷新 / In-Conversation RAG Refresh

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:1766` — `refreshRagBlock()`

```
触发时机: 每次 VCP 工具调用循环结束后 (by chatCompletionHandler)

refreshRagBlock(metadata, contextData, originalUserQuery):
  ① 从 metadata 提取 { dbName, topK, modifiers }
  ② 构建新查询文本:
     newQuery = contextData.latestUserMessage  ← 最新 user 消息
  ③ 调用 _processRAGPlaceholder({...}) 执行完整 RAG 检索
  ④ 返回新的 VCP_RAG_BLOCK 字符串

// 在 chatCompletionHandler 中:
// 正则替换 messages 中的所有 VCP_RAG_BLOCK:
const ragBlockRegex = /<!-- VCP_RAG_BLOCK_START (.*?) -->([\s\S]*?)<!-- VCP_RAG_BLOCK_END -->/g
messages = messages.map(msg => ({
    ...msg,
    content: msg.content.replace(ragBlockRegex, newRagBlock)
}))
```

---

## 11. Rerank 集成 / Rerank Integration

**📁 位置**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:2339` — `_rerankDocuments()`

```javascript
// 配置检查 (JIT 检查，而非启动时标志)
if (!rerankConfig.url || !rerankConfig.apiKey || !rerankConfig.model)
    return documents.slice(0, originalK)

// 断路器: 1分钟内 ≥5 次失败 → 自动跳过
if (recentFailures >= 5)
    return documents.slice(0, originalK)

// 按最大 token 分批
batches = splitByTokenLimit(documents, rerankConfig.maxTokensPerBatch)

for (batch of batches):
    response = await POST rerankConfig.url {
        model: rerankConfig.model,
        query: queryText,
        documents: batch.map(d => d.text),
        top_n: originalK
    }
    → 提取 results[].index 和 results[].relevance_score

// 按 rerank_score 排序，取前 originalK 个
return topK.map(r => ({ ...documents[r.index], rerank_score: r.score }))
```

---

## 12. 三级缓存系统 / Three-Level Cache System

| 缓存名 / Cache | 存储对象 | TTL | 最大条目 | 配置键 |
|---|---|---|---|---|
| `queryResultCache` | 完整 RAG 结果 + VCP Info | 1 小时 | 200 | `RAG_CACHE_MAX_SIZE` |
| `embeddingCache` | 文本 → 向量 | 2 小时 | 500 | `EMBEDDING_CACHE_MAX_SIZE` |
| `aiMemoCache` | AIMemo 召回结果 | 30 分钟 | 50 | `AIMEMO_CACHE_MAX_SIZE` |

**缓存键生成 / Cache key generation**:
```javascript
_generateCacheKey({ userContent, aiContent, dbName, modifiers, dynamicK }) {
    return crypto.createHash('md5')
        .update(`${userContent}|${aiContent}|${dbName}|${modifiers}|${dynamicK}`)
        .digest('hex')
}
```

**缓存失效触发器 / Cache invalidation triggers**:
```javascript
// 配置文件变更 (rag_params.json 或 config.env) → 清空查询缓存
if (currentConfigHash !== this.lastConfigHash) {
    this.queryResultCache.clear()
    this.lastConfigHash = currentConfigHash
}
```

---

## 13. VCP Info 广播 / VCP Info Broadcast

每次 RAG 检索完成后，插件会通过 WebSocket 向管理面板实时推送详情：

```javascript
pushVcpInfo({
    type: 'RAG_RETRIEVAL_DETAILS',
    dbName: dbName,
    query: combinedQueryForDisplay,
    k: finalK,
    tagWeight: tagWeight,
    coreTagsUsed: coreTagsForSearch,
    results: cleanedResults,  // _cleanResultsForBroadcast() 清理 PII
    metrics: { L, R, S, beta, dynamicK, truncationRatio },
    fromCache: false
})
```

结果清理 `_cleanResultsForBroadcast()`:
- 截断 `text` 超长内容 (保留前 500 字符)
- 移除 `fullPath`（隐私信息）
- 保留 `score`, `sourceFile`, `matchedTags`, `rerank_score`

---

## 14. 结果格式化 / Result Formatting

### 14.1 标准结果 / Standard Results

```
formatStandardResults(results, displayName, metadata):
  <!-- VCP_RAG_BLOCK_START {"dbName":"小可","k":5} -->
  [--- 从"小可日记本"中检索到的相关记忆片段 ---]
  * [2026-02-20] - 小可
    今天完成了 VCP 文档...
    Tag: 量子力学, 物理
  * [2026-02-18] - 小可
    关于代码重构...
  [--- 记忆片段结束 ---]
  <!-- VCP_RAG_BLOCK_END -->
```

### 14.2 时间感知结果 / Time-Aware Results

```
formatCombinedTimeAwareResults(results, timeRanges, dbName, metadata):
  [--- "小可日记本" 多时间感知检索结果 ---]
  [合并查询的时间范围: "2026-02-20 ~ 2026-02-20" 和 "2026-02-13 ~ 2026-02-19"]
  [统计: 共找到 7 条不重复记忆 (语义相关 4条, 时间范围 3条)]
  
  【语义相关记忆】
  * [2026-02-20] ...
  
  【时间范围记忆】
  * [2026-02-18] ...
```

### 14.3 语义组结果 / Semantic Group Results

```
formatGroupRAGResults(results, displayName, activatedGroups, metadata):
  [激活的语义组:]
    • 量子力学 (80%激活): 匹配到 "量子纠缠, 叠加态"
  
  [检索到 5 条相关记忆]
  * ...
```

---

## 15. 工具方法参考 / Utility Method Reference

| 方法 | 描述 |
|---|---|
| `getSingleEmbedding(text)` | 调用 Embedding API（无缓存）|
| `getSingleEmbeddingCached(text)` | 调用 Embedding API（带 LRU 缓存）|
| `_getEmbeddingFromCacheOnly(text)` | 只查缓存，不触发 API |
| `_parseAggregateSyntax(rawName, mods)` | 解析 `A\|B\|C` 语法 |
| `_extractKMultiplier(modifiers)` | 提取 `:1.5` 倍率修饰符 |
| `_truncateCoreTags(tags, ratio, metrics)` | Tag 列表截断去噪 |
| `_filterContextDuplicates(results, prefixes)` | 工具调用上下文去重 |
| `_sigmoid(x)` | σ(x) = 1/(1+e^(-x)) |
| `_stripHtml(html)` | 移除 HTML 标签 (cheerio) |
| `_stripEmoji(text)` | 移除 Emoji |
| `_stripToolMarkers(text)` | 移除 VCP 工具调用标记 |
| `_isLikelyBase64(str)` | 检测是否为 Base64 数据 |
| `_jsonToMarkdown(obj)` | JSON 转 Markdown 文本（减少向量噪音）|
| `_estimateTokens(text)` | 估算 token 数（中英混合）|
| `cosineSimilarity(v1, v2)` | 余弦相似度 |
| `getDiaryContent(charName)` | 读取近 N 天日记全文（AIMemo 用）|
| `_getFileHash(path)` | 文件 MD5 哈希 |

---

> ⬆️ [README.md](./README.md) | ⬅️ [01_knowledge_base_manager.md](./01_knowledge_base_manager.md) | ➡️ [03_epa_residual_result.md](./03_epa_residual_result.md)
