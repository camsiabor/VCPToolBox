# 13 AI Agent 记忆系统终极深度解析 / AI Agent Memory System — Ultimate Deep Dive

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `KnowledgeBaseManager.js`, `EPAModule.js`, `ResidualPyramid.js`, `ResultDeduplicator.js`, `EmbeddingUtils.js`, `TextChunker.js`, `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js`, `Plugin/RAGDiaryPlugin/AIMemoHandler.js`, `Plugin/RAGDiaryPlugin/ContextVectorManager.js`, `Plugin/RAGDiaryPlugin/MetaThinkingManager.js`, `Plugin/RAGDiaryPlugin/SemanticGroupManager.js`, `Plugin/DailyNoteWrite/daily-note-write.js`, `Plugin/LightMemo/LightMemo.js`, `Plugin/ThoughtClusterManager/ThoughtClusterManager.js`, `Plugin/AgentDream/AgentDream.js`  
> **版本 / Version**: VCP 6.4 TagMemo V4 · 2026-02-23  
> **前置阅读 / See Also**: [10_agent_memory](../10_agent_memory/README.md), [11_agent_rag](../11_agent_rag/README.md), [03_memory_rag](../03_memory_rag/README.md)

---

## 📚 子文档目录 / Sub-Document Index

> 本 README 是总览；每个子文档专注于一个模块的完整代码级分析。  
> This README is the overview; each sub-doc covers one module in full code-level detail.

| 文档 / Document | 覆盖模块 / Modules Covered |
|---|---|
| **[01_knowledge_base_manager.md](./01_knowledge_base_manager.md)** | `KnowledgeBaseManager.js` — SQLite schema, VexusIndex 双索引, 文件监听批量管道, TagMemo 调用链, 热调控参数 |
| **[02_rag_diary_plugin.md](./02_rag_diary_plugin.md)** | `RAGDiaryPlugin.js` — 全占位符语法, 检索路由, Shotgun V4 路径, 聚合检索, 时间感知双路, Rerank, 三级缓存, VCP Info 广播 |
| **[03_epa_residual_result.md](./03_epa_residual_result.md)** | `EPAModule.js` + `ResidualPyramid.js` + `ResultDeduplicator.js` — 加权 PCA, 投影数学, 残差金字塔 Gram-Schmidt, SVD 潜在主题去重 |
| **[04_embedding_chunker.md](./04_embedding_chunker.md)** | `EmbeddingUtils.js` + `TextChunker.js` — 并发 Worker Pool, 批次切分, Token 计算, 重叠窗口分块 |
| **[05_context_vector_manager.md](./05_context_vector_manager.md)** | `ContextVectorManager.js` — 四级向量查找, 衰减聚合, 语义分段(Shotgun 弹药), 逻辑深度 L, 语义宽度 S |
| **[06_semantic_group_aimemo.md](./06_semantic_group_aimemo.md)** | `SemanticGroupManager.js` + `AIMemoHandler.js` — 语义组激活, 向量增强, 原子写入, AI 分批日记召回, LRU+TTL 缓存 |
| **[07_meta_thinking_time_parser.md](./07_meta_thinking_time_parser.md)** | `MetaThinkingManager.js` + `TimeExpressionParser.js` — 递归推理链, 自动主题检测, 中文时间表达式解析 |
| **[08_write_plugins.md](./08_write_plugins.md)** | `DailyNoteWrite` + `LightMemo` + `ThoughtClusterManager` + `AgentDream` — Tag 处理管道, BM25+向量混合检索, 梦境审批门, 四插件对比 |

---

## 目录 / Table of Contents

1. [设计哲学与全局架构 / Design Philosophy & Global Architecture](#1-设计哲学与全局架构--design-philosophy--global-architecture)
2. [五层记忆模型 / Five-Layer Memory Model](#2-五层记忆模型--five-layer-memory-model)
3. [记忆写入完整代码路径 / Memory Write — Complete Code Path](#3-记忆写入完整代码路径--memory-write--complete-code-path)
4. [文件索引管道 / File Indexing Pipeline](#4-文件索引管道--file-indexing-pipeline)
5. [记忆读取完整代码路径 / Memory Read — Complete Code Path](#5-记忆读取完整代码路径--memory-read--complete-code-path)
6. [TagMemo V3.7 数学管道全解析 / TagMemo V3.7 Full Mathematical Pipeline](#6-tagmemo-v37-数学管道全解析--tagmemo-v37-full-mathematical-pipeline)
7. [TagMemo V4 霰弹枪查询 / TagMemo V4 Shotgun Query](#7-tagmemo-v4-霰弹枪查询--tagmemo-v4-shotgun-query)
8. [动态参数计算 / Dynamic Parameter Calculation](#8-动态参数计算--dynamic-parameter-calculation)
9. [上下文向量管理器 / ContextVectorManager](#9-上下文向量管理器--contextvectormanager)
10. [EPA 嵌入投影分析 / EPA Embedding Projection Analysis](#10-epa-嵌入投影分析--epa-embedding-projection-analysis)
11. [残差金字塔精排 / Residual Pyramid Reranking](#11-残差金字塔精排--residual-pyramid-reranking)
12. [AIMemo AI 驱动记忆召回 / AIMemo AI-Driven Recall](#12-aimemo-ai-驱动记忆召回--aimemo-ai-driven-recall)
13. [元思考系统 / Meta-Thinking System](#13-元思考系统--meta-thinking-system)
14. [AgentDream 梦境记忆整合 / AgentDream — Dream Memory Consolidation](#14-agentdream-梦境记忆整合--agentdream--dream-memory-consolidation)
15. [ThoughtCluster 思维簇 / ThoughtCluster](#15-thoughtcluster-思维簇--thoughtcluster)
16. [LightMemo BM25+向量混合搜索 / LightMemo Hybrid Search](#16-lightmemo-bm25向量混合搜索--lightmemo-hybrid-search)
17. [Rust VexusIndex 向量引擎 / Rust VexusIndex Vector Engine](#17-rust-vexusindex-向量引擎--rust-vexusindex-vector-engine)
18. [RAG 区块注入与对话中刷新 / RAG Block Injection & In-Conversation Refresh](#18-rag-区块注入与对话中刷新--rag-block-injection--in-conversation-refresh)
19. [与主流框架横向对比 / Comparison with Mainstream Frameworks](#19-与主流框架横向对比--comparison-with-mainstream-frameworks)
20. [AI Coding Agent 调试指南 / AI Coding Agent Debug Guide](#20-ai-coding-agent-调试指南--ai-coding-agent-debug-guide)
21. [维护与扩展指南 / Maintenance & Extension Guide](#21-维护与扩展指南--maintenance--extension-guide)

---

## 1. 设计哲学与全局架构 / Design Philosophy & Global Architecture

### 1.1 核心哲学 / Core Philosophy

VCP 记忆系统建立在三个根本信念之上：

| 信念 / Belief | 传统方案 / Traditional | VCP 方案 / VCP |
|---|---|---|
| 记忆格式 / Memory format | 数据库里的结构化 JSON | 人类可读 Markdown 文件 |
| 记忆主权 / Memory ownership | 系统管理，AI 被动接受 | AI 自主写入、反思、整合 |
| 向量空间 / Vector space | 欧氏平坦空间 | 充满 Tag 引力场的曲率空间 |

> **向量空间并非平坦的，而是充满语义引力。**  
> Tags serve as gravitational anchors — queries are "pulled" toward semantic cores.

### 1.2 系统拓扑 / System Topology

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         VCP Memory Topology                              │
│                                                                          │
│  ① 写入层 / Write Layer                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐                │
│  │DailyNoteWrite│  │ThoughtCluster│  │    AgentDream   │                │
│  │(日记 Markdown)│  │(思维簇 .md)  │  │(梦境整合 .md)  │                │
│  └──────┬───────┘  └──────┬───────┘  └───────┬────────┘                │
│         └─────────────────┼──────────────────┘                          │
│                           │ 写入 dailynote/<character>/                  │
│                           ▼                                              │
│  ② 索引层 / Indexing Layer                                               │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │               KnowledgeBaseManager.js                            │  │
│  │  chokidar 文件监听 → TextChunker → Embedding API                 │  │
│  │  → SQLite (chunks/tags/files) + VexusIndex (.usearch 文件)       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ③ 检索层 / Retrieval Layer                                              │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              RAGDiaryPlugin.processMessages()                     │  │
│  │  Query Embedding → ContextVectorManager.segmentContext()          │  │
│  │  → _calculateDynamicParams() → TagMemo V3.7 → Shotgun V4         │  │
│  │  → EPA + ResidualPyramid + ResultDeduplicator + optional Rerank   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ④ 注入层 / Injection Layer                                              │
│  <!-- VCP_RAG_BLOCK_START --> ... <!-- VCP_RAG_BLOCK_END -->            │
│  system prompt 静态占位符 {{AllCharacterDiariesData}}                    │
│                                                                          │
│  ⑤ 特殊记忆层 / Special Memory Layers                                    │
│  AIMemo (AI 驱动全量召回) | MetaThinking (递归反思) | LightMemo (BM25) │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 五层记忆模型 / Five-Layer Memory Model

VCP 实现了比 MemGPT 三层模型更细致的**五层记忆体系**：

```
┌────────────────────────────────────────────────────────────────────┐
│  Layer 5: 梦境整合记忆 / Dream Consolidation Memory                │
│  AgentDream.js — 在独立梦境空间中执行记忆合并/删除/感悟            │
│  触发: 定时/手动 | 输出: dailynote/<char>/梦境感悟*.md             │
├────────────────────────────────────────────────────────────────────┤
│  Layer 4: 元思考记忆 / Meta-Thinking Memory                        │
│  MetaThinkingManager.js — 对记忆的语义抽象与主题链递归推理          │
│  触发: 对话中检测主题切换 | 输出: 日记中的推理结论                  │
├────────────────────────────────────────────────────────────────────┤
│  Layer 3: 语义组/思维簇记忆 / Semantic Group + ThoughtCluster       │
│  SemanticGroupManager + ThoughtClusterManager                      │
│  存储: VectorStore/semantic_vectors/ + dailynote/<name>簇/          │
├────────────────────────────────────────────────────────────────────┤
│  Layer 2: 日记记忆 / Diary Memory (主力长期记忆)                    │
│  dailynote/<character>/<date>-<time>.md                            │
│  检索: KnowledgeBaseManager + RAGDiaryPlugin                       │
├────────────────────────────────────────────────────────────────────┤
│  Layer 1: 工作记忆 / Working Memory (短期)                          │
│  当前 messages[] + AIMemo 触发召回 + LightMemo 快速记录             │
│  生命周期: 单次对话                                                  │
└────────────────────────────────────────────────────────────────────┘
```

| 层级 / Layer | 生命周期 | 容量 | 写入者 | 检索方式 |
|---|---|---|---|---|
| 工作记忆 | 单次对话 | context window | 对话自动 | 直接注入 |
| 日记记忆 | 永久 | 无限 | AI 主动 (DailyNoteWrite) | TagMemo 向量 + AIMemo |
| 语义组/思维簇 | 永久 | 无限 | AI 主动 (SemanticGroup/ThoughtCluster) | 向量 + 关键词 |
| 元思考 | 永久 | 有限（摘要）| 自动触发 | 主题向量匹配 |
| 梦境整合 | 永久 | 有限（精华）| 定时自动 | TagMemo 向量 |

---

## 3. 记忆写入完整代码路径 / Memory Write — Complete Code Path

### 3.1 路径 A：AI 主动写日记（推荐长期记忆）

```
AI 生成工具调用
<<<[TOOL_REQUEST]>>>
tool_name:「始」DailyNoteWrite「末」,
maidName:「始」小可「末」,
dateString:「始」2026-02-23「末」,
contentText:「始」今天学了量子力学...
Tag: 量子力学, 物理, 学习「末」
<<<[END_TOOL_REQUEST]>>>

    │  Plugin.js 解析 VCP 工具调用协议
    │  → spawn child process: node daily-note-write.js
    │
    ▼
Plugin/DailyNoteWrite/daily-note-write.js
  writeDiary(maidName, dateString, contentText)         [line 323]
    ├── processTagsInContent(contentText)               [line 330]
    │   ├── 验证 content 末行是否有 "Tag: ..." 格式     [line 75-77]
    │   ├── 如有 → fixTagFormat() 规范化 Tag 行         [line 94-130]
    │   └── 如无 → generateTagsWithAI()                 [line 172]
    │       → POST Embedding/Completion API 生成 Tag
    │
    ├── sanitizePathComponent(folderName)               [line 351]
    ├── 路径计算: dailynote/<sanitizedFolder>/<date>-<time>.md
    ├── fs.mkdir(dirPath, { recursive: true })          [line 372]
    └── fs.writeFile(filePath, fileContent)             [line 374]
        文件格式: "[2026-02-23] - 小可\n<content>\nTag: 量子力学, 物理"
```

**📁 输出文件位置 / Output file location**:
```
dailynote/
└── 小可/
    └── 2026-02-23-14_30_00.md    ← 文件名带时间戳防冲突
```

### 3.2 路径 B：AI 写思维簇（按主题聚类记忆）

```
AI 调用 ThoughtClusterManager
→ Plugin/ThoughtClusterManager/ThoughtClusterManager.js

createClusterFile({ clusterName: "量子力学簇", content })  [line 44]
  ├── 验证 clusterName 必须以 '簇' 结尾                  [line 50]
  ├── fs.mkdir(dailynote/<clusterName>/, { recursive })   [line 56]
  └── fs.writeFile(dailynote/<clusterName>/<timestamp>.md) [line 61]

editClusterFile({ clusterName, targetText, replacementText })
  └── 读取 → 替换 → 写回 (原子操作)
```

### 3.3 路径 C：AgentDream 梦境感悟写入

```
AgentDream 定时触发
  → 从 dailynote/ 随机采样"记忆种子"
  → KnowledgeBaseManager.search() 找到相关记忆
  → 调用 AI 生成梦境叙事 + 操作指令 (Merge/Delete/Insight)
  → 生成 dream_logs/<timestamp>.json (待管理员审批)
  → 审批通过后 → DailyNoteWrite 写入"梦境感悟"日记
```

### 3.4 路径 D：文件变更后自动触发索引

写入任何 `.md`/`.txt` 文件后，chokidar 立即触发 → 见[§4 文件索引管道](#4-文件索引管道--file-indexing-pipeline)。

---

## 4. 文件索引管道 / File Indexing Pipeline

这是写入与向量检索之间的关键链路，理解它对调试至关重要。

```
新文件写入 dailynote/<char>/<date>.md
              │
              ▼
KnowledgeBaseManager._startWatcher()               [line 950]
  chokidar.watch(rootPath, { ignoreInitial: false })
  事件: 'add' | 'change'
              │
              ▼ handleFile(filePath)               [line 952]
  ├── 过滤 ignoreFolders (e.g. VCP论坛)
  ├── 过滤 ignorePrefixes (e.g. 已整理)
  ├── 过滤 ignoreSuffixes (e.g. 夜伽)
  ├── 过滤非 .md/.txt 文件
  └── pendingFiles.add(filePath)
              │
              ▼ _scheduleBatch()                   [line 976]
  batchTimer = setTimeout(_flushBatch, 2000ms)    ← 防抖2秒
              │
              ▼ _flushBatch()                      [line 981]
  ┌─────────────────────────────────────────────┐
  │ 步骤1: 读取文件 + 变更检测                   │
  │   stat() 对比 mtime/size                    │
  │   MD5 checksum 对比内容                     │
  │   相同则跳过，不同则继续                    │
  ├─────────────────────────────────────────────┤
  │ 步骤2: 文本分块                             │
  │   TextChunker.chunkText(content)           │
  │   → 按语义边界切割（段落/代码块/标题）     │
  ├─────────────────────────────────────────────┤
  │ 步骤3: 提取 Tag                             │
  │   _extractTags(content)                    │
  │   → 正则匹配 "Tag: xxx, yyy" 行            │
  ├─────────────────────────────────────────────┤
  │ 步骤4: 批量 Embedding API 调用              │
  │   getEmbeddingsBatch(chunkTexts)           [line 1059]
  │   → 并行调用 Embedding 模型               │
  │   (默认: google/gemini-embedding-001)      │
  │   (维度: VECTORDB_DIMENSION, 默认 3072)   │
  ├─────────────────────────────────────────────┤
  │ 步骤5: SQLite 事务写入                      │[line 1072]
  │   INSERT/UPDATE files                      │
  │   DELETE old chunks                        │
  │   INSERT new chunks (content + vector BLOB)│
  │   INSERT/UPDATE tags                        │
  │   INSERT file_tags 关联                    │
  ├─────────────────────────────────────────────┤
  │ 步骤6: VexusIndex 更新                     │
  │   先 remove() 旧 chunk IDs               [line 1154]
  │   再 add(chunkId, vectorBuffer)           [line 1184]
  │   tagIndex.add(tagId, tagVectorBuffer)    [line 1163]
  └─────────────────────────────────────────────┘
              │
              ▼ _scheduleIndexSave()               [line 1175]
  延迟 120s 保存日记索引 .usearch 文件
  延迟 300s 保存 Tag 索引 .usearch 文件
```

**关键配置 / Key Config**:
```bash
KNOWLEDGEBASE_BATCH_WINDOW_MS=2000    # 防抖时间窗口
KNOWLEDGEBASE_MAX_BATCH_SIZE=50       # 单批最大文件数
KNOWLEDGEBASE_INDEX_SAVE_DELAY=120000 # 索引保存延迟(2分钟)
KNOWLEDGEBASE_FULL_SCAN_ON_STARTUP=true  # 启动时全量扫描
VECTORDB_DIMENSION=3072               # ⚠️ 必须与 Embedding 模型一致
```

---

## 5. 记忆读取完整代码路径 / Memory Read — Complete Code Path

每次收到 `POST /v1/chat/completions` 请求，记忆读取按以下顺序触发：

```
POST /v1/chat/completions
{ messages: [...], model: "gpt-4o", ... }
         │
         ▼
[Plugin.js — messagePreprocessor 阶段]
  RAGDiaryPlugin.processMessages(messages, pluginConfig)  [line 857]
         │
         ▼
  ① 检测 AIMemo 许可证                                    [line 864-873]
    if (system prompt 含 "[[AIMemo=True]]")
      isAIMemoLicensed = true

  ② 提取查询文本 (最近 3 轮对话)                          [line 940-960]
    userContent = messages.filter(role=user).last
    aiContent   = messages.filter(role=assistant).last
    combinedQuery = userContent + aiContent.slice(0, 500)

  ③ 获取查询向量                                          [line 963]
    queryVector = await getSingleEmbedding(combinedQuery)

  ④ TagMemo V4 上下文分段                                 [line 974-978]
    historySegments = contextVectorManager.segmentContext(messages)
    // 从历史消息提取衰减加权的上下文向量段

  ⑤ 动态参数计算                                          [line 972]
    { k, tagWeight, tagTruncationRatio, metrics } =
      await _calculateDynamicParams(queryVector, userContent, aiContent)

  ⑥ 时间意图解析                                          [line 967-970]
    timeRanges = timeParser.extractTimeRanges(userContent)

  ⑦ 对每个 [[日记本]] 占位符执行检索                      [line 857-1036]
    _processSingleSystemMessage(...)
         │
         ▼
[_processSingleSystemMessage — 核心检索]               [line 1037]
  解析日记本语法:
    [[小可]]              → 单日记本检索
    [[小可|初音::Meta]]   → 聚合检索 (V4 多库合并)
    [[小可::AIMemo]]      → AI 驱动全量召回
    [[VCP元思考:主题]]    → 元思考递归推理链
         │
         ▼
  向量检索路径 (标准 TagMemo):
    vectorDBManager.search(dbName, queryVector, k, tagWeight, coreTags)
         │ ↓ 内部调用
         ▼
    KnowledgeBaseManager._searchSpecificIndex()            [line 315]
      ├── [1] tagBoost > 0 → _applyTagBoostV3()          [line 333]
      │    (TagMemo V3.7 全管道 — 见 §6)
      ├── [2] VexusIndex.search(searchBuffer, k)          [line 355]
      │    (Rust N-API KNN 搜索)
      └── [3] hydrate: SQL JOIN chunks + files            [line 363]
               → { text, score, sourceFile, matchedTags, ... }
         │
         ▼
  TagMemo V4 Shotgun 多向量搜索                            [line 2018]
    searchVectors = [currentQuery, ...historySegments.slice(-3)]
    results = await Promise.all(searchVectors.map(v => search(v, k/2)))
    flattenedResults = flatten + _filterContextDuplicates()
         │
         ▼
  智能去重                                                 [line 2057]
    uniqueResults = await vectorDBManager.deduplicateResults(
      flattenedResults, finalQueryVector
    )  // ResultDeduplicator (SVD + 残差)
         │
         ▼
  (可选) Rerank                                           [line 2059]
    finalResults = await _rerankDocuments(userContent, uniqueResults, k)
    // 外部 Rerank API (如 Cohere, Jina)
         │
         ▼
  格式化并注入 system prompt                              [line 2080-2083]
    retrievedContent = formatStandardResults(results, displayName)
    system_message.content 中替换 [[日记本名]] 占位符
    为 <!-- VCP_RAG_BLOCK_START ... -->内容<!-- VCP_RAG_BLOCK_END -->
```

---

## 6. TagMemo V3.7 数学管道全解析 / TagMemo V3.7 Full Mathematical Pipeline

**📁 位置 / Location**: `KnowledgeBaseManager.js:446` — `_applyTagBoostV3()`

TagMemo V3.7 是整个系统最核心的算法，共分 7 个阶段：

### 阶段 1: EPA 分析 — "你在哪个语义世界"

```javascript
// line 453-455
const epaResult = this.epa.project(originalFloat32);
const resonance = this.epa.detectCrossDomainResonance(originalFloat32);
const queryWorld = epaResult.dominantAxes[0]?.label || 'Unknown';
// 输出: logicDepth (0~1), entropy (0~1), resonance (0~∞)
// queryWorld: 如 "物理", "技术", "政治" 等语义世界标签
```

### 阶段 2: 残差金字塔分析 — "语义能量结构"

```javascript
// line 458-459
const pyramid = this.residualPyramid.analyze(originalFloat32);
// pyramid.features: { tagMemoActivation, coverage, novelty, depth }
// tagMemoActivation: 0~1, 表征 "有多少语义可以被 Tag 解释"
```

### 阶段 3: 动态 Boost 因子计算

```javascript
// line 462-473
// 核心公式:
const dynamicBoostFactor =
  (logicDepth * (1 + log(1 + resonance)) / (1 + entropy * 0.5))
  * activationMultiplier;

// activationMultiplier 从 rag_params.json 热调控:
//   actRange = [0.5, 1.5]  (默认)
//   activationMultiplier = 0.5 + tagMemoActivation * 1.0

// effectiveTagBoost:
const effectiveTagBoost = baseTagBoost * clamp(dynamicBoostFactor, 0.3, 2.0)
```

**直觉解释 / Intuitive explanation**:
- `logicDepth` 高 → 意图明确 → 放大 boost → Tag 引力更强
- `entropy` 高 → 信息散乱 → 减弱 boost → 防止噪音放大
- `resonance` 高 → 跨域共振 → 放大 boost → 激活跨领域联想

### 阶段 4: 残差金字塔 Tag 收集 + 世界观门控 + 语言补偿

```
pyramid.levels (最多3层 Gram-Schmidt 正交分解)
  ↓
每个 Tag 的调整权重 = contribution × layerDecay × langPenalty × coreBoost

layerDecay  = 0.7^level      (深层 Tag 衰减)
langPenalty = 1.0             (中文世界中英文技术词: 0.05~0.1 惩罚)
coreBoost   = dynamicCoreBoostFactor  (核心 Tag: 1.20~1.40 奖励)
```

### 阶段 4.5: 逻辑拉回 (Logic Pull-back)

```javascript
// line 550-576
// 利用 tagCooccurrenceMatrix 拉回与高权重 Tag 强相关的逻辑词
// 前 5 个高权重 Tag → 各自关联的前 4 个共现 Tag
// 关联词权重 = 父 Tag 权重 × 0.5
```

### 阶段 5: 语义去重 (Semantic Deduplication)

```javascript
// line 620-659
// 按权重降序排列所有 Tag
// 对每个 Tag，计算与已入选 Tag 的余弦相似度
// 如果 similarity > deduplicationThreshold (默认 0.88):
//   冗余: 将该 Tag 20% 权重转给代表性 Tag，跳过自身
// 目标: 消除 "委内瑞拉局势" vs "委内瑞拉危机" 这类冗余
```

### 阶段 6: 构建上下文向量 (Context Vector Construction)

```javascript
// line 661-685
// 对所有去重后的 Tag 向量加权平均:
contextVec = Σ(tagVec[i] × adjustedWeight[i]) / totalWeight
// 然后 L2 归一化
```

### 阶段 7: 最终线性融合 (Final Linear Fusion)

```
fusedVector = (1 - effectiveTagBoost) × originalQueryVector
            + effectiveTagBoost × contextVector
// 再次 L2 归一化

// 直觉: 保留原始查询方向 + 被 Tag 引力场"拉扯"的分量
// effectiveTagBoost 通常在 0.1 ~ 0.6 之间
```

**完整数学公式摘要 / Mathematical Summary**:

```
fused_q = normalize(
  (1 - β_eff) · q_orig
  + β_eff · normalize(Σᵢ wᵢ · t_vec_i)
)

where:
  β_eff = β_base · clamp(L·(1+ln(1+R))/(1+H·0.5) · A_mul, 0.3, 2.0)
  wᵢ = contribution_i · 0.7^level_i · lang_penalty_i · core_boost_i
  A_mul = 0.5 + tagMemoActivation · 1.0
  L = logicDepth, R = resonance, H = entropy
```

---

## 7. TagMemo V4 霰弹枪查询 / TagMemo V4 Shotgun Query

**📁 位置 / Location**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:2018`

TagMemo V4 引入"霰弹枪"多向量并行搜索，解决单一向量查询无法覆盖对话全上下文的问题。

```
当前查询向量 q_current (from 最新 user message)
    +
历史上下文向量段 [q_seg1, q_seg2, q_seg3] (from ContextVectorManager)
    │
    ▼
searchVectors = [
  { vector: q_current,    type: 'current',   k: dynamicK },
  { vector: q_seg1,       type: 'history_0', k: dynamicK/2 },
  { vector: q_seg2,       type: 'history_1', k: dynamicK/2 },
  { vector: q_seg3,       type: 'history_2', k: dynamicK/2 },
]
    │
    ▼
Promise.all(searchVectors.map(qv => vectorDBManager.search(dbName, qv.vector, qv.k)))
    │
    ▼
flattenedResults = all results flattened
    │
    ▼
_filterContextDuplicates(flattenedResults, contextDiaryPrefixes)
    │ 过滤与当前工具调用上下文重复的条目
    ▼
vectorDBManager.deduplicateResults(flattenedResults, finalQueryVector)
    │ SVD + 残差金字塔去重 (ResultDeduplicator)
    ▼
(可选) Rerank → 截断到 finalK 个结果
```

**V4 vs V3 对比 / V4 vs V3 Comparison**:

| 特性 | V3 | V4 |
|---|---|---|
| 查询向量数 | 1 | 1 + min(historySegments, 3) |
| 并发搜索 | 否 | ✅ Promise.all |
| 历史上下文 | 无 | ✅ 衰减加权历史段 |
| 去重 | 基础 | ✅ SVD + 残差金字塔 |

---

## 8. 动态参数计算 / Dynamic Parameter Calculation

**📁 位置 / Location**: `Plugin/RAGDiaryPlugin/RAGDiaryPlugin.js:430`

每次检索前都会动态计算 K 值、Tag 权重和截断比例：

```javascript
async _calculateDynamicParams(queryVector, userText, aiText) {
    // 1. 基础 K 值 (基于文本长度)
    //    userLen > 100: k_base=6, > 30: k_base=4, 否则: k_base=3

    // 2. EPA 指标
    const { logicDepth: L, resonance: R } = await vectorDBManager.getEPAAnalysis(queryVector)

    // 3. 语义宽度 (来自 ContextVectorManager)
    const S = contextVectorManager.computeSemanticWidth(queryVector)

    // 4. 动态 Beta (TagWeight):
    // β = sigmoid(L · ln(2 + R) - S · noise_penalty)
    // noise_penalty = 0.05 (from rag_params.json)
    // 映射到 [0.05, 0.45] 范围

    // 5. 动态 K:
    // finalK = clamp(k_base + round(L*3 + ln1p(R)*2), 3, 10)

    // 6. 动态 Tag 截断比例:
    // tagTruncationRatio = clamp(0.6 + L*0.3 - S*0.2 + min(R,1)*0.1, 0.5, 0.9)
}
```

**参数语义一览 / Parameter Semantics**:

| 参数 | 值域 | 含义 |
|---|---|---|
| `L` (logicDepth) | 0~1 | 查询意图的聚焦程度；高=明确意图 |
| `R` (resonance) | 0~∞ | 跨语义域共振强度；高=涉及多领域 |
| `S` (semanticWidth) | 0~∞ | 历史对话的语义分散度；高=话题发散 |
| `β` (tagWeight) | 0.05~0.45 | Tag 引力场强度；高=更依赖 Tag |
| `K` | 3~10 | 检索返回的 chunk 数量 |
| `truncationRatio` | 0.5~0.9 | 保留 Tag 列表的比例 |

---

## 9. 上下文向量管理器 / ContextVectorManager

**📁 位置 / Location**: `Plugin/RAGDiaryPlugin/ContextVectorManager.js`

> ⚠️ **未被其他文档记录 / Not documented elsewhere** — 这是 TagMemo V4 的关键新组件。

ContextVectorManager 维护当前会话的向量化历史，用于：
1. `segmentContext()` — 提供多个历史上下文向量段 (Shotgun V4 的弹药)
2. `computeSemanticWidth()` — 衡量历史对话的语义发散程度

### 9.1 向量映射机制

```javascript
// 构造函数初始化
this.vectorMap = new Map()  // normalizedHash → { vector, role, timestamp }
this.historyAssistantVectors = []   // 按时序的 assistant 向量
this.historyUserVectors = []        // 按时序的 user 向量
this.decayRate = 0.75               // 衰减率 (越近的消息权重越高)
this.maxContextWindow = 10          // 最多看最近 10 楼
```

### 9.2 模糊匹配复用向量

```javascript
// Dice 系数文本相似度 ≥ 0.85 时复用已有向量 (避免重复 API 调用)
_calculateSimilarity(str1, str2) {
    // 计算 bigram 交集 / (|b1| + |b2|)
}
```

### 9.3 衰减加权历史段

```
messages 按时序排列: [...older, ..., recent_user, recent_ai]
                                                  ↑
                      ContextVectorManager 跳过最后1对 (当前轮)
                      对其余消息计算向量，按距离远近应用衰减:
                      weight_i = decayRate^(distance_from_tail)
                               = 0.75^1, 0.75^2, 0.75^3, ...
```

### 9.4 语义宽度 (Semantic Width)

```javascript
// computeSemanticWidth(queryVector):
//   与历史向量池中所有向量计算余弦相似度
//   S = variance(similarities)
//   S 高 → 历史话题发散 → 应减小 TagWeight (减少噪音)
//   S 低 → 历史话题聚焦 → 可增大 TagWeight (利用主题连贯)
```

---

## 10. EPA 嵌入投影分析 / EPA Embedding Projection Analysis

**📁 位置 / Location**: `EPAModule.js`

EPA (Embedding Projection Analysis) 是 VCP 独创的语义空间分析器，基于**加权 PCA** 构建正交语义基底：

### 10.1 初始化：构建正交基

```
1. 从 SQLite tags 表加载所有带向量的 Tag (~N 个)
2. 鲁棒 K-Means 聚类 (k=32): 提取代表性质心
3. 加权中心化 PCA (Weighted PCA):
   - 对质心矩阵去中心化 (减均值)
   - SVD 分解: M = U·S·Vᵀ
   - 选择解释方差 ≥ 1% 的主成分 K
4. 结果: orthoBasis[K], basisMean, basisEnergies, basisLabels
5. 持久化到缓存文件 (epa_basis_cache.json)
```

### 10.2 投影：分析查询向量

```javascript
project(vector) {
    // 1. 去中心化: centeredVec = vec - basisMean
    // 2. 投影到 K 个主成分轴: projections[k] = dot(centeredVec, orthoBasis[k])
    // 3. 计算概率分布 (softmax of |projections|)
    // 4. Shannon 熵: entropy = -Σ p_i · log(p_i)
    // 5. 逻辑深度: logicDepth = 1 - (entropy / ln(K))
    //    [0=完全分散, 1=高度集中在一个轴]
    // 6. 主导轴标签 (dominantAxes[0].label = 语义世界名称)
}
```

### 10.3 跨域共振检测

```javascript
detectCrossDomainResonance(vector) {
    // 找到投影中能量 ≥ 0.1 的主成分 (活跃轴)
    // resonance = count(activeAxes) × averageProjectionMagnitude
    // 高共振 → 该查询涉及多个语义领域 → 需要更广泛的检索
}
```

**Rust 加速 / Rust Acceleration**: 如果 VexusIndex 支持 `.project()` 方法，EPA 会优先调用 Rust 实现进行矩阵投影，否则回退到 JS 实现。

---

## 11. 残差金字塔精排 / Residual Pyramid Reranking

**📁 位置 / Location**: `ResidualPyramid.js`

残差金字塔通过 **Gram-Schmidt 正交投影** 分解查询向量的语义能量谱：

### 11.1 算法流程

```
输入: queryVector q
初始残差: r₀ = q

循环 (最多 maxLevels=3 次):
  Level L:
    1. VexusIndex.search(r_L, topK=10) → 找到最相关的 Tag 向量 {t₁, t₂, ..., t_K}
    2. Gram-Schmidt 正交化 {t₁...t_K}:
       → 获得正交基 {e₁, e₂, ..., e_K'}
    3. 计算投影: p_L = Σ <r_L, e_k> · e_k
    4. 计算残差: r_{L+1} = r_L - p_L
    5. 本层解释能量: energyExplainedByLevel = (||r_L||² - ||r_{L+1}||²) / ||q||²
    6. 记录 pyramid.levels[L] = { tags, energyExplained, ... }
    
    终止条件: ||r_{L+1}||² / ||q||² < minEnergyRatio (默认 0.1)
              即 Tag 已解释了 90% 以上的语义能量
```

### 11.2 输出特征

```javascript
pyramid.features = {
    tagMemoActivation: totalExplainedEnergy,   // Tag 可解释的能量占比
    coverage: 1 - residualEnergy / totalEnergy, // 覆盖率
    novelty: residualEnergy / totalEnergy,       // 新颖度 (未被 Tag 覆盖的部分)
    depth: pyramid.levels.length                 // 分解深度
}
```

### 11.3 与 LlamaIndex MMR 对比

| 特性 | ResidualPyramid | MMR (LlamaIndex/Haystack) |
|---|---|---|
| 去重原理 | 正交子空间投影 | 相关性-冗余权衡 |
| 数学基础 | Gram-Schmidt | 贪心最大边际 |
| 物理直觉 | 能量残差分层 | 相似度差异最大化 |
| 维度感知 | ✅ (利用全维度向量) | 🔍 (依赖余弦相似度) |

---

## 12. AIMemo AI 驱动记忆召回 / AIMemo AI-Driven Recall

**📁 位置 / Location**: `Plugin/RAGDiaryPlugin/AIMemoHandler.js`

AIMemo 不同于 TagMemo 向量检索 — 它让 AI 模型**直接阅读日记全文**并执行语义召回：

### 12.1 激活条件

```
system prompt 必须包含: [[AIMemo=True]]   (全局许可证)
日记本语法必须包含: [[日记本名::AIMemo]]  (逐日记本激活)
```

### 12.2 处理流程

```
processAIMemoAggregated(dbNames, userContent, aiContent)    [line 63]

  ① 缓存检查 (LRU+TTL, 30分钟)
    cacheKey = MD5(dbNames + userContent + aiContent)
    if (cache hit) → 直接返回

  ② 收集日记文件
    _getDiaryFiles(dbName) → 读取近 N 天的 .md 文件列表

  ③ 按 token 分批 (maxTokensPerBatch = 60000)
    每批包含若干完整日记文件

  ④ 调用 AIMemo 专用模型
    POST AIMemoUrl {
      model: AIMemoModel,
      messages: [
        { role: "system", content: promptTemplate },
        { role: "user", content: "日记内容:\n" + batchContent 
                                + "\n用户问题: " + userContent }
      ]
    }

  ⑤ 解析并合并所有批次的召回结果

  ⑥ 写入缓存并返回格式化的记忆文本
```

### 12.3 适用场景 vs TagMemo

| 场景 | TagMemo (向量) | AIMemo (AI召回) |
|---|---|---|
| 精确语义匹配 | ✅ 优秀 | ✅ 优秀 |
| 复杂推理/综合 | ❌ 不支持 | ✅ AI 可综合推理 |
| 速度 | ✅ 毫秒级 | ❌ 数秒~数十秒 |
| 成本 | ✅ 低 (Embedding) | ❌ 高 (LLM 调用) |
| 日记量大时 | ✅ 不变慢 | ❌ token 限制 |

---

## 13. 元思考系统 / Meta-Thinking System

**📁 位置 / Location**: `Plugin/RAGDiaryPlugin/MetaThinkingManager.js`

元思考是 VCP 独有的**递归推理链**机制，让 AI 能够对自己的记忆进行高阶抽象：

### 13.1 工作原理

```
meta_thinking_chains.json 定义推理链:
{
  "chains": {
    "技术思考": {
      "prompt": "从技术角度分析这些记忆...",
      "steps": ["分析", "总结", "洞察"]
    },
    "情感反思": { ... },
    "default": { ... }
  }
}

触发:
  system prompt 含 [[VCP元思考:技术思考]]
  → MetaThinkingManager 检测主题向量与 queryVector 的相似度
  → 选择最匹配的推理链
  → 执行多步递归推理 (每步结果作为下一步输入)
  → 输出高阶反思结论注入 system prompt
```

### 13.2 主题向量缓存

```javascript
// 每个推理链主题 (如 "技术思考") 都有对应的 Embedding 向量
// 存储在 meta_chain_vector_cache.json
// 配置文件变更时 (MD5 hash 检测) 自动重建缓存
```

---

## 14. AgentDream 梦境记忆整合 / AgentDream — Dream Memory Consolidation

**📁 位置 / Location**: `Plugin/AgentDream/AgentDream.js`  
**📁 详细文档 / Full doc**: `AgentDream.md`

AgentDream 是 VCP 生态中最独特的记忆组件——通过**模拟"做梦"**来整合和净化长期记忆：

### 14.1 梦境触发与流程

```
定时触发 / 手动触发
       │
       ▼
① 记忆采集 (Memory Seeding)
  从 dailynote/ 随机抽取一条"记忆种子"
  → KnowledgeBaseManager.search(seedVector, k=5)
  → 找到语义相关的记忆碎片集合

② 梦境叙事生成 (Dream Narrative)
  渲染 dreampost.txt 模板:
    {{memorySeeds}} → 种子记忆片段
    {{relatedMemories}} → 语义联想片段
  → 调用 AI 生成意识流梦境叙事

③ 梦境操作提取 (Dream Operations)
  AI 在叙事中嵌入三类操作 (VCP 串语法):
    ① DiaryMerge   → 合并相似日记 (减少冗余)
    ② DiaryDelete  → 删除过时/错误记忆
    ③ DreamInsight → 创造新的感悟日记

④ 操作审批 (Approval Gate)
  将操作生成 dream_logs/<timestamp>.json
  → 管理员在 AdminPanel 审批页面确认
  → 通过后执行实际文件操作

⑤ 梦境日记写入 (Dream Diary)
  DailyNoteWrite 写入 "梦境感悟" 日记
  → 触发索引管道 (见 §4)
```

### 14.2 设计哲学

AgentDream 实现了类似**睡眠记忆巩固 (Sleep Memory Consolidation)** 的机制：
- 重播和联想已有记忆 → 增强语义连接
- 合并相似记忆 → 减少向量索引冗余
- 生成高阶感悟 → 丰富元思考材料
- 人工审批门 → 防止 AI 自我篡改记忆

---

## 15. ThoughtCluster 思维簇 / ThoughtCluster

**📁 位置 / Location**: `Plugin/ThoughtClusterManager/ThoughtClusterManager.js`

ThoughtCluster 是比 DailyNoteWrite 更**结构化**的长期记忆写入方式：

```
命名约定: 目录名必须以 "簇" 结尾 (如 "量子力学簇", "编程经验簇")
存储结构:
  dailynote/
  └── 量子力学簇/
      ├── 2026-02-23T14-30-00-000Z.md   ← CreateClusterFile
      └── 2026-02-20T09-15-00-000Z.md

API:
  CreateClusterFile(clusterName, content) → 写入新条目
  EditClusterFile(clusterName, targetText, replacementText) → 修改已有条目
```

**与日记的区别 / vs Diary**:
- 日记按时间归档，思维簇按**主题**归档
- 日记适合记录事件，思维簇适合记录经验/规律/知识
- 两者都会进入统一的 KnowledgeBaseManager 索引

---

## 16. LightMemo BM25+向量混合搜索 / LightMemo Hybrid Search

**📁 位置 / Location**: `Plugin/LightMemo/LightMemo.js`

LightMemo 使用 **BM25 + 向量相似度 + 可选 Rerank** 的三阶段混合检索：

### 16.1 三阶段检索流程

```
① BM25 初筛 (关键词检索)                              [line 156-194]
  Jieba 分词 → BM25 打分 → 取 k×5 候选
  语义组扩词: _expandQueryTokens() (利用 SemanticGroupManager)

② 向量相似度精排                                      [line 576]
  _scoreByVectorSimilarity(candidates, queryVector)
  → 对 BM25 筛出的候选做向量相似度计算
  → BM25 分 × λ + 向量分 × (1-λ) 混合打分

③ (可选) Rerank 最终排序                              [line 348]
  如果 rerank=true → 外部 Rerank API 二次排序
```

### 16.2 适用场景

LightMemo 适用于：
- **关键词明确**的精确查找（如"查找关于量子纠缠的记录"）
- **不需要深度语义理解**的快速检索
- 与 RAGDiaryPlugin 互补：LightMemo 关键词精确，RAG 语义模糊

---

## 17. Rust VexusIndex 向量引擎 / Rust VexusIndex Vector Engine

**📁 位置 / Location**: `rust-vexus-lite/`  
**📁 文档 / Doc**: `docs/RUST_VECTOR_ENGINE.md`

VexusIndex 是基于 USearch 的 Rust N-API 向量索引，提供：

```javascript
// 创建索引
const idx = new VexusIndex(dimension, capacity)  // e.g., (3072, 50000)

// 加载已有索引
const idx = VexusIndex.load(filePath, null, dimension, capacity)

// 添加向量
idx.add(id, vectorBuffer)           // id: integer, vectorBuffer: Buffer

// 删除向量
idx.remove(id)

// KNN 搜索
const results = idx.search(queryBuffer, k)
// 返回: [{ id: integer, score: float (余弦相似度) }]

// 保存到磁盘
idx.save(filePath)

// Rust 加速 EPA 投影 (VexusIndex v2+)
idx.project(vecBuf, basisBuf, meanBuf, K) → { projections, probabilities, entropy, totalEnergy }

// 统计
idx.stats() → { totalVectors: number }
```

**存储文件 / Storage files**:
```
VectorStore/
├── knowledge_base.sqlite          ← 文本+向量 BLOB 主数据库
├── index_global_tags.usearch      ← 全局 Tag 向量索引
└── diary_<md5(charName)>.usearch  ← 每个日记本独立索引
```

**性能特性 / Performance**:
- USearch 采用 HNSW 算法，搜索复杂度 O(log N)
- 单机支持数百万向量
- 向量 Buffer 使用 `Buffer.from(float32Array.buffer, byteOffset, byteLength)` 传递

---

## 18. RAG 区块注入与对话中刷新 / RAG Block Injection & In-Conversation Refresh

### 18.1 注入格式

```html
<!-- VCP_RAG_BLOCK_START {"diaryName":"小可","topK":5,"query":"量子力学"} -->
[相关记忆 / Relevant Memories]

**来自 2026-02-20-14_30_00.md**
今天在图书馆看了量子力学导论...
Tag: 量子力学, 物理, 学习

**来自 2026-02-15-09_00_00.md**
关于量子纠缠的一些思考...
<!-- VCP_RAG_BLOCK_END -->
```

### 18.2 对话中刷新机制

```javascript
// chatCompletionHandler.js:195-286
// _refreshRagBlocksIfNeeded(messages, newContext)

// 触发条件: 每次 VCP 工具调用循环结束后
// 检测 messages 中已有的 VCP_RAG_BLOCK
// 用最新 user 消息 (而非对话开始时的查询) 重新检索
// 替换旧 RAG 区块 → 保持记忆上下文始终与最新话题相关

const ragBlockRegex = /<!-- VCP_RAG_BLOCK_START ([\s\S]*?) -->([\s\S]*?)<!-- VCP_RAG_BLOCK_END -->/g;
```

---

## 19. 与主流框架横向对比 / Comparison with Mainstream Frameworks

### 19.1 记忆管理框架对比 / Memory Framework Comparison

| 特性 / Feature | **VCP** | **LangChain Memory** | **LangGraph** | **MemGPT/Letta** | **Zep** | **LangMem** |
|---|---|---|---|---|---|---|
| 记忆格式 | Markdown 文件 | In-memory / Redis | Graph State | 结构化 JSON | PostgreSQL | Vector DB |
| 人类可读 | ✅ 完全透明 | 🔍 部分 | ❌ 图状态 | ❌ JSON | ❌ DB | ❌ |
| AI 自主写入 | ✅ DailyNoteWrite | ❌ 手动 | 🔍 节点操作 | ✅ | ✅ | ✅ |
| 向量检索 | ✅ Rust N-API | ✅ | ✅ | ✅ | ✅ | ✅ |
| Tag 语义引力场 | ✅ **独有** | ❌ | ❌ | ❌ | ❌ | ❌ |
| 正交基 EPA | ✅ **独有** | ❌ | ❌ | ❌ | ❌ | ❌ |
| 残差金字塔 | ✅ **独有** | ❌ | ❌ | ❌ | ❌ | ❌ |
| 梦境记忆整合 | ✅ **独有** | ❌ | ❌ | ❌ | ❌ | ❌ |
| 对话中 RAG 刷新 | ✅ **独有** | ❌ | ❌ | ❌ | ❌ | ❌ |
| 多 Agent 隔离 | ✅ 每角色独立 | ❌ | 🔍 | ✅ | ✅ | ❌ |
| 记忆反思 | ✅ MetaThinking | ❌ | 🔍 | ✅ | ❌ | ✅ |
| 离线可用 | ✅ 完全本地 | 🔍 | 🔍 | ❌ 云 | ❌ 云/自托管 | 🔍 |
| 热调控 | ✅ rag_params.json | ❌ | ❌ | ❌ | ❌ | ❌ |
| 记忆层数 | **5层** | 2层 | 2层 | 3层 | 2层 | 3层 |
| 编程语言 | Node.js+Rust | Python | Python | Python | Python/Go | Python |

### 19.2 LangChain Memory 详细对比

**LangChain Memory** 提供多种记忆类型：

```python
# LangChain 标准记忆
from langchain.memory import (
    ConversationBufferMemory,       # 完整对话历史
    ConversationSummaryMemory,      # 摘要记忆
    ConversationVectorStoreMemory,  # 向量检索记忆
    ConversationEntityMemory        # 实体记忆
)

# 使用示例
memory = ConversationVectorStoreMemory(
    vectorstore=Chroma(...),
    memory_key="history"
)
chain = ConversationChain(llm=llm, memory=memory)
```

**与 VCP 的关键差异**:

```
LangChain Memory:
  + Python 生态最完整，插件丰富
  + ConversationEntityMemory 自动实体抽取
  - 记忆存储格式不透明，调试困难
  - 无 Tag 语义引力场，纯向量相似度
  - 无对话中 RAG 刷新机制
  - 记忆写入需手动配置，AI 不能自主写入

VCP:
  + 记忆完全人类可读 (Markdown)
  + AI 自主控制记忆写入时机和内容
  + TagMemo 语义引力场 + EPA + 残差金字塔 = 更精准语义对齐
  + 梦境整合实现记忆自我优化 (langchain 无对应概念)
  - Node.js 生态，Python 数据工具需通过插件调用
```

### 19.3 LangGraph 详细对比

**LangGraph** 是 LangChain 的图执行引擎，适合复杂多步骤工作流：

```python
# LangGraph 状态图定义
from langgraph.graph import StateGraph, END
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    memory: dict        # 图状态中的记忆存储

graph = StateGraph(AgentState)
graph.add_node("retrieve_memory", retrieve_from_vectorstore)
graph.add_node("generate", llm_call)
graph.add_edge("retrieve_memory", "generate")
```

**与 VCP 的关键差异**:

```
LangGraph:
  + 最适合复杂多步骤工作流
  + 图结构天然支持分支/循环/条件执行
  + 状态管理是核心能力
  - 图状态记忆不透明，难以人工检查
  - 长期记忆需要外接向量数据库
  - 无内置 Agent 记忆主权概念

VCP:
  + 工具调用和记忆系统无缝集成
  + AI 对记忆有完整自主权
  + 无需额外编排框架，记忆系统内置于中间层
  - 工作流编排能力不如 LangGraph 灵活
```

### 19.4 LangMem 详细对比

**LangMem** (2024, Anthropic 生态) 是新兴的 AI 长期记忆框架：

```python
# LangMem 使用
from langmem import create_memory_manager

manager = create_memory_manager(
    "claude-3-5-sonnet",
    schemas=[UserProfile, WorkHistory]  # 结构化记忆 schema
)
# 自动从对话中提取记忆并存储到向量数据库
memories = manager.search("相关查询", user_id="user123")
```

**与 VCP 的关键差异**:

```
LangMem:
  + 自动从对话中提取记忆，无需 AI 主动写入
  + 结构化 Schema 保证记忆质量
  + 内置记忆反思 (Memory Reflection) 功能
  - 依赖 Anthropic/Claude 生态
  - 存储格式不透明，调试困难

VCP:
  + AI 主动写入，记忆质量由 AI 自主保证
  + Markdown 格式完全透明
  + 独创梦境整合机制 (LangMem 无对应)
  + 零外部服务依赖
```

### 19.5 RAG 框架对比 / RAG Framework Comparison

| 特性 / Feature | **VCP TagMemo** | **LlamaIndex** | **Haystack** | **Chroma+LangChain** |
|---|---|---|---|---|
| 向量引擎 | Rust USearch | FAISS/Qdrant/etc | 多种 | ChromaDB |
| Tag 引力场 | ✅ **独有** | ❌ | ❌ | ❌ |
| 正交基精排 | ✅ EPA **独有** | ❌ | ❌ | ❌ |
| 残差金字塔 | ✅ **独有** | ❌ | ❌ | ❌ |
| SVD 语义去重 | ✅ | ❌ | ✅ MMR | ❌ |
| 元思考链 | ✅ **独有** | ❌ | ❌ | ❌ |
| 对话中刷新 | ✅ **独有** | ❌ | ❌ | ❌ |
| Shotgun 多向量 | ✅ V4 **独有** | ❌ | ❌ | ❌ |
| 时间意图解析 | ✅ **独有** | ❌ | ❌ | ❌ |
| 动态 K/β 计算 | ✅ EPA 驱动 **独有** | ❌ | ❌ | ❌ |
| 热调控参数 | ✅ rag_params.json | ❌ | ❌ | ❌ |
| 多级缓存 | ✅ 3级 (查询/向量/AIMemo) | 🔍 | 🔍 | ❌ |
| 离线可用 | ✅ 完全离线 | 🔍 | 🔍 | ✅ |
| 编程语言 | Node.js + Rust | Python | Python | Python |

---

## 20. AI Coding Agent 调试指南 / AI Coding Agent Debug Guide

### 20.1 快速问题定位 / Quick Issue Localization

```bash
# 1. 检查 RAG 检索是否触发
grep "\[RAGDiaryPlugin\]" server_logs.txt

# 2. 检查 TagMemo 算法调试输出
grep "\[TagMemo-V3.7\]" server_logs.txt
# 输出格式: World=物理, Depth=0.732, Resonance=0.423
# 输出格式: Coverage=0.812, Explained=81.2%
# 输出格式: Effective Boost: 0.234, Dynamic Core Boost: 1.328

# 3. 检查动态参数计算
grep "\[RAGDiaryPlugin\]\[V3\]" server_logs.txt
# 输出格式: L=0.732, R=0.423, S=0.156 => Beta=0.681, TagWeight=0.318, K=7

# 4. 检查 EPA 初始化
grep "\[EPA\]" server_logs.txt

# 5. 检查索引状态
grep "\[KnowledgeBase\]" server_logs.txt | grep "✅\|❌\|⚠️"

# 6. 检查 Shotgun Query
grep "Shotgun Query" server_logs.txt
# 输出: Shotgun Query: Executing 4 parallel searches...
```

### 20.2 常见问题与解决方案 / Common Issues & Solutions

| 问题 / Issue | 诊断 / Diagnosis | 解决 / Fix |
|---|---|---|
| RAG 不触发 | system prompt 缺少 `[[日记本名]]` 占位符 | 检查 agent 配置 |
| 检索结果无关 | `Depth` 低 + `S` 高 → TagWeight 过小 | 调大 `rag_params.json` 中 `tagWeightRange[1]` |
| 检索结果重复 | `deduplicationThreshold` 过高 | 降低至 0.8~0.85 |
| 新日记没有被索引 | chokidar 未触发 / Embedding API 失败 | 检查 KNOWLEDGEBASE_ROOT_PATH 配置 + API 密钥 |
| 维度不匹配错误 | `VECTORDB_DIMENSION` 与实际模型不符 | 重置向量库: `node reset_vectordb.js` |
| EPA 未初始化 | Tag 数量 < 8 | 先写入足够多的日记，触发 Tag 积累 |
| AIMemo 无响应 | `[[AIMemo=True]]` 未在 system prompt 中 | 检查 agent system prompt 配置 |
| TagMemo 英文 Tag 被抑制 | 语言惩罚过强 | 调整 `LANG_PENALTY_CROSS_DOMAIN` |

### 20.3 向量维度不匹配修复流程

```bash
# 症状: "[KnowledgeBase] Dimension mismatch! Expected 3072, got 1536"
# 原因: 更换了 Embedding 模型 (如从 3-small 换到 gemini-embedding)

# 1. 更新配置
echo "VECTORDB_DIMENSION=3072" >> config.env
echo "WhitelistEmbeddingModel=google/gemini-embedding-001" >> config.env

# 2. 重置向量数据库 (⚠️ 会删除所有向量，但不影响 Markdown 原文件)
node reset_vectordb.js

# 3. 重建索引 (等待 chokidar 全量扫描)
node server.js  # 启动后自动全量扫描
# 或手动重建
node rebuild_vector_indexes.js
```

### 20.4 TagMemo 效果调优 / TagMemo Tuning

```json
// rag_params.json — 可热调控 (无需重启)
{
  "RAGDiaryPlugin": {
    "noise_penalty": 0.05,         // 增大 → 减少噪音 Tag 影响
    "tagWeightRange": [0.05, 0.45] // 增大上限 → 强化 Tag 引力
  },
  "KnowledgeBaseManager": {
    "activationMultiplier": [0.5, 1.5], // 控制 TagMemo 激活强度
    "dynamicBoostRange": [0.3, 2.0],    // Boost 因子范围
    "coreBoostRange": [1.20, 1.40],     // 核心 Tag 加权范围
    "deduplicationThreshold": 0.88,     // Tag 去重相似度阈值
    "techTagThreshold": 0.08,           // 英文技术 Tag 过滤门槛
    "normalTagThreshold": 0.015         // 普通 Tag 过滤门槛
  }
}
```

---

## 21. 维护与扩展指南 / Maintenance & Extension Guide

### 21.1 添加新的记忆写入方式 / Adding New Memory Writers

1. 在 `Plugin/` 下创建新插件目录
2. 实现 `plugin-manifest.json` (type: `synchronous` 或 `asynchronous`)
3. 最终写入 `.md` 或 `.txt` 文件到 `dailynote/<character>/` 下
4. KnowledgeBaseManager 的 chokidar 会自动监听并索引

### 21.2 修改 TagMemo 算法 / Modifying TagMemo

```
KnowledgeBaseManager.js → _applyTagBoostV3()   [line 446]
  ⚠️ 修改前必须理解整个 7 阶段管道
  ⚠️ 修改后需要: node rebuild_tag_index_custom.js 重建 Tag 索引
  ✅ 调优参数优先通过 rag_params.json 热调控，避免修改源码
```

### 21.3 添加新的 RAG 语法 / Adding New RAG Syntax

当前支持的占位符语法：
```
[[日记本名]]             ← 基础向量检索
[[A|B|C]]               ← 聚合多库检索
[[日记本名::AIMemo]]     ← AI 驱动召回
[[日记本名::Meta]]       ← 仅元思考
[[日记本名::Group]]      ← 语义组检索
[[VCP元思考:主题]]        ← 元思考推理链
《《日记本名》》           ← 混合模式
```

扩展新语法的入口：
- `RAGDiaryPlugin.processMessages()` [line 857] — 占位符扫描
- `RAGDiaryPlugin._parseAggregateSyntax()` [line 1554] — 语法解析
- `RAGDiaryPlugin._processSingleSystemMessage()` [line 1037] — 检索执行

### 21.4 Rust 向量引擎扩展 / Extending the Rust Vector Engine

```
rust-vexus-lite/
├── src/lib.rs           ← N-API 接口定义
└── Cargo.toml           ← 依赖管理

构建:
  cd rust-vexus-lite && npm run build

新增 JS API 步骤:
  1. 在 src/lib.rs 添加新 #[napi] 函数
  2. npm run build 重新编译
  3. 在 KnowledgeBaseManager.js 中调用新 API
```

### 21.5 关键配置文件一览 / Key Config Files

| 文件 / File | 用途 / Purpose | 是否热调控 / Hot Reload |
|---|---|---|
| `config.env` | 全局配置（API Key、维度等）| ❌ 需重启 |
| `rag_params.json` | TagMemo/RAG 算法参数 | ✅ 自动监听 |
| `Plugin/RAGDiaryPlugin/config.env` | RAG 插件配置 | ❌ 需重启 |
| `Plugin/RAGDiaryPlugin/meta_thinking_chains.json` | 元思考链定义 | ✅ 向量缓存重建 |
| `Plugin/RAGDiaryPlugin/semantic_groups.edit.json` | 语义组编辑 | ✅ 同步到 main |
| `Plugin/RAGDiaryPlugin/rag_tags.json` | 优先 Tag 列表 | 🔍 需验证 |
| `agent_map.json` | Agent 别名映射 | ✅ 热更新 |

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md)  
> ⬅️ 上一节 / Prev: [12_agent_task_schedule](../12_agent_task_schedule/README.md)  
> 🔗 相关 / Related: [10_agent_memory](../10_agent_memory/README.md) | [11_agent_rag](../11_agent_rag/README.md) | [03_memory_rag](../03_memory_rag/README.md)
