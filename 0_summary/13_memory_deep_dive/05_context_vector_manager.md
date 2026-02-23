# 05 ContextVectorManager — 会话上下文向量管理器
# 05 ContextVectorManager — Session Context Vector Manager

> **所属 / Parent**: [README.md](./README.md)  
> **源文件 / Source**: `Plugin/RAGDiaryPlugin/ContextVectorManager.js` (442 行)  
> **⚠️ 注意**: 此组件在之前的文档中完全未记录 / Previously undocumented component

---

## 目录 / Table of Contents

1. [组件职责 / Component Role](#1-组件职责--component-role)
2. [数据结构 / Data Structures](#2-数据结构--data-structures)
3. [核心流程：updateContext()](#3-核心流程updatecontext)
4. [模糊匹配与向量复用 / Fuzzy Match & Vector Reuse](#4-模糊匹配与向量复用--fuzzy-match--vector-reuse)
5. [衰减聚合 aggregateContext()](#5-衰减聚合-aggregatecontext)
6. [语义分段 segmentContext() — TagMemo V4 核心](#6-语义分段-segmentcontext--tagmemo-v4-核心)
7. [指标计算 / Metric Computation](#7-指标计算--metric-computation)
8. [与 processMessages() 的协作时序 / Collaboration Timing](#8-与-processmessages-的协作时序--collaboration-timing)
9. [调试指南 / Debug Guide](#9-调试指南--debug-guide)

---

## 1. 组件职责 / Component Role

ContextVectorManager 是 **TagMemo V4 的关键新组件**，解决了单向量查询无法覆盖多轮对话上下文的问题：

| 能力 / Capability | 对应方法 |
|---|---|
| 维护当前会话所有消息的向量映射 | `updateContext()` |
| 向量查找：精确 → 模糊 → 缓存 → API | `_findFuzzyMatch()`, `updateContext()` |
| 提供衰减加权的历史向量集合 | `aggregateContext()` |
| 语义相似度分段（为 Shotgun Query 提供弹药）| `segmentContext()` |
| 计算历史对话的语义宽度 S | `computeSemanticWidth()` |
| 计算向量的逻辑深度 L | `computeLogicDepth()` |

---

## 2. 数据结构 / Data Structures

```javascript
class ContextVectorManager {
    constructor(plugin) {
        this.plugin = plugin          // RAGDiaryPlugin 实例
        
        // ── 核心向量映射 / Core Vector Map ──
        // normalizedHash → { vector, role, originalText, timestamp }
        this.vectorMap = new Map()
        
        // ── 有序历史向量列表 / Ordered History Vectors ──
        // 按消息顺序排列（updateContext 后填充）
        this.historyAssistantVectors = []   // assistant 角色的历史向量
        this.historyUserVectors = []         // user 角色的历史向量
        
        // ── 超参数 / Hyperparameters ──
        this.fuzzyThreshold    = 0.85    // 模糊匹配阈值 (Dice 系数)
        this.decayRate         = 0.75    // 衰减率 (越早的消息权重越低)
        this.maxContextWindow  = 10      // 聚合窗口：最多看最近 10 楼
    }
}
```

---

## 3. 核心流程：updateContext()

**📁 位置**: `ContextVectorManager.js:93`

在每次 `processMessages()` 运行时调用一次，刷新会话向量映射：

```
updateContext(messages, { allowApi = false }):

  识别最后的 user/assistant 消息索引:
    lastUserIndex  = messages.findLastIndex(m => m.role === 'user')
    lastAiIndex    = messages.findLastIndex(m => m.role === 'assistant')

  并行处理所有消息 Promise.all([]):
    for (msg of messages):
      // 排除: system 消息, 最后一条 user, 最后一条 assistant
      if (role === 'system' || index === lastUserIndex || index === lastAiIndex):
        continue
      
      content = 提取文本内容（支持 string 和 {type:'text'} 格式）
      
      查找策略（4 级降级）:
        ① vectorMap.get(normalizedHash)        精确哈希匹配
        ② _findFuzzyMatch(normalizedText)      Dice 系数 ≥ 0.85 模糊匹配
        ③ plugin._getEmbeddingFromCacheOnly()  只查 Embedding 缓存（不触发 API）
        ④ if (allowApi): 
              plugin.getSingleEmbeddingCached()  触发 API (allowApi=false 时跳过)
      
      if (vector found):
        vectorMap.set(hash, { vector, role, originalText, timestamp })
        push to newAssistantVectors or newUserVectors

  排序 + 赋值:
    historyAssistantVectors = sorted(newAssistantVectors, by index)
    historyUserVectors       = sorted(newUserVectors, by index)
```

**⚠️ 重要**: `allowApi=false` 意味着 `updateContext` **不触发任何 Embedding API 调用**。它只利用 `processMessages` 中已经计算的 query embedding（这条消息在主流程中已经 embedding 过并缓存）。

---

## 4. 模糊匹配与向量复用 / Fuzzy Match & Vector Reuse

**📁 位置**: `ContextVectorManager.js:53-90`

### 文本归一化

```javascript
_normalize(text) {
    let cleaned = plugin._stripHtml(text)
    cleaned = plugin._stripEmoji(cleaned)
    cleaned = plugin._stripToolMarkers(cleaned)  // 移除 VCP 工具调用噪音
    return cleaned.toLowerCase().replace(/\s+/g, ' ').trim()
}
```

### Dice 系数相似度

```javascript
_calculateSimilarity(str1, str2) {
    // Bigram 集合
    const getBigrams = (str) => {
        const bigrams = new Set()
        for (let i = 0; i < str.length - 1; i++):
            bigrams.add(str.substring(i, i + 2))
        return bigrams
    }
    
    const b1 = getBigrams(str1), b2 = getBigrams(str2)
    let intersect = 0
    for (const b of b1): if (b2.has(b)): intersect++
    
    return (2.0 × intersect) / (b1.size + b2.size)
}
```

**模糊匹配使用场景**:
```
AI 回复 (第1次):  "今天天气很好"
AI 回复 (第2次):  "今天天气很好，适合散步"  ← 微小编辑

Dice 系数 ≈ 0.87 > 0.85 → 复用第1次的向量
→ 避免了一次不必要的 Embedding API 调用
```

---

## 5. 衰减聚合 aggregateContext()

**📁 位置**: `ContextVectorManager.js:187`

将历史向量加权聚合为单一向量，近期消息权重更高：

```
aggregateContext(role = 'assistant'):
  vectors = historyAssistantVectors (or historyUserVectors)
  
  ★ 限制窗口: vectors = vectors.slice(-maxContextWindow)
              最多取最近 10 条
  
  for (vector, idx of vectors):
    age    = vectors.length - idx    // 1 = 最近, N = 最早
    weight = decayRate^age           // 0.75^1=0.75, 0.75^2=0.56, ...
    
    aggregated += vector × weight
    totalWeight += weight
  
  return aggregated / totalWeight
```

**衰减权重示意**:
```
消息楼层:  ...  楼N-4  楼N-3  楼N-2  楼N-1  楼N
权重:      ...   0.32   0.42   0.56   0.75   1.00
                                              ↑ 最近消息
```

---

## 6. 语义分段 segmentContext() — TagMemo V4 核心

**📁 位置**: `ContextVectorManager.js:297`

这是 TagMemo V4 Shotgun Query 的核心输入，将对话历史分割为**主题连续的语义段**：

```
segmentContext(messages, similarityThreshold = 0.70):

  ① 重建有序序列 (按 message index):
     for (msg of messages):
       if (role === 'system'): skip
       normalized = _normalize(content)
       hash = SHA256(normalized)
       entry = vectorMap.get(hash)  ← 精确查找
       if (entry.vector):
         sequence.push({ index, role, text, vector })
  
  ② 执行滑动分段:
     currentSegment = { vectors: [seq[0]], texts, startIndex, endIndex, roles }
     
     for (i = 1; i < sequence.length; i++):
       sim = cosineSimilarity(seq[i-1].vector, seq[i].vector)
       
       if (sim >= 0.70):         ← 语义相似 → 合并
         currentSegment.vectors.push(seq[i].vector)
         currentSegment.endIndex = seq[i].index
       else:                     ← 语义断裂 → 新段
         segments.push(_finalizeSegment(currentSegment))
         currentSegment = new segment starting at seq[i]
     
     segments.push(_finalizeSegment(currentSegment))  ← 最后一段
  
  ③ _finalizeSegment(seg):
     avgVec = mean(seg.vectors)      // 计算段平均向量
     avgVec = normalize(avgVec)      // L2 归一化
     return {
         vector: avgVec,             // 该段的代表向量 → 用于 Shotgun
         text:   seg.texts.join('\n'),
         roles:  unique(seg.roles),  // ['user', 'assistant'] 等
         range:  [startIndex, endIndex],
         count:  seg.vectors.length
     }

返回: segments (按时序排列，每段代表一个对话主题)
```

**分段示例**:
```
对话历史:
  [楼1] 用户: 今天学了量子纠缠                     ─┐
  [楼2] AI:  量子纠缠是个很有意思的现象...           │ segment 1 (物理学)
  [楼3] 用户: 能用例子解释吗？                       ─┘
  [楼4] 用户: 对了，上周那个 Python 代码            ─┐
  [楼5] AI:  好的，我来看看代码...                   │ segment 2 (编程)
  [楼6] 用户: 能优化这个函数吗？                     ─┘
  [楼7] 用户: 量子计算跟量子纠缠有什么关系？         ─┐ segment 3 (新物理查询)

Shotgun Query 使用:
  current  = 楼7 的向量 (新物理学查询)
  history_0 = segment 1 的均值向量 (量子纠缠段)
  history_1 = segment 2 的均值向量 (Python 段)
→ 三个向量并行搜索日记本，全面覆盖上下文主题
```

---

## 7. 指标计算 / Metric Computation

### 7.1 逻辑深度 computeLogicDepth()

**📁 位置**: `ContextVectorManager.js:229`

```javascript
computeLogicDepth(vector, topK = 64) {
    // 计算前 K 个维度的能量占比
    // 能量集中 → L 高 (意图聚焦)
    // 能量分散 → L 低 (意图模糊)
    
    energies = vector.map(v => v^2)
    totalEnergy = sum(energies)
    
    sorted = sort(energies, desc)
    topKEnergy = sum(sorted.slice(0, topK))
    
    concentration = topKEnergy / totalEnergy
    expectedUniform = topK / dim              // 均匀分布下的期望值
    
    L = (concentration - expectedUniform) / (1 - expectedUniform)
    return clamp(L, 0, 1)
}
```

**注意**: 这个 `computeLogicDepth` 是一个简化版本（不依赖 EPA 正交基）。完整版本在 `EPAModule.project()` 中。`_calculateDynamicParams` 实际上使用了 EPA 的版本。

### 7.2 语义宽度 computeSemanticWidth()

**📁 位置**: `ContextVectorManager.js:260`

```javascript
computeSemanticWidth(vector) {
    // 向量模长反映语义确定性
    // 归一化后模长应≈1，但未归一化的向量模长差异可以用来衡量"宽度"
    magnitude = ||vector||
    return magnitude × spreadFactor  // spreadFactor = 1.2
}
```

**与 `_calculateDynamicParams` 的对应关系**:
```javascript
// _calculateDynamicParams 中:
const S = this.contextVectorManager.computeSemanticWidth(queryVector)
// S 用于 Beta (TagWeight) 计算:
// β_input = L × ln(2+R) - S × noise_penalty
// S 越大 → β 越小 → 减少 Tag 引力噪音
```

---

## 8. 与 processMessages() 的协作时序 / Collaboration Timing

```
processMessages(messages):
  │
  ├─ Step A: getSingleEmbeddingCached(combinedQuery)
  │          ↑ 这个向量被缓存到 embeddingCache
  │
  ├─ Step B: contextVectorManager.updateContext(messages, {allowApi: false})
  │          ↑ 利用 Step A 的缓存为历史消息找向量（不新建 API 调用）
  │          ↑ 更新 historyAssistantVectors 和 historyUserVectors
  │
  ├─ Step C: historySegments = contextVectorManager.segmentContext(messages)
  │          ↑ 基于 Step B 的 vectorMap 生成语义分段
  │
  ├─ Step D: _calculateDynamicParams(queryVector, ...)
  │          ↑ 调用 contextVectorManager.computeSemanticWidth(queryVector)
  │          ↑ 需要 Step A 的向量
  │
  └─ Step E: _processRAGPlaceholder({ historySegments, ... })
             ↑ 使用 Step C 的 historySegments 构建 Shotgun Query
```

---

## 9. 调试指南 / Debug Guide

```bash
# 上下文向量更新成功
grep "\[ContextVectorManager\] 上下文向量映射已更新" logs.txt
# 输出: 历史AI向量: 5, 历史用户向量: 4

# 语义分段结果 (通过 RAGDiaryPlugin 日志)
grep "Tagmemo V4: Detected" logs.txt
# 输出: [RAGDiaryPlugin] Tagmemo V4: Detected 3 history segments.

# Shotgun Query 并行搜索
grep "Shotgun Query: Executing" logs.txt
# 输出: Shotgun Query: Executing 4 parallel searches...

# 常见问题:
# - historySegments 为空: vectorMap 中没有历史向量
#   原因: 对话轮次太少 (<2轮) 或消息太短 (<2字符)
# - 所有消息归为一个 segment: 对话主题非常一致（正常情况）
```

---

> ⬆️ [README.md](./README.md) | ⬅️ [04_embedding_chunker.md](./04_embedding_chunker.md) | ➡️ [06_semantic_group_aimemo.md](./06_semantic_group_aimemo.md)
