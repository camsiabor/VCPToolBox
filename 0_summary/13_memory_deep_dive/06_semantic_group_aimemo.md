# 06 SemanticGroupManager + AIMemoHandler — 语义组与 AI 驱动记忆
# 06 SemanticGroupManager + AIMemoHandler — Semantic Groups & AI Memory

> **所属 / Parent**: [README.md](./README.md)  
> **源文件 / Sources**:  
> - `Plugin/RAGDiaryPlugin/SemanticGroupManager.js` (461 行)  
> - `Plugin/RAGDiaryPlugin/AIMemoHandler.js` (约 350 行)

---

## 目录 / Table of Contents

1. [SemanticGroupManager — 语义组管理器](#1-semanticgroupmanager--语义组管理器)
   - 1.1 [设计目的 / Design Purpose](#11-设计目的--design-purpose)
   - 1.2 [数据结构 / Data Structure](#12-数据结构--data-structure)
   - 1.3 [初始化与文件同步 / Init & File Sync](#13-初始化与文件同步--init--file-sync)
   - 1.4 [组激活检测 detectAndActivateGroups()](#14-组激活检测-detectandactivategroups)
   - 1.5 [向量增强 getEnhancedVector()](#15-向量增强-getenhancedvector)
   - 1.6 [向量预计算 precomputeGroupVectors()](#16-向量预计算-precomputegroupvectors)
   - 1.7 [原子写入保护 / Atomic Write Protection](#17-原子写入保护--atomic-write-protection)
   - 1.8 [运营指南 / Operations Guide](#18-运营指南--operations-guide)
2. [AIMemoHandler — AI 驱动记忆召回](#2-aimemohandler--ai-驱动记忆召回)
   - 2.1 [设计目的 / Design Purpose](#21-设计目的--design-purpose)
   - 2.2 [配置参数 / Config](#22-配置参数--config)
   - 2.3 [单库处理 processAIMemo()](#23-单库处理-processaimemo)
   - 2.4 [聚合处理 processAIMemoAggregated()](#24-聚合处理-processaimemoaggregated)
   - 2.5 [分批策略 / Batching Strategy](#25-分批策略--batching-strategy)
   - 2.6 [缓存机制 / Cache Mechanism](#26-缓存机制--cache-mechanism)
3. [协作关系 / Collaboration](#3-协作关系--collaboration)

---

## 1. SemanticGroupManager — 语义组管理器

**📁 位置**: `Plugin/RAGDiaryPlugin/SemanticGroupManager.js`

### 1.1 设计目的 / Design Purpose

语义组解决了一个核心问题：**用户可能用不同表达指代同一概念**。

```
示例 / Example:
  "量子力学" ≡ "量子物理" ≡ "量子纠缠" ≡ "量子叠加"
  "编程" ≡ "Python" ≡ "代码" ≡ "编码"

配置语义组后:
  [[小可::Group]] → 用户提到任何组内词汇
                    → 自动激活该组
                    → 将组向量混合进查询向量
                    → 提升该主题的检索召回率
```

### 1.2 数据结构 / Data Structure

```json
// semantic_groups.json
{
  "config": {
    "activationThreshold": 0.3,    // 激活强度阈值
    "maxGroupsToActivate": 3       // 最多同时激活 3 个组
  },
  "groups": {
    "量子力学": {
      "words": ["量子", "纠缠", "叠加", "薛定谔", "不确定性"],
      "auto_learned": ["量子计算", "量子比特"],  // 自动学习的扩展词
      "weight": 1.2,                             // 组权重（向量混合时使用）
      "vector_id": "uuid-v4-string",             // 对应 semantic_vectors/ 目录下的文件名
      "words_hash": "sha256-of-sorted-words",    // 词汇变更检测
      "last_activated": "2026-02-20T14:00:00Z",
      "activation_count": 42
    }
  }
}
```

**向量存储分离设计** (防止大文件):
```
Plugin/RAGDiaryPlugin/
├── semantic_groups.json          ← 组配置（不含向量）
├── semantic_groups.edit.json     ← 编辑文件（热更新入口）
└── semantic_vectors/
    ├── <uuid>.json               ← 每个组的向量（独立文件）
    └── <uuid>.json
```

### 1.3 初始化与文件同步 / Init & File Sync

```
初始化时序:

① fs.mkdir(semantic_vectors/)   创建向量目录
② synchronizeFromEditFile()     同步编辑文件
③ loadGroups()                  加载主配置
④ precomputeGroupVectors()      预计算/更新向量

synchronizeFromEditFile():
  读取 semantic_groups.edit.json（如存在）
  比较与 semantic_groups.json 的核心数据差异:
    - config 是否变化
    - 组名是否增减
    - words 是否变化
  如果有差异 → 智能合并:
    使用 edit.json 的 words（新词汇）
    保留 main.json 的 vector_id 等元数据（避免不必要的重新向量化）
  原子写入（rename 操作）
```

**热更新工作流**:
```
用户编辑语义组 → 修改 semantic_groups.edit.json
                → chokidar 检测 (或 AdminPanel API 调用)
                → synchronizeFromEditFile() 自动合并
                → precomputeGroupVectors() 更新向量
                → 无需重启服务器
```

### 1.4 组激活检测 detectAndActivateGroups()

**📁 位置**: `SemanticGroupManager.js:274`

```javascript
detectAndActivateGroups(text):
  activatedGroups = new Map()
  
  for (groupName, groupData of groups):
    allWords = [...groupData.words, ...groupData.auto_learned]
    
    matchedWords = allWords.filter(word =>
        flexibleMatch(text, word)
    )
    // flexibleMatch = text.toLowerCase().includes(word.toLowerCase())
    
    if (matchedWords.length > 0):
      activationStrength = matchedWords.length / allWords.length
      activatedGroups.set(groupName, {
          strength:     activationStrength,
          matchedWords: matchedWords,
          allWords:     allWords
      })
      
      // 更新统计
      groups[groupName].last_activated = now
      groups[groupName].activation_count++
  
  return activatedGroups
```

**激活强度示例**:
```
"量子力学" 组 words: ["量子", "纠缠", "叠加", "薛定谔", "不确定性"]
用户输入: "量子纠缠和叠加态之间有什么关系？"
匹配词: ["量子", "纠缠", "叠加"] (3/5)
激活强度: 3/5 = 0.6 (60%)
```

### 1.5 向量增强 getEnhancedVector()

**📁 位置**: `SemanticGroupManager.js:397`

```
getEnhancedVector(originalQuery, activatedGroups, precomputedQueryVector):

  ① 获取查询向量 (优先使用预计算向量)
     queryVector = precomputedQueryVector || getSingleEmbedding(originalQuery)
  
  ② 收集激活组的向量
     vectors = [queryVector (weight=1.0)]
     for (groupName, data of activatedGroups):
       groupVector = groupVectorCache.get(groupName)
       groupWeight = groups[groupName].weight × data.strength
       vectors.push(groupVector, weight=groupWeight)
  
  ③ 加权平均合并
     result = weightedAverageVectors(vectors, weights)
  
  返回增强后的查询向量

// 例子:
// queryVector (weight=1.0): "量子纠缠和叠加态..."
// 量子力学组向量 (weight=1.2 × 0.6 = 0.72): "量子力学相关主题：量子, 纠缠, 叠加, ..."
// → 增强向量偏向物理学方向，提升物理类记忆的召回率
```

### 1.6 向量预计算 precomputeGroupVectors()

**📁 位置**: `SemanticGroupManager.js:322`

```
precomputeGroupVectors():
  for (groupName, groupData of groups):
    allWords = [...words, ...auto_learned]
    currentHash = SHA256(sorted(allWords))
    
    if (hash 变化 || vectorCache 中没有):
      // 构建组描述文本并向量化
      description = "${groupName}相关主题：${allWords.join(', ')}"
      vector = await ragPlugin.getSingleEmbedding(description)
      
      // 清理旧向量文件（如果存在）
      if (groupData.vector_id): fs.unlink(old_vector_file)
      
      // 保存新向量到独立文件
      vectorId = crypto.randomUUID()
      fs.writeFile(semantic_vectors/${vectorId}.json, JSON.stringify(vector))
      
      // 更新内存缓存 + 主配置
      groupVectorCache.set(groupName, vector)
      groupData.vector_id = vectorId
      groupData.words_hash = currentHash
    
  if (changesMade): saveGroups()
```

### 1.7 原子写入保护 / Atomic Write Protection

```javascript
// 写入前先写临时文件，再 rename（原子操作）
saveGroups():
    if (saveLock): throw error  // 防止并发写入
    saveLock = true
    
    tempPath = groupsFilePath + '.' + uuid + '.tmp'
    try:
        fs.writeFile(tempPath, JSON.stringify(data))  // 写临时
        fs.rename(tempPath, groupsFilePath)           // 原子替换
    finally:
        saveLock = false
```

### 1.8 运营指南 / Operations Guide

```bash
# 添加新语义组
# 方法1: 编辑 semantic_groups.edit.json (热更新，无需重启)
# 方法2: 通过 AdminPanel API (POST /api/admin/semantic-groups)

# 查看激活日志
grep "\[SemanticGroup\]" logs.txt
grep "激活的语义组" logs.txt

# 清空向量缓存 (语义组效果异常时)
rm Plugin/RAGDiaryPlugin/semantic_vectors/*.json
rm Plugin/RAGDiaryPlugin/semantic_groups.json
# 然后从 .edit.json 重新初始化
```

---

## 2. AIMemoHandler — AI 驱动记忆召回

**📁 位置**: `Plugin/RAGDiaryPlugin/AIMemoHandler.js`

### 2.1 设计目的 / Design Purpose

AIMemo 让 AI 模型**直接阅读日记文件**进行推理性召回，补充向量搜索无法处理的场景：

```
向量搜索能做到:           AIMemo 额外能做到:
  - 语义相似匹配           - 多条记忆综合推理
  - 模糊概念关联           - "这些记忆有什么规律？"
                           - 跨时间段的演变分析
                           - "你最近在这件事上有什么变化？"
                           - 反事实推理
```

### 2.2 配置参数 / Config

```bash
# config.env (从 AIMemoHandler.loadConfig() 读取)
AIMemoModel=gemini-2.0-flash                  # 用于 AIMemo 的模型
AIMemoBatch=5                                  # 每批处理的日记文件数
AIMemoUrl=https://api.example.com/v1          # API URL
AIMemoApi=your-aimemo-api-key                 # API Key (可与主模型不同)
AIMemoMaxTokensPerBatch=60000                 # 每批最大 token 数
AIMemoPrompt=AIMemoPrompt.txt                 # 提示词模板文件名
```

### 2.3 单库处理 processAIMemo()

```
processAIMemo(dbName, userContent, aiContent):
  
  ① 检查配置完整性 (isConfigured())
     → url + apiKey + model + promptTemplate 全部非空
  
  ② 缓存检查
     cacheKey = MD5(dbName + userContent + aiContent)
     if (cached): return cached.content
  
  ③ 获取日记文件列表
     _getDiaryFiles(dbName):
       → 读取 dailynote/<dbName>/ 目录
       → 按文件修改时间排序（最新优先）
       → 返回 [{ path, content, mtime }]
  
  ④ 分批处理
     → 见 §2.5
  
  ⑤ 合并并格式化结果
  ⑥ 写入缓存
  ⑦ 推送 VCP Info (via ragPlugin.pushVcpInfo)
```

### 2.4 聚合处理 processAIMemoAggregated()

**📁 位置**: `AIMemoHandler.js:63`

当多个日记本一起触发 AIMemo 时：

```
processAIMemoAggregated(dbNames, userContent, aiContent, combinedQueryForDisplay):

  ① 缓存检查: cacheKey = MD5(dbNames.sort() + user + ai)
  
  ② 收集所有日记文件 (基于文件级别)
     allDiaryFiles = []
     for (dbName of dbNames):
       files = _getDiaryFiles(dbName)
       allDiaryFiles.push(...files.map(f => ({ ...f, dbName })))
     
     if (allDiaryFiles.length === 0): return '[没有找到日记内容]'
  
  ③ 对所有文件混合分批 (跨日记本)
     → 见 §2.5
  
  ④ 并行处理所有批次 (Promise.all)
  
  ⑤ 合并 → 格式化 → 缓存 → VCP Info 广播
```

### 2.5 分批策略 / Batching Strategy

```
目标: 将日记文件分成 ≤ maxTokensPerBatch (60000) 的批次

_buildBatches(allDiaryFiles, maxTokensPerBatch):
  batches = []
  currentBatch = []
  currentBatchTokens = 0
  
  for (file of allDiaryFiles):
    fileTokens = estimateTokens(file.content)
    
    if (fileTokens > maxTokensPerBatch):
      // 单文件超限: 强制截断
      file.content = file.content.substring(0, safeLength)
      fileTokens = maxTokensPerBatch × 0.9
    
    if (currentBatchTokens + fileTokens > maxTokensPerBatch && currentBatch.length > 0):
      batches.push(currentBatch)
      currentBatch = [file]
      currentBatchTokens = fileTokens
    else:
      currentBatch.push(file)
      currentBatchTokens += fileTokens
  
  if (currentBatch.length > 0): batches.push(currentBatch)
  return batches
```

**API 调用结构**:
```json
POST AIMemoUrl/v1/chat/completions
{
  "model": "AIMemoModel",
  "messages": [
    {
      "role": "system",
      "content": "<AIMemoPrompt.txt 内容>"
    },
    {
      "role": "user",
      "content": "日记内容:\n<batchContent>\n\n用户问题: <userContent>\nAI最近的回复: <aiContent>"
    }
  ]
}
```

### 2.6 缓存机制 / Cache Mechanism

```javascript
// LRU + TTL 双重控制
aiMemoCache = new Map()                  // LRU Map (最近最少使用)
aiMemoCacheMaxSize = 50                  // 最多 50 条缓存
aiMemoCacheTTL = 1800000                 // 30 分钟

_getCacheKey(dbNames, userContent, aiContent):
  normalized = dbNames.sort().join(',')
  return MD5(normalized + userContent + aiContent)

_getCache(key):
  entry = aiMemoCache.get(key)
  if (entry && Date.now() - entry.timestamp < TTL):
    return entry
  else:
    aiMemoCache.delete(key)
    return null

_setCache(key, content, vcpInfo):
  if (cache.size >= maxSize):
    // 删除最旧的条目 (Map 维护插入顺序)
    firstKey = cache.keys().next().value
    cache.delete(firstKey)
  cache.set(key, { content, vcpInfo, timestamp: Date.now() })
```

---

## 3. 协作关系 / Collaboration

```
RAGDiaryPlugin._processSingleSystemMessage()
  │
  ├── [[日记本::Group]] → SemanticGroupManager.detectAndActivateGroups()
  │                     → SemanticGroupManager.getEnhancedVector()
  │                     → 增强后的查询向量传入 _processRAGPlaceholder()
  │
  ├── [[日记本::AIMemo]] (需 [[AIMemo=True]])
  │   ├── 单库  → AIMemoHandler.processAIMemo()
  │   └── 多库  → AIMemoHandler.processAIMemoAggregated()
  │              → AIMemoHandler._getDiaryFiles()
  │              → AIMemoHandler._buildBatches()
  │              → AI API 调用
  │
  └── [[A|B|C::AIMemo]] → 聚合 AIMemo
                        → AIMemoHandler.processAIMemoAggregated([A, B, C])
```

---

> ⬆️ [README.md](./README.md) | ⬅️ [05_context_vector_manager.md](./05_context_vector_manager.md) | ➡️ [07_meta_thinking_time_parser.md](./07_meta_thinking_time_parser.md)
