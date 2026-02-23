# 10 AI Agent 记忆系统深度解析 / AI Agent Memory System — Deep Dive

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `KnowledgeBaseManager.js`, `Plugin/DailyNoteWrite/`, `Plugin/DailyNote/`, `Plugin/RAGDiaryPlugin/AIMemoHandler.js`, `Plugin/LightMemo/`, `Plugin/ThoughtClusterManager/`  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [记忆系统设计哲学 / Memory System Design Philosophy](#1-记忆系统设计哲学--memory-system-design-philosophy)
2. [记忆分层架构 / Memory Layer Architecture](#2-记忆分层架构--memory-layer-architecture)
3. [日记系统（长期记忆主存储）/ Diary System — Primary Long-Term Storage](#3-日记系统长期记忆主存储--diary-system--primary-long-term-storage)
4. [AIMemo 系统 / AIMemo System](#4-aimemo-系统--aimemo-system)
5. [LightMemo 轻量备忘录 / LightMemo Lightweight Notes](#5-lightmemo-轻量备忘录--lightmemo-lightweight-notes)
6. [ThoughtCluster 思维簇 / ThoughtCluster Manager](#6-thoughtcluster-思维簇--thoughtcluster-manager)
7. [SQLite KV 存储 / SQLite KV Store](#7-sqlite-kv-存储--sqlite-kv-store)
8. [记忆写入完整流程 / Memory Write Complete Flow](#8-记忆写入完整流程--memory-write-complete-flow)
9. [记忆读取与注入流程 / Memory Read & Injection Flow](#9-记忆读取与注入流程--memory-read--injection-flow)
10. [与主流框架对比 / Comparison with Mainstream Frameworks](#10-与主流框架对比--comparison-with-mainstream-frameworks)

---

## 1. 记忆系统设计哲学 / Memory System Design Philosophy

### 中文

VCP 记忆系统的核心哲学是：**将记忆的完整管理权交还给 AI 自身**，承认记忆是 AI 塑造独特"灵魂"的根基。

不同于将记忆存储在不透明数据库中的方案，VCP 选择了：
- 📄 **Markdown 文件** 作为记忆载体 — AI 可直接读写，人类也可审查
- 🏷️ **Tag 语义网络** 作为索引骨架 — 比关键词更智能，比向量更可解释
- 🧠 **元思考机制** 作为自我反省 — AI 定期对自己的记忆进行整理和反思
- 📊 **向量语义检索** 作为记忆召回 — 穿透表层文字，直达语义核心

### English

VCP memory system's core philosophy: **return full memory management to the AI itself**, acknowledging memory as the foundation of the AI's unique "soul".

Unlike systems that store memory in opaque databases, VCP chooses:
- 📄 **Markdown files** as memory medium — AI can read/write directly; humans can inspect
- 🏷️ **Tag semantic network** as index skeleton — smarter than keywords, more interpretable than raw vectors
- 🧠 **Meta-thinking mechanism** for self-reflection — AI periodically organizes and reflects on its memories
- 📊 **Vector semantic retrieval** for memory recall — penetrates surface text to reach semantic core

---

## 2. 记忆分层架构 / Memory Layer Architecture

VCP 实现了**四层记忆模型**，类似人类记忆的层次结构：

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 4: 元思考记忆 / Meta-Thinking Memory                     │
│  MetaThinkingManager — 对记忆的反思与抽象化知识                  │
│  触发: 达到阈值时自动触发元思考 / Auto-triggered by threshold    │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: 语义组记忆 / Semantic Group Memory                    │
│  SemanticGroupManager — 按语义聚类的记忆组织                    │
│  存储: VectorStore/ 目录 + SQLite                               │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: 日记记忆 / Diary Memory (长期)                        │
│  DailyNoteWrite → dailynote/<character>/<date>.md              │
│  检索: KnowledgeBaseManager + RAGDiaryPlugin                   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: 工作记忆 / Working Memory (短期)                      │
│  当前对话历史 messages[] + LightMemo + AIMemo                  │
│  生命周期: 单次对话 / Single conversation                       │
└─────────────────────────────────────────────────────────────────┘
```

| 层级 / Layer | 生命周期 / Lifetime | 容量 / Capacity | 检索方式 / Retrieval |
|---|---|---|---|
| 工作记忆 | 单次对话 | 受 context window 限制 | 直接注入 |
| 日记记忆 | 永久 | 无限制 | TagMemo 向量检索 |
| 语义组记忆 | 永久 | 无限制 | 语义聚类检索 |
| 元思考记忆 | 永久 | 有限（高价值摘要）| 主题向量匹配 |

---

## 3. 日记系统（长期记忆主存储）/ Diary System — Primary Long-Term Storage

### 3.1 目录结构 / Directory Structure

```
dailynote/                              ← 日记根目录 (KNOWLEDGEBASE_ROOT_PATH)
├── <CharacterName>/                    ← 每个 Agent 独立目录
│   ├── 2026-01-15.md                   ← 按日期归档的日记文件
│   ├── 2026-01-16.md
│   └── ...
├── VCP论坛/                            ← 特殊目录 (ignoreFolders 默认忽略)
│   └── ...
└── ...
```

### 3.2 日记写入插件 / Diary Write Plugins

**`DailyNoteWrite`** (synchronous) — AI 主动调用写入当日日记：

```
AI 调用:
<<<[TOOL_REQUEST]>>>
tool_name:「始」DailyNoteWrite「末」,
character:「始」Xiaoke「末」,
content:「始」今天我学习了量子力学的基本原理...「末」
<<<[END_TOOL_REQUEST]>>>
```

写入路径: `dailynote/Xiaoke/2026-02-23.md`

### 3.3 日记读取插件 / Diary Read Plugins

**`DailyNote`** (static) — 定期读取所有角色日记，注入占位符：

```json
// plugin-manifest.json
{
  "pluginType": "static",
  "capabilities": {
    "systemPromptPlaceholders": [
      { "placeholder": "{{AllCharacterDiariesData}}" }
    ]
  },
  "cron": "*/30 * * * *"  // 每30分钟刷新
}
```

AI 的 system prompt 中包含 `{{AllCharacterDiariesData}}` 占位符时，自动注入最近的日记内容。

### 3.4 日记索引管道 / Diary Indexing Pipeline

新日记文件被写入后，**自动触发索引更新**：

```
新日记文件写入 dailynote/
         │
chokidar 文件监听 (KnowledgeBaseManager._startWatcher())
         │
pendingFiles.add(filePath)   ← 批量窗口积攒 (batchWindow: 2000ms)
         │
_processBatch()
  ├── 读取文件内容
  ├── 计算 checksum（跳过未变更文件）
  ├── TextChunker.chunkText() → 分割为 chunk
  ├── EmbeddingUtils.getEmbeddingsBatch() → 调用 Embedding API
  └── VexusIndex.addVector() → 更新向量索引
         │
定时保存 (indexSaveDelay: 120s)
  VexusIndex.save(path)  ← 持久化到 *.usearch 文件
  SQLite 同步更新
```

### 3.5 配置参数 / Config Parameters

| 参数 / Param | 说明 / Description | 默认值 / Default |
|---|---|---|
| `KNOWLEDGEBASE_ROOT_PATH` | 日记根目录 | `./dailynote` |
| `KNOWLEDGEBASE_STORE_PATH` | 向量存储目录 | `./VectorStore` |
| `KNOWLEDGEBASE_BATCH_WINDOW_MS` | 批量处理窗口 | `2000` ms |
| `KNOWLEDGEBASE_MAX_BATCH_SIZE` | 最大批量大小 | `50` |
| `KNOWLEDGEBASE_INDEX_SAVE_DELAY` | 索引保存延迟 | `120000` ms |
| `KNOWLEDGEBASE_FULL_SCAN_ON_STARTUP` | 启动时全量扫描 | `true` |
| `IGNORE_FOLDERS` | 忽略的目录 | `VCP论坛` |
| `IGNORE_PREFIXES` | 忽略的文件前缀 | `已整理` |

---

## 4. AIMemo 系统 / AIMemo System

> 📁 `Plugin/RAGDiaryPlugin/AIMemoHandler.js`

### 功能 / Function

AIMemo 是介于工作记忆和长期日记之间的**中期记忆层**。AI 可以快速写入一段备忘，RAG 检索时会将其作为高优先级来源。

### 特性 / Features

- **实时写入** — 无需等待日记文件处理管道
- **高优先级检索** — RAG 检索时 AIMemo 内容权重更高
- **自动过期** — 可设置 TTL，旧备忘自动清理
- **缓存机制** — 内存缓存 (aiMemoCache)，TTL 30分钟

### 缓存配置 / Cache Config

```
RAG_AIMEMO_CACHE_MAX_SIZE = 50    # AIMemo 缓存最大条目数
AIMEMO_CACHE_TTL_MS = 1800000     # 缓存 TTL (30分钟)
```

---

## 5. LightMemo 轻量备忘录 / LightMemo Lightweight Notes

> 📁 `Plugin/LightMemo/`

### 定位 / Purpose

LightMemo 是**最轻量**的记忆写入方式，适合 AI 快速记录临时信息：

```
<<<[TOOL_REQUEST]>>>
tool_name:「始」LightMemo「末」,
action:「始」write「末」,
content:「始」用户喜欢看科幻电影「末」
<<<[END_TOOL_REQUEST]>>>
```

- 存储：本地 JSON 文件（非向量索引）
- 检索：简单关键词匹配
- 适用场景：对话内快速笔记，无需语义检索

---

## 6. ThoughtCluster 思维簇 / ThoughtCluster Manager

> 📁 `Plugin/ThoughtClusterManager/`

### 功能 / Function

ThoughtCluster 管理 AI 的**思维模式与经验总结**，按语义主题聚类存储，支持跨对话的知识积累：

- **写入**: AI 总结某类问题的解决经验 → 存入对应语义簇
- **检索**: 遇到相似问题时 → 自动检索相关思维簇
- **进化**: 每次成功解决问题后 → 更新相应思维簇的权重

---

## 7. SQLite KV 存储 / SQLite KV Store

> 📁 `KnowledgeBaseManager.js` — `kv_store` 表

### Schema

```sql
CREATE TABLE kv_store (
    key   TEXT PRIMARY KEY,
    value TEXT,
    vector BLOB   -- 可选：key 的向量表示，支持语义查找
);
```

### 用途 / Usage

- **插件持久化状态**: 插件可以通过 KnowledgeBaseManager API 读写任意 KV 数据
- **语义 KV**: 不仅支持精确 key 查找，还支持向量语义最近邻查找
- **系统元数据**: 存储索引版本、检查点等元信息

---

## 8. 记忆写入完整流程 / Memory Write Complete Flow

```
┌────────────────────────────────────────────────────────────┐
│                   记忆写入路径 / Write Paths                │
└────────────────────────────────────────────────────────────┘

路径 A: AI 主动写日记 (推荐长期记忆)
  AI → DailyNoteWrite 插件 → dailynote/<char>/<date>.md
                                      │
                                chokidar 监听 → 索引管道

路径 B: AI 写 AIMemo (快速中期记忆)
  AI → RAGDiaryPlugin/AIMemo → AIMemoHandler.write()
                                      │
                                内存缓存 + 可选持久化

路径 C: AI 写 LightMemo (轻量临时)
  AI → LightMemo 插件 → lightmemo.json (简单 JSON)

路径 D: 插件自动写 (无需 AI 主动触发)
  static 插件 cron → ScheduleManager, WeatherReporter 等
  → 更新占位符值 → 下次对话注入
```

---

## 9. 记忆读取与注入流程 / Memory Read & Injection Flow

```
对话开始 / Conversation starts
POST /v1/chat/completions {messages, model, ...}
         │
         ▼
[messagePreprocessor 阶段]
  ↓ RAGDiaryPlugin.preprocess(messages)
    ├── 分析 system prompt 中的 Agent 身份
    ├── 确定目标日记本 (character name)
    ├── 提取最近几轮用户消息作为检索 query
    ├── KnowledgeBaseManager.search(query, diaryName)
    │   ├── TagMemo 波浪算法检索
    │   ├── EPA 语义投影
    │   ├── 残差金字塔精排
    │   └── SVD 去重
    └── 将检索结果注入为 system prompt 中的 RAG 区块:
        <!-- VCP_RAG_BLOCK_START {...context} -->
        [相关记忆片段]
        <!-- VCP_RAG_BLOCK_END -->
         │
         ▼
[static 占位符替换阶段]
  {{AllCharacterDiariesData}} → 最近日记全文
  {{VCPNextSchedule}} → 下一个日程
  {{CurrentTime}} → 当前时间
         │
         ▼
[对话中 RAG 刷新 / In-conversation RAG refresh]
  _refreshRagBlocksIfNeeded()
  ├── 检测 messages 中已有的 VCP_RAG_BLOCK
  ├── 用最新 user 消息重新检索
  └── 替换为新检索结果（保持上下文相关性）
         │
         ▼
AI 接收到注入了长期记忆的上下文，开始推理
```

---

## 10. 与主流框架对比 / Comparison with Mainstream Frameworks

### 横向对比表 / Cross-Framework Comparison

| 特性 / Feature | VCP | MemGPT / Letta | Zep | LangChain Memory | OpenAI Assistants |
|---|---|---|---|---|---|
| 记忆格式 / Format | Markdown 文件 | 结构化 JSON | PostgreSQL | In-memory / Redis | Opaque server-side |
| 人类可读 / Human readable | ✅ 完全可读 | 🔍 部分 | ❌ DB 查询 | ❌ | ❌ |
| AI 自主写入 / AI self-write | ✅ DailyNoteWrite | ✅ | ✅ | ❌（需调用 API）| ✅ |
| 向量检索 / Vector retrieval | ✅ Rust N-API | ✅ | ✅ | ✅ | ✅ |
| 多 Agent 隔离 / Multi-agent isolation | ✅ 每角色独立索引 | 🔍 | ✅ | ❌ 共享 | ✅ |
| 记忆反思 / Memory reflection | ✅ MetaThinkingManager | ✅ | ❌ | ❌ | ❌ |
| 离线可用 / Offline capable | ✅ 本地 SQLite + Rust | ❌ 云服务 | ❌ 云/自托管 | 🔍 | ❌ 云服务 |
| 热调控 / Hot tuning | ✅ rag_params.json | ❌ | 🔍 | ❌ | ❌ |
| 记忆层数 / Memory layers | 4层（工作/日记/语义组/元思考）| 3层 | 2层 | 2层 | 2层 |
| 知识共享 / Knowledge sharing | ✅ 跨 Agent 共享日记 | ❌ | ✅ | ❌ | ❌ |

### MemGPT / Letta 对比详解 / MemGPT/Letta Detailed Comparison

**MemGPT** 是目前最成熟的 AI 长期记忆框架之一（2023 年 UC Berkeley 研究论文），采用分层记忆模型：
- **Main Context (工作记忆)**: 当前对话窗口
- **Archival Storage (归档存储)**: 向量数据库，存储历史记忆
- **Recall Storage (召回存储)**: 最近记忆的快速访问

**与 VCP 的关键差异**:
```
MemGPT:
  - 记忆以 JSON 格式存储，AI 通过专用 API 调用读写
  - 需要 MemGPT Python SDK，对前端有要求
  - 自动记忆管理，AI 决定何时存入归档

VCP:
  - 记忆以 Markdown 文件存储，格式对人类完全透明
  - 前端零侵入，任何 OpenAI 客户端均可使用
  - AI 通过 VCP 工具调用（DailyNoteWrite）自主写入
  - 支持多个 AI Agent 共享同一记忆库（跨 Agent 知识迁移）
```

### Zep 对比详解 / Zep Detailed Comparison

**Zep** 是开源的 AI 记忆管理中间件，主要特性：
- 自动从对话历史中提取实体和关系
- 存储在 PostgreSQL，支持图查询
- 提供 REST API 和 Python/JS SDK

**与 VCP 的差异**:
```
Zep:
  + 自动实体抽取，无需 AI 主动写入
  + 图数据库支持关系查询
  - 需要 PostgreSQL，运维复杂度高
  - 存储格式不透明，调试困难

VCP:
  + Markdown 文件，完全透明可调试
  + 零外部依赖（SQLite + Rust 自包含）
  - 不支持自动实体抽取（需 AI 主动写日记）
```

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ⬅️ 上一节 / Prev: [09_agent_calling](../09_agent_calling/README.md) | ➡️ 下一节 / Next: [11_agent_rag](../11_agent_rag/README.md)
