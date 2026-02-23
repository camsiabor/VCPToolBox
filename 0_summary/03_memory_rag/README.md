# 03 记忆与 RAG 系统 / Memory & RAG System

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `KnowledgeBaseManager.js`, `EPAModule.js`, `ResidualPyramid.js`, `ResultDeduplicator.js`, `rust-vexus-lite/`  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [系统概述 / System Overview](#1-系统概述--system-overview)
2. [架构总览 / Architecture Overview](#2-架构总览--architecture-overview)
3. [双索引系统 / Dual Index System](#3-双索引系统--dual-index-system)
4. [TagMemo 浪潮算法 / TagMemo Wave Algorithm](#4-tagmemo-浪潮算法--tagmemo-wave-algorithm)
5. [EPA 模块 / EPA Module](#5-epa-模块--epa-module)
6. [残差金字塔 / Residual Pyramid](#6-残差金字塔--residual-pyramid)
7. [SVD 结果去重器 / SVD Result Deduplicator](#7-svd-结果去重器--svd-result-deduplicator)
8. [Rust N-API 向量引擎 / Rust N-API Vector Engine](#8-rust-n-api-向量引擎--rust-n-api-vector-engine)
9. [RAG 参数热调控 / RAG Parameter Hot Tuning](#9-rag-参数热调控--rag-parameter-hot-tuning)
10. [日记系统集成 / Diary System Integration](#10-日记系统集成--diary-system-integration)

---

## 1. 系统概述 / System Overview

### 中文

VCP 记忆系统是一个基于向量语义检索的 **RAG (Retrieval-Augmented Generation)** 系统，为 AI Agent 提供**长期记忆**和**上下文感知**能力。

核心设计哲学：**向量空间并非平坦的，而是充满语义引力**。
- 标签（Tag）被视为语义空间中的**引力锚点**
- 算法根据感应到的标签引力，将向量向核心语义点进行"拉扯"和"扭曲"
- 目标：穿透表层文字，直达语义核心

### English

VCP's memory system is a **RAG (Retrieval-Augmented Generation)** system based on vector semantic retrieval, providing AI Agents with **long-term memory** and **contextual awareness**.

Core design philosophy: **Vector space is not flat — it is filled with semantic gravity.**
- Tags serve as **gravitational anchors** in semantic space
- The algorithm "pulls" and "warps" vectors toward core semantic points based on detected tag gravity
- Goal: penetrate surface text to reach the semantic core

---

## 2. 架构总览 / Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    KnowledgeBaseManager.js                       │
│  ┌────────────────┐ ┌──────────────┐ ┌──────────────────────┐  │
│  │  diaryIndices  │ │  tagIndex    │ │  SQLite              │  │
│  │  (Map 结构)    │ │ (VexusIndex) │ │  knowledge_base.db   │  │
│  │  每日记本独立  │ │  全局 Tag 索引│ │  WAL 模式 / ACID     │  │
│  └────────────────┘ └──────────────┘ └──────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────┐
│   EPAModule     │ │ ResidualPyramid │ │ ResultDeduplicator  │
│  (语义空间定位) │ │  (能量精细拆解) │ │  (智能结果去重)      │
│  Semantic       │ │  Residual       │ │  SVD-based          │
│  Projection     │ │  Energy Decomp  │ │  Deduplication      │
└─────────────────┘ └─────────────────┘ └─────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                   rust-vexus-lite/                               │
│              Rust N-API 向量引擎 / Rust N-API Vector Engine      │
│              USearch-based KNN search                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 双索引系统 / Dual Index System

### 3.1 `diaryIndices` — 日记本向量索引

```javascript
// KnowledgeBaseManager.js:59
this.diaryIndices = new Map();  // Map<diaryName, VexusIndex>
```

- **结构**: `Map<diaryName, VexusIndex>`
- **特性**: 每个日记本拥有独立的向量索引，互相隔离
- **懒加载**: 仅在首次访问时加载对应索引，节约内存
- **故障隔离**: 单个索引损坏不影响其他日记本

```javascript
// 懒加载示例 / Lazy load example
// KnowledgeBaseManager.js:210-220
async _getOrLoadDiaryIndex(diaryName) {
    if (this.diaryIndices.has(diaryName)) {
        return this.diaryIndices.get(diaryName);
    }
    const safeName = crypto.createHash('md5').update(diaryName).digest('hex');
    const idx = await this._loadOrBuildIndex(`diary_${safeName}`, 50000, 'chunks', diaryName);
    this.diaryIndices.set(diaryName, idx);
    return idx;
}
```

### 3.2 `tagIndex` — 全局 Tag 向量索引

```javascript
// KnowledgeBaseManager.js:60
this.tagIndex = null;  // VexusIndex (全局 / Global)
```

- **用途**: 存储所有 Tag 的向量表示，支持语义 Tag 检索
- **TagMemo 算法的基础**: Tag 向量用于计算语义引力
- **全局共享**: 所有日记本共用同一个 Tag 索引

---

## 4. TagMemo 浪潮算法 / TagMemo Wave Algorithm

> 详细数学推导见 / Detailed math derivation: `TagMemo_Wave_Algorithm_Deep_Dive.md`

### 4.1 算法直觉 / Algorithm Intuition

```
普通向量搜索（余弦相似度）:        TagMemo 浪潮算法:
                                   
query ─── 距离 ──► chunk          query ─── 语义引力场 ──► chunk
(表面词义匹配)                     (深层语义锚定)
```

**核心思想 / Core Idea**: 将 Tag（关键词/概念）作为**语义锚点**，在检索时重新塑形向量空间，使相关语义自然"浮现"（像浪潮一样）。

### 4.2 算法步骤 / Algorithm Steps

```
Step 1: 编码查询向量
  query_vec = Embedding(query_text)

Step 2: 检索相关 Tag（来自 tagIndex）
  relevant_tags = tagIndex.search(query_vec, top_k=N)

Step 3: 计算 Tag 引力（语义锚定）
  for each tag in relevant_tags:
    gravity = tag.similarity * tag.weight
    query_vec = query_vec + gravity * tag_vec  ← 向量"拉扯"

Step 4: 用重塑后的向量检索 chunks
  results = diaryIndex.search(query_vec_reshaped, top_k=K)

Step 5: EPA 投影优化（可选）
  results = EPA.project(results, context)

Step 6: 残差金字塔精排（可选）
  results = ResidualPyramid.rerank(results, query_vec)

Step 7: SVD 去重
  final_results = ResultDeduplicator.deduplicate(results)
```

### 4.3 关键参数 / Key Parameters

| 参数 / Parameter | 位置 / Location | 默认值 / Default | 说明 / Description |
|---|---|---|---|
| `tagInfluence` | `rag_params.json` | 0.3 | Tag 引力强度 / Tag gravity strength |
| `tagTopK` | `rag_params.json` | 5 | 检索 Tag 数量 / Number of tags to retrieve |
| `chunkTopK` | `rag_params.json` | 10 | 检索 chunk 数量 / Number of chunks to retrieve |
| `minSimilarity` | `rag_params.json` | 0.3 | 最低相似度阈值 / Minimum similarity threshold |

---

## 5. EPA 模块 / EPA Module

> 📁 `EPAModule.js`

**EPA = Embedding Projection Analysis（嵌入投影分析）**

### 功能 / Function

EPA 模块负责在**语义子空间**中对检索结果进行投影和重新排序，减少噪声，增强与查询意图的对齐。

### 核心概念 / Core Concepts

```
高维向量空间 (1536-dim)
         │
         ▼ 投影到查询相关子空间
低维语义子空间 (~64-dim)
         │
         ▼ 在子空间中重新计算相似度
精排结果 / Reranked results
```

- **降维投影**: 将高维向量投影到查询定义的语义相关子空间
- **信噪比提升**: 消除与查询无关的语义维度带来的噪声
- **计算效率**: 在低维子空间计算，速度更快

---

## 6. 残差金字塔 / Residual Pyramid

> 📁 `ResidualPyramid.js`

### 功能 / Function

残差金字塔对检索到的 chunk 进行**多层次语义能量分解**，识别出最具信息量的语义层级，优先返回信息密度高的内容。

### 金字塔结构 / Pyramid Structure

```
Level 3: 抽象语义层 (Abstract semantic) ←── 高权重 / High weight
Level 2: 主题语义层 (Topic semantic)
Level 1: 词汇语义层 (Lexical semantic)  ←── 低权重 / Low weight
Level 0: 原始向量   (Raw vector)
```

- 每个 chunk 在不同语义层级有不同的"能量分布"
- 算法根据查询的语义层级偏好，动态调整各层级权重
- 避免仅靠表层词汇相似度就返回不相关内容

---

## 7. SVD 结果去重器 / SVD Result Deduplicator

> 📁 `ResultDeduplicator.js`

### 功能 / Function

使用**奇异值分解 (SVD)** 检测并去除语义高度相似的冗余 chunk，确保返回给 AI 的上下文内容多样性最大化。

### 工作原理 / How It Works

```
检索结果 [chunk1, chunk2, ..., chunkN]
         │
         ▼ 计算结果向量矩阵 R
         │
         ▼ SVD 分解: R = U · Σ · V^T
         │
         ▼ 识别共线（高度相似）的向量对
         │
         ▼ 保留多样性最大的子集
最终结果 [diverse_chunk1, diverse_chunk2, ..., diverse_chunkK]
```

- **去冗余**: 避免 AI 接收到大量重复或相似的记忆片段
- **多样性最大化**: 优先保留语义最不相关（互补性最强）的结果
- **上下文质量提升**: AI 能看到更全面、更多样的相关记忆

---

## 8. Rust N-API 向量引擎 / Rust N-API Vector Engine

> 📁 `rust-vexus-lite/`  
> 详细文档 / Detailed docs: `docs/RUST_VECTOR_ENGINE.md`

### 架构 / Architecture

```
Node.js (KnowledgeBaseManager.js)
         │ N-API 调用 / N-API call
         ▼
rust-vexus-lite/
├── src/lib.rs           ← N-API 绑定层 / N-API binding layer
├── src/index.rs         ← 向量索引实现 / Vector index implementation
└── index.node           ← 编译产物 / Compiled artifact
```

### 关键接口 / Key Interface

| 方法 / Method | 说明 / Description |
|---|---|
| `createIndex(dim, maxElements)` | 创建新向量索引 / Create new vector index |
| `addVector(id, vector)` | 添加向量 / Add vector |
| `searchKNN(queryVec, k)` | K 近邻搜索 / K-nearest neighbor search |
| `saveIndex(path)` | 持久化索引 / Persist index to disk |
| `loadIndex(path)` | 加载索引 / Load index from disk |

### 构建 / Build

```bash
cd rust-vexus-lite
npm run build         # Release 构建
npm run build:debug   # Debug 构建（含调试符号）
```

⚠️ **前置要求 / Prerequisites**: Rust toolchain (`rustup`), `cargo`, Node.js `node-gyp` 依赖

---

## 9. RAG 参数热调控 / RAG Parameter Hot Tuning

> 📁 `rag_params.json`  
> 详细指南 / Tuning guide: `TAGMEMO_TUNING_GUIDE.md`

### 热调控机制 / Hot Tuning Mechanism

`rag_params.json` 支持**运行时热重载**，修改参数无需重启服务器：

```json
{
  "tagInfluence": 0.3,           // Tag 引力强度 (0.0-1.0)
  "tagTopK": 5,                  // 检索 Tag 数量
  "chunkTopK": 10,               // 检索 chunk 数量
  "minSimilarity": 0.3,          // 最低相似度阈值
  "epaEnabled": true,            // 启用 EPA 模块
  "residualPyramidEnabled": true,// 启用残差金字塔
  "deduplicationEnabled": true,  // 启用 SVD 去重
  "maxContextLength": 4000       // 最大上下文长度（tokens）
}
```

### 调优建议 / Tuning Tips

| 场景 / Scenario | 建议调整 / Recommended Adjustment |
|---|---|
| 检索结果太泛 / Results too broad | 提高 `minSimilarity` (0.5+) |
| 检索结果太少 / Too few results | 降低 `minSimilarity` (0.2-), 提高 `chunkTopK` |
| 语义匹配不精准 / Poor semantic match | 提高 `tagInfluence` (0.4-0.6) |
| 结果重复太多 / Too many duplicates | 确保 `deduplicationEnabled: true` |
| 响应太慢 / Too slow | 降低 `tagTopK`, `chunkTopK`；禁用 EPA/残差金字塔 |

---

## 10. 日记系统集成 / Diary System Integration

### 数据流 / Data Flow

```
AI 写入日记 / AI writes diary
  DailyNoteWrite 插件 → 写入 dailynote/<character>/<date>.md
         │
         ▼
chokidar 文件监听 / File watch
  KnowledgeBaseManager 检测新文件
         │
         ▼
文本分块 / Text chunking
  TextChunker.js → 分割为 chunks
         │
         ▼
向量化 / Embedding
  EmbeddingUtils.js → 调用 Embedding API → 获取向量
         │
         ▼
索引更新 / Index update
  VexusIndex.addVector() → 更新 diaryIndices[character]
  SQLite → 持久化 chunk 元数据
         │
         ▼
AI 检索记忆 / AI retrieves memory
  RAGDiaryPlugin / DailyNote → KnowledgeBaseManager.search()
  → TagMemo 算法检索 → 结果注入 system prompt
```

### 目录约定 / Directory Convention

```
dailynote/
├── <CharacterName>/       ← 每个角色独立目录 / Per-character directory
│   ├── YYYY-MM-DD.md      ← 日期日记文件 / Dated diary file
│   └── ...
└── ...
```

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ⬅️ 上一节 / Prev: [02_plugin_system](../02_plugin_system/README.md) | ➡️ 下一节 / Next: [04_api_routes](../04_api_routes/README.md)
