# 01 KnowledgeBaseManager — 知识库核心管理器详解
# 01 KnowledgeBaseManager — Core Knowledge Base Manager

> **所属 / Parent**: [README.md](./README.md)  
> **源文件 / Source**: `KnowledgeBaseManager.js` (1349 行 / lines)  
> **依赖 / Deps**: `EPAModule`, `ResidualPyramid`, `ResultDeduplicator`, `EmbeddingUtils`, `TextChunker`, `rust-vexus-lite`, `better-sqlite3`, `chokidar`

---

## 目录 / Table of Contents

1. [职责与定位 / Responsibility](#1-职责与定位--responsibility)
2. [构造函数与配置 / Constructor & Config](#2-构造函数与配置--constructor--config)
3. [初始化序列 / Initialization Sequence](#3-初始化序列--initialization-sequence)
4. [SQLite 数据库 Schema](#4-sqlite-数据库-schema)
5. [双索引系统 / Dual Index System](#5-双索引系统--dual-index-system)
6. [文件监听与批量索引 / File Watcher & Batch Indexing](#6-文件监听与批量索引--file-watcher--batch-indexing)
7. [搜索 API / Search API](#7-搜索-api--search-api)
8. [TagMemo V3.7 核心 / TagMemo V3.7 Core](#8-tagmemo-v37-核心--tagmemo-v37-core)
9. [Tag 共现矩阵 / Tag Co-occurrence Matrix](#9-tag-共现矩阵--tag-co-occurrence-matrix)
10. [热调控参数系统 / Hot-Reload Param System](#10-热调控参数系统--hot-reload-param-system)
11. [兼容性 API / Compatibility APIs](#11-兼容性-api--compatibility-apis)
12. [错误处理策略 / Error Handling](#12-错误处理策略--error-handling)
13. [性能特性 / Performance Characteristics](#13-性能特性--performance-characteristics)

---

## 1. 职责与定位 / Responsibility

KnowledgeBaseManager 是整个记忆系统的**向量基础设施层**，向上为 RAGDiaryPlugin 提供语义搜索 API，向下管理 SQLite 数据库、Rust VexusIndex、文件监听、Embedding 批量调用。

**它不处理提示词注入，不处理对话逻辑，只做：**

| 能力 / Capability | 接口 / Interface |
|---|---|
| 文本向量化 + 存储 | 自动（chokidar → _flushBatch） |
| KNN 向量搜索 | `search(diaryName, queryVec, k, tagBoost, coreTags)` |
| Tag 引力场变换 | `applyTagBoost(vector, tagBoost)` |
| EPA 语义分析 | `getEPAAnalysis(vector)` |
| 结果智能去重 | `deduplicateResults(candidates, queryVector)` |
| 日记本名称向量 | `getDiaryNameVector(diaryName)` |
| 相似 Tag 搜索 | `searchSimilarTags(input, k)` |
| KV 键值存储 | `kvGet(key)`, `kvSet(key, value, vector)` |

---

## 2. 构造函数与配置 / Constructor & Config

**📁 位置**: `KnowledgeBaseManager.js:27`

```javascript
constructor(config = {}) {
    this.config = {
        // ── 路径配置 / Path Config ──
        rootPath:   KNOWLEDGEBASE_ROOT_PATH,     // dailynote/ 根目录
        storePath:  KNOWLEDGEBASE_STORE_PATH,    // VectorStore/ 索引目录

        // ── 模型配置 / Model Config ──
        model:      WhitelistEmbeddingModel,     // 默认 google/gemini-embedding-001
        dimension:  VECTORDB_DIMENSION,          // ⚠️ 必须与模型一致，默认 3072

        // ── 批处理配置 / Batch Config ──
        batchWindow:      2000,    // 防抖窗口(ms)，新文件在 2s 内到达会合批
        maxBatchSize:     50,      // 单批最大文件数
        indexSaveDelay:   120000,  // 索引保存延迟(ms)
        tagIndexSaveDelay:300000,  // Tag 索引保存延迟(ms)

        // ── 过滤规则 / Filter Rules ──
        ignoreFolders:  ['VCP论坛'],  // 忽略的目录
        ignorePrefixes: ['已整理'],   // 忽略的文件名前缀
        ignoreSuffixes: ['夜伽'],     // 忽略的文件名后缀

        // ── Tag 控制 / Tag Control ──
        tagBlacklist:       Set<string>,  // 黑名单 (精确匹配)
        tagBlacklistSuper:  string[],     // 超级黑名单 (模糊匹配，TAG_BLACKLIST_SUPER)
        tagExpandMaxCount:  30,           // 逻辑拉回最大扩展 Tag 数

        // ── 语言补偿 / Language Compensation ──
        langConfidenceEnabled:   true,
        langPenaltyUnknown:      0.05,   // 语言未知时的惩罚
        langPenaltyCrossDomain:  0.1,    // 跨语言时的惩罚
    };
}
```

---

## 3. 初始化序列 / Initialization Sequence

**📁 位置**: `KnowledgeBaseManager.js:75` — `initialize()`

```
initialize()
  │
  ├─① fs.mkdir(storePath)                      确保 VectorStore/ 目录存在
  ├─② new Database(dbPath)                     连接 SQLite (WAL 模式)
  ├─③ _initSchema()                            建表 (files/chunks/tags/file_tags/kv_store)
  │
  ├─④ 加载 Tag 索引 (同步决策)
  │   ├── VexusIndex.load(tagIdxPath)          如果 .usearch 文件存在
  │   └── new VexusIndex() + _recoverTagsAsync()  否则新建并后台恢复
  │
  ├─⑤ _hydrateDiaryNameCacheSync()            同步预热日记本名称向量缓存
  │   └── SQL: SELECT key, value FROM kv_store WHERE key LIKE 'diary_name:%'
  │
  ├─⑥ _buildCooccurrenceMatrix()              构建 Tag 共现矩阵 (后台异步)
  │
  ├─⑦ EPAModule.initialize()                  加权 PCA 初始化正交语义基底
  │   └── 如果 epa_basis_cache 存在则从 kv_store 加载
  │
  ├─⑧ new ResidualPyramid(tagIndex, db)       初始化残差金字塔 (无需额外初始化)
  ├─⑨ new ResultDeduplicator(db)              初始化结果去重器
  │
  ├─⑩ _startWatcher()                         启动 chokidar 文件监听
  ├─⑪ loadRagParams()                         加载 rag_params.json
  └─⑫ _startRagParamsWatcher()               监听 rag_params.json 变更
```

**⚠️ 关键点**: 步骤 ⑤ 是**同步阻塞**的，确保 RAGDiaryPlugin 启动时日记本名称向量可用。

---

## 4. SQLite 数据库 Schema

**📁 位置**: `KnowledgeBaseManager.js:167` — `_initSchema()`

```sql
-- 文件元数据表 / File metadata
CREATE TABLE files (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    path        TEXT UNIQUE NOT NULL,      -- 相对路径 (从 rootPath 起)
    diary_name  TEXT NOT NULL,             -- 一级目录名 = 日记本名
    checksum    TEXT NOT NULL,             -- MD5 (内容变更检测)
    mtime       INTEGER NOT NULL,          -- 修改时间戳
    size        INTEGER NOT NULL,
    updated_at  INTEGER                    -- 更新时间戳
);

-- 文本块表 / Text chunks
CREATE TABLE chunks (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    file_id      INTEGER NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    chunk_index  INTEGER NOT NULL,
    content      TEXT NOT NULL,
    vector       BLOB                      -- Float32Array 的二进制序列化
);

-- 语义标签表 / Semantic tags
CREATE TABLE tags (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    name    TEXT UNIQUE NOT NULL,
    vector  BLOB                           -- Float32Array 的二进制序列化
);

-- 文件-标签关联表 / File-tag relations
CREATE TABLE file_tags (
    file_id  INTEGER NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    tag_id   INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (file_id, tag_id)
);

-- 键值存储 / KV store (also used for EPA cache, diary name vectors)
CREATE TABLE kv_store (
    key     TEXT PRIMARY KEY,
    value   TEXT,
    vector  BLOB   -- 可选：key 的向量表示，支持语义查找
);
```

**向量 BLOB 格式 / Vector BLOB format**:
```javascript
// 写入 / Write
const vecBuf = Buffer.from(new Float32Array(vectorArray).buffer);
stmt.run(vecBuf);

// 读取 / Read
const raw = row.vector;  // Buffer
const vec = new Float32Array(raw.buffer, raw.byteOffset, dimension);
```

---

## 5. 双索引系统 / Dual Index System

### 5.1 diaryIndices — 日记本向量索引 / Diary Vector Index

```javascript
// line 59
this.diaryIndices = new Map();  // Map<diaryName, VexusIndex>

// 懒加载 / Lazy load (line 210)
async _getOrLoadDiaryIndex(diaryName) {
    if (this.diaryIndices.has(diaryName)) return this.diaryIndices.get(diaryName);
    const safeName = crypto.createHash('md5').update(diaryName).digest('hex');
    const idx = await this._loadOrBuildIndex(
        `diary_${safeName}`,  // 文件名: diary_<md5>.usearch
        50000,                // capacity
        'chunks',             // 数据来源表
        diaryName             // 过滤条件
    );
    this.diaryIndices.set(diaryName, idx);
    return idx;
}
```

**文件命名 / File naming**: MD5 哈希防止路径特殊字符问题：
```
VectorStore/diary_a1b2c3d4e5f6....usearch  ← diary_<md5(diaryName)>
```

### 5.2 tagIndex — 全局 Tag 索引 / Global Tag Index

```javascript
// 固定文件路径 / Fixed file path
const tagIdxPath = path.join(storePath, 'index_global_tags.usearch');

// 所有 Agent 共享同一个 Tag 索引
// Tag 向量在写入时通过 Embedding API 计算
```

### 5.3 索引操作 API / Index Operation API

```javascript
// 搜索 / Search
results = idx.search(vectorBuffer, k)
// 返回: [{ id: number (chunk.id), score: number (余弦相似度) }]

// 添加 / Add
idx.add(chunkId, vectorBuffer)

// 删除 / Delete
if (idx.remove) idx.remove(chunkId)

// 保存 / Save
idx.save(filePath)

// 统计 / Stats
const { totalVectors } = idx.stats ? idx.stats() : { totalVectors: 1 }
```

---

## 6. 文件监听与批量索引 / File Watcher & Batch Indexing

**📁 位置**: `KnowledgeBaseManager.js:950` — `_startWatcher()`

### 6.1 监听与防抖 / Watch & Debounce

```
chokidar.watch(rootPath, { ignoreInitial: !fullScanOnStartup })
  事件: 'add' | 'change' → handleFile(filePath)
  事件: 'unlink' → _handleDelete(filePath)

handleFile(filePath):
  过滤检查:
    √ 目录在 ignoreFolders 中 → 跳过
    √ 文件名前缀匹配 ignorePrefixes → 跳过
    √ 文件名后缀匹配 ignoreSuffixes → 跳过
    √ 不是 .md 或 .txt → 跳过
  通过 → pendingFiles.add(filePath)
  
  if (pendingFiles.size >= maxBatchSize):
    _flushBatch()  ← 立即处理（防止内存积压）
  else:
    _scheduleBatch()  ← setTimeout(2000ms) 防抖
```

### 6.2 批量处理管道 / Batch Processing Pipeline

**📁 位置**: `KnowledgeBaseManager.js:981` — `_flushBatch()`

```
_flushBatch()
  加锁: isProcessing = true

  阶段1: 变更检测 (并行 I/O)
    Promise.all(batchFiles.map(async file => {
      stat = await fs.stat(file)
      if (mtime + size 未变) → 跳过
      content = await fs.readFile(file)
      checksum = MD5(content)
      if (checksum 未变) → 只更新 mtime/size，跳过重建向量
      doc = {
        relPath, diaryName, checksum, mtime, size,
        chunks: chunkText(content),          ← TextChunker 分块
        tags:   _extractTags(content)        ← 正则提取 Tag 行
      }
    }))

  阶段2: Embedding 批量调用
    allChunks = flatten(docs.map(d => d.chunks))
    newTags   = docs.flatMap(d => d.tags).filter(未在DB中)
    
    chunkVectors = await getEmbeddingsBatch(allChunks, config)
    tagVectors   = await getEmbeddingsBatch(newTags, config)
    (Tag 分批: 每批 ≤ 100 个，防止API限制)

  阶段3: SQLite 事务写入 (原子)
    db.transaction(() => {
      for (每个 doc):
        获取旧 chunkIds (用于后续 Vexus 删除)
        UPDATE/INSERT files
        DELETE old chunks
        INSERT new chunks (with vectorBLOB)
        DELETE old file_tags
        INSERT new file_tags
      for (每个新 tag):
        INSERT OR IGNORE tags
        UPDATE tags SET vector=?
    })()

  阶段4: VexusIndex 更新
    for (旧 chunkId): idx.remove(id)       ← 先删除防 Duplicate Key
    for (新 chunk):   idx.add(id, vecBuf)  ← 再添加
    for (新 tag):     tagIndex.add(id, vecBuf) (带 Upsert 容错)

  阶段5: 延迟保存 (防止频繁 I/O)
    _scheduleIndexSave('global_tags')      ← 300s 后保存
    for (每个 diary): _scheduleIndexSave(diaryName) ← 120s 后保存

  解锁: isProcessing = false
  if (pendingFiles 非空) → setImmediate(_flushBatch)  ← 继续处理队列
```

### 6.3 Tag 提取逻辑 / Tag Extraction

**📁 位置**: `KnowledgeBaseManager.js:_extractTags()`

```javascript
_extractTags(content) {
    // 匹配 "Tag: 量子力学, 物理学, 学习" 格式
    const tagLineMatch = content.match(/^Tag:\s*(.+)$/im);
    if (!tagLineMatch) return [];
    
    const tagLine = tagLineMatch[1];
    const tags = tagLine.split(/[,，、;；]/).map(t => t.trim()).filter(Boolean);
    
    // 过滤黑名单
    return tags.filter(t => 
        !this.config.tagBlacklist.has(t.toLowerCase()) &&
        !this.config.tagBlacklistSuper.some(b => t.includes(b))
    );
}
```

---

## 7. 搜索 API / Search API

**📁 位置**: `KnowledgeBaseManager.js:277` — `search()`

### 7.1 函数签名（重载）

```javascript
// 形式1: 指定日记本搜索（最常用）
search(diaryName: string, queryVec: number[], k: number, tagBoost: number, coreTags: string[])

// 形式2: 全局搜索（搜所有日记本）
search(queryVec: number[], k: number, tagBoost: number)
```

### 7.2 内部路径

```
search()
  ├── diaryName 不为空 → _searchSpecificIndex(diaryName, vector, k, tagBoost, coreTags)
  └── diaryName 为空   → _searchAllIndices(vector, k, tagBoost)

_searchSpecificIndex():
  ① 获取或懒加载 VexusIndex
  ② if (tagBoost > 0):
       _applyTagBoostV3(queryVec, tagBoost, coreTags)  ← TagMemo 核心
  ③ idx.search(searchBuffer, k)
  ④ hydrate: SQL JOIN chunks + files → { text, score, sourceFile, matchedTags, ... }

_searchAllIndices():
  ① 并行搜索所有日记本 (Promise.all)
  ② 合并 + 按 score 排序
  ③ 截取前 K 个并 hydrate
```

### 7.3 返回数据结构

```javascript
// 搜索结果条目 / Search result entry
{
    text:           string,   // chunk 文本内容
    score:          number,   // 余弦相似度 (0~1)
    sourceFile:     string,   // 文件名 (basename)
    fullPath:       string,   // 文件完整路径
    matchedTags:    string[], // TagMemo 匹配的 Tag 名称列表
    boostFactor:    number,   // 实际应用的 boost 强度
    tagMatchScore:  number,   // Tag 匹配总得分
    tagMatchCount:  number,   // 匹配的 Tag 数量
    coreTagsMatched:string[], // 核心 Tag 命中列表
}
```

---

## 8. TagMemo V3.7 核心 / TagMemo V3.7 Core

**📁 位置**: `KnowledgeBaseManager.js:446` — `_applyTagBoostV3()`  
→ 详见 [README.md §6](./README.md#6-tagmemo-v37-数学管道全解析--tagmemo-v37-full-mathematical-pipeline) 与 [03_epa_residual_result.md](./03_epa_residual_result.md)

**简要流程 / Brief flow**:
```
① EPA.project() → logicDepth(L), entropy(H), resonance(R)
② ResidualPyramid.analyze() → tagMemoActivation, coverage
③ 计算 effectiveTagBoost = f(L, H, R, activation)
④ 收集 pyramid.levels 中所有 Tags，应用世界观门控 + 语言补偿
⑤ 逻辑拉回: tagCooccurrenceMatrix 扩充强关联 Tag
⑥ 核心 Tag 补全 (coreTagSet 中缺失的强制补充)
⑦ 批量查 Tag 向量 (1次 SQL)
⑧ 语义去重 (余弦相似度 > 0.88 的 Tag 合并)
⑨ 构建上下文向量: contextVec = Σ(tagVec × weight) / totalWeight
⑩ 最终融合: fused = normalize((1-β)·q + β·contextVec)
```

---

## 9. Tag 共现矩阵 / Tag Co-occurrence Matrix

**📁 位置**: `KnowledgeBaseManager.js:_buildCooccurrenceMatrix()`

**目的**: 捕获 Tag 之间的同文件出现规律，用于"逻辑拉回"（Logic Pull-back）步骤。

```javascript
// 构建: 启动时或有新文件写入后异步构建
// 数据结构: Map<tagId, Map<coTagId, coCount>>

// SQL 查询（三表连接）:
SELECT ft1.tag_id, ft2.tag_id, COUNT(*) as cocount
FROM file_tags ft1
JOIN file_tags ft2 ON ft1.file_id = ft2.file_id AND ft1.tag_id < ft2.tag_id
GROUP BY ft1.tag_id, ft2.tag_id
ORDER BY cocount DESC
LIMIT 10000

// 使用场景 (TagMemo 步骤 4.5):
// 高权重 Tag 的前 5 名 → 各自的前 4 个共现 Tag
// 拉回权重 = 父 Tag 权重 × 0.5
```

---

## 10. 热调控参数系统 / Hot-Reload Param System

**📁 位置**: `KnowledgeBaseManager.js:130-165`

```javascript
// 加载 rag_params.json
async loadRagParams() {
    const data = await fs.readFile('rag_params.json', 'utf-8');
    this.ragParams = JSON.parse(data);
}

// chokidar 监听 rag_params.json
// 文件变更时自动重新加载，无需重启服务器
this.ragParamsWatcher = chokidar.watch(paramsPath);
this.ragParamsWatcher.on('change', async () => {
    await this.loadRagParams();
});

// 在 _applyTagBoostV3 中使用
const config = this.ragParams?.KnowledgeBaseManager || {};
const boostRange = config.dynamicBoostRange || [0.3, 2.0];
```

**完整参数结构**: 见 [README.md §20.4](./README.md#20-ai-coding-agent-调试指南--ai-coding-agent-debug-guide)

---

## 11. 兼容性 API / Compatibility APIs

### 11.1 getDiaryNameVector

```javascript
// 用于 RAGDiaryPlugin 判断日记本与查询的语义相似度（聚合检索路由）
async getDiaryNameVector(diaryName) {
    // 1. 查内存缓存 (diaryNameVectorCache Map)
    // 2. 查 kv_store WHERE key = 'diary_name:<diaryName>'
    // 3. 未命中 → 调用 Embedding API 并存入 kv_store
}
```

### 11.2 searchSimilarTags

```javascript
// 用于 LightMemo 的 Tag 扩展
async searchSimilarTags(input, k = 10) {
    // input: 文本 or 向量
    // 直接在 tagIndex 中 KNN 搜索
    // 返回: [{ id, name, score }]
}
```

### 11.3 kvGet / kvSet

```javascript
// 通用键值存储，支持可选语义向量
kvGet(key)                        // 精确 key 查找
kvSet(key, value, vector)         // 存储 (可选附带向量)
kvSearchSimilar(queryVec, k)      // 向量 KNN 搜索 kv_store
```

---

## 12. 错误处理策略 / Error Handling

| 场景 / Scenario | 策略 / Strategy |
|---|---|
| Vexus 引擎缺失 | `process.exit(1)` — 致命错误 |
| Tag 索引损坏 | 新建空索引 + `_recoverTagsAsync()` 后台重建 |
| 日记本索引损坏 | 新建空索引 + 下次批处理时重建 |
| 维度不匹配 | 打印错误 + 返回空数组（不崩溃） |
| Duplicate Key | Upsert 策略：先 remove() 再 add() |
| Embedding API 失败 | 3次重试 + 指数退避，超时后跳过 |
| 文件读取失败 | 打印 warn + 跳过该文件（不影响其他文件） |

---

## 13. 性能特性 / Performance Characteristics

| 操作 / Operation | 时间复杂度 | 说明 |
|---|---|---|
| KNN 搜索 | O(log N) | USearch HNSW 算法 |
| Tag 引力场变换 | O(M×D) | M=Tag数, D=维度 |
| 批量 Embedding | O(B÷C) 并发 | B=批次数, C=并发数(默认5) |
| SQLite 事务写入 | O(N) | N=chunk数，WAL 模式加速 |
| 共现矩阵构建 | O(T²×F) | T=Tag数, F=文件数，一次性 |

**内存占用估算 / Memory estimation**:
```
每个 3072 维向量 = 3072 × 4 bytes = ~12 KB
10000 chunks × 12KB = ~120 MB (diaryIndex)
50000 tags × 12KB   = ~600 MB (tagIndex)

实际: USearch 压缩存储，比上述估算少 30~50%
```

---

> ⬆️ [README.md](./README.md) | ➡️ [02_rag_diary_plugin.md](./02_rag_diary_plugin.md)
