# 07 MetaThinkingManager + TimeExpressionParser — 元思考与时间解析
# 07 MetaThinkingManager + TimeExpressionParser — Meta-Thinking & Time Parsing

> **所属 / Parent**: [README.md](./README.md)  
> **源文件 / Sources**:  
> - `Plugin/RAGDiaryPlugin/MetaThinkingManager.js` (396 行)  
> - `Plugin/RAGDiaryPlugin/TimeExpressionParser.js` (213 行)  
> - `Plugin/RAGDiaryPlugin/timeExpressions.config.js` (配置文件)

---

## 目录 / Table of Contents

1. [MetaThinkingManager — 元思考递归推理链](#1-metathinkingmanager--元思考递归推理链)
   - 1.1 [设计目的 / Design Purpose](#11-设计目的--design-purpose)
   - 1.2 [配置文件结构 / Config File Structure](#12-配置文件结构--config-file-structure)
   - 1.3 [主题向量缓存 / Theme Vector Cache](#13-主题向量缓存--theme-vector-cache)
   - 1.4 [自动主题检测 / Auto Theme Detection](#14-自动主题检测--auto-theme-detection)
   - 1.5 [递归推理链执行 / Recursive Reasoning Chain](#15-递归推理链执行--recursive-reasoning-chain)
   - 1.6 [向量融合策略 / Vector Fusion Strategy](#16-向量融合策略--vector-fusion-strategy)
   - 1.7 [结果格式化 / Result Formatting](#17-结果格式化--result-formatting)
2. [TimeExpressionParser — 时间表达式解析器](#2-timeexpressionparser--时间表达式解析器)
   - 2.1 [设计目的 / Design Purpose](#21-设计目的--design-purpose)
   - 2.2 [解析策略 / Parsing Strategy](#22-解析策略--parsing-strategy)
   - 2.3 [硬编码表达式 / Hardcoded Expressions](#23-硬编码表达式--hardcoded-expressions)
   - 2.4 [动态模式匹配 / Dynamic Pattern Matching](#24-动态模式匹配--dynamic-pattern-matching)
   - 2.5 [寒暄语触发设计 / Greeting Trigger Design](#25-寒暄语触发设计--greeting-trigger-design)
   - 2.6 [输出格式 / Output Format](#26-输出格式--output-format)
3. [与 RAGDiaryPlugin 的集成点 / Integration Points](#3-与-ragdiaryPlugin-的集成点--integration-points)

---

## 1. MetaThinkingManager — 元思考递归推理链

**📁 位置**: `Plugin/RAGDiaryPlugin/MetaThinkingManager.js`

### 1.1 设计目的 / Design Purpose

元思考让 AI 能够对记忆进行**多层次抽象推理**，而不仅是平铺直叙的检索：

```
普通 RAG 检索:             元思考链:
  问: "我学了什么数学？"     
  → 找出所有数学相关记忆    step1: 搜索"基础知识" → 找到记忆碎片 → 更新查询向量
                            step2: 搜索"方法论" → 在更聚焦的方向上搜索
                            step3: 搜索"元认知" → 在更高层上进行抽象
                            → 三阶段结果的有机融合
```

### 1.2 配置文件结构 / Config File Structure

**📁 位置**: `Plugin/RAGDiaryPlugin/meta_thinking_chains.json`

```json
{
  "chains": {
    "default": {
      "kSequence": [3, 3, 3],
      "clusters": ["小可", "知识库", "感悟簇"]
      // ⚠️ "default" 是自动模式的回退链，不参与主题相似度竞争
    },
    "技术思考": {
      "kSequence": [5, 3, 2],
      "clusters": ["代码日记", "技术笔记", "经验总结"]
    },
    "情感反思": {
      "kSequence": [4, 4, 2],
      "clusters": ["日记本", "情感记录", "成长感悟"]
    }
  }
}
```

**字段说明**:

| 字段 | 说明 |
|---|---|
| `chains` | 所有推理链定义 |
| `default` | 自动模式下的回退链 (不作为自动切换目标) |
| `kSequence` | 每阶段的 K 值 (长度必须等于 clusters 长度) |
| `clusters` | 每阶段搜索的日记本名称 (按顺序) |

### 1.3 主题向量缓存 / Theme Vector Cache

**📁 位置**: `MetaThinkingManager.js:_buildAndSaveMetaChainThemeCache()`

```
初始化时构建主题向量缓存:

对每个推理链名称 (跳过 'default'):
  themeVector = await ragPlugin.getSingleEmbedding(chainName)
  // 例: getSingleEmbedding("技术思考")
  metaChainThemeVectors[chainName] = themeVector

保存到磁盘:
  meta_chain_vector_cache.json = {
    sourceHash: MD5(meta_thinking_chains.json),
    vectors: { "技术思考": [...], "情感反思": [...] }
  }

缓存失效检测:
  hash = MD5(meta_thinking_chains.json 文件内容)
  if (hash != cache.sourceHash): 重建缓存
```

### 1.4 自动主题检测 / Auto Theme Detection

当占位符为 `[[VCP元思考:Auto]]` 时触发自动模式：

```
isAutoMode = true

for (themeName, themeVector of metaChainThemeVectors):
  similarity = cosineSimilarity(queryVector, themeVector)
  if (similarity > maxSimilarity):
    maxSimilarity = similarity
    bestChain = themeName

if (maxSimilarity >= autoThreshold (=0.65)):
  finalChainName = bestChain  ← 切换到最匹配的主题
else:
  finalChainName = 'default'  ← 相似度不足，使用默认链

// 日志输出
[MetaThinkingManager][Auto] 最匹配的主题是 "技术思考"，相似度: 0.7823
[MetaThinkingManager][Auto] 相似度超过阈值 0.65，切换到主题: 技术思考
```

### 1.5 递归推理链执行 / Recursive Reasoning Chain

**📁 位置**: `MetaThinkingManager.js:processMetaThinkingChain()`

```
执行 3 阶段递归推理:

初始化:
  currentQueryVector = queryVector   ← 从原始查询开始

for (i = 0; i < chain.length; i++):
  clusterName = chain[i]              // 当前阶段的日记本
  k = kSequence[i]                    // 当前阶段的 K 值
  
  ① 搜索当前阶段:
     searchResults = vectorDBManager.search(clusterName, currentQueryVector, k)
     // 注意: 使用上一阶段融合后的向量，不是原始 queryVector
     
     if (no results): 
       mark as degraded
       continue  ← 保持 currentQueryVector 不变
  
  ② 向量融合 (为下一阶段准备):
     if (i < chain.length - 1):  ← 不是最后一阶段
       resultVectors = 从 searchResults 中提取向量
       avgResultVector = mean(resultVectors)
       currentQueryVector = weightedAverage(
           [queryVector, avgResultVector],
           [0.8, 0.2]                   ← 保持原始查询权重更高
       )

// 关键: 每阶段搜索都基于上一阶段结果的"语义导向"
// 阶段1: 搜索基础知识 → 找到量子基础
// 阶段2: queryVector 向量偏移到"量子基础"方向 → 搜索方法论
//        发现了量子力学的数学基础...
// 阶段3: queryVector 再次偏移 → 搜索元认知层次
//        达到更高层次的抽象
```

### 1.6 向量融合策略 / Vector Fusion Strategy

```javascript
// 融合公式
currentQueryVector = weightedAverage(
    [originalQueryVector, avgResultVector],
    [0.8, 0.2]
)
// 原始查询权重 0.8 >> 结果向量权重 0.2
// 设计意图: 保持对原始用户意图的"锚点"，同时让搜索方向受上一阶段结果的轻微引导
// 如果 0.2 权重太大，可能导致"语义漂移"
```

**向量融合数学表达**:
```
q_next = normalize(0.8 × q_original + 0.2 × mean(results_i))
```

### 1.7 结果格式化 / Result Formatting

```
_formatMetaThinkingResults(chainResults, chainName, activatedGroups):

输出格式:
  <!-- VCP_META_THINKING_START {"chainName":"技术思考","stages":3} -->
  [--- VCP元思考链: 技术思考 ---]
  
  [第1阶段: 代码日记 (K=5, 找到 4 条)]
  * [记忆片段1]
  * [记忆片段2]
  ...
  
  [第2阶段: 技术笔记 (K=3, 找到 3 条)]
  * [记忆片段]
  ...
  
  [第3阶段: 经验总结 (K=2, 找到 2 条)]
  * [最终抽象记忆]
  
  [--- 元思考链结束 ---]
  <!-- VCP_META_THINKING_END -->
```

---

## 2. TimeExpressionParser — 时间表达式解析器

**📁 位置**: `Plugin/RAGDiaryPlugin/TimeExpressionParser.js`

### 2.1 设计目的 / Design Purpose

```
用户说 "今天发生了什么？" → 应该优先检索今天的日记
用户说 "上周的学习情况" → 应该检索上周一到周日的日记
用户说 "你好" → 触发近期记忆（1天内）
```

TimeExpressionParser 将这些**自然语言时间表达式**解析为 `{ start: Date, end: Date }` 的时间范围，供 RAGDiaryPlugin 进行时间过滤。

### 2.2 解析策略 / Parsing Strategy

**📁 位置**: `TimeExpressionParser.js:parse()`

```
parse(text):
  results = []
  remainingText = text  ← 消费模式（避免重复匹配）
  
  Step 1: 硬编码表达式匹配（从长到短排序，防止短词先匹配）
    for (expr of sortedHardcodedKeys):
      if (text.includes(expr)):
        result = _getDayBoundaries(now - config.days)
        results.push(result)
        remainingText = remainingText.replace(expr, '')
  
  Step 2: 动态正则模式匹配
    for (pattern of expressions.patterns):
      globalRegex = new RegExp(pattern.regex.source, 'g')
      while ((match = globalRegex.exec(remainingText))):
        result = _handleDynamicPattern(match, pattern.type, now)
        results.push(result)
  
  return results (时间范围数组，每个匹配一个)
```

### 2.3 硬编码表达式 / Hardcoded Expressions

来源: `timeExpressions.config.js`

| 表达式 | 类型 | 时间范围 |
|---|---|---|
| `今天` | days=0 | 今日 00:00 ~ 23:59 |
| `昨天` | days=1 | 昨日 00:00 ~ 23:59 |
| `前天` | days=2 | 前日 00:00 ~ 23:59 |
| `最近` | days=5 | 5天前 ~ 今日 |
| `上周` | type=lastWeek | 上周一 ~ 上周日 |
| `上个月` | type=lastMonth | 上月1日 ~ 上月末日 |
| `本周` / `这周` | type=thisWeek | 本周一 ~ 今日 |
| `本月` / `这个月` | type=thisMonth | 本月1日 ~ 今日 |
| `在吗` / `你好` | days=1 | 昨日 ~ 今日 (寒暄触发) |

### 2.4 动态模式匹配 / Dynamic Pattern Matching

| 正则模式 | 示例匹配 | 含义 |
|---|---|---|
| `/(\d+\|[一二...十])天前/` | "3天前", "三天前" | N天前 |
| `/上周([一二三四五六日天])/` | "上周三" | 上周特定星期 |
| `/(\d+\|[一...十])周前/` | "2周前" | N周前 |
| `/(\d+\|[一...十])个月前/` | "3个月前" | N月前 |

**汉字数字转换**:
```javascript
// 支持 "三天前" → 3天前
const chineseNums = { '一': 1, '二': 2, '三': 3, ... '十': 10 }
const num = isNaN(match) ? chineseNums[match] : parseInt(match)
```

### 2.5 寒暄语触发设计 / Greeting Trigger Design

```javascript
// 特殊设计: 打招呼时触发近期记忆（1天范围）
'在吗':    { days: 1 },
'在不':    { days: 1 },
'你好':    { days: 1 },
'哈喽':    { days: 1 },
'早上好':  { days: 0 }  // 早上好 → 今日

// 设计意图:
// 当用户只是打个招呼，VCP 会用最近的记忆来启动对话
// 让 AI 知道"昨天或今天发生了什么"，保持对话的连续性感
```

### 2.6 输出格式 / Output Format

```javascript
// _getDayBoundaries(date) 返回
{
    start: Date,  // 当天 00:00:00.000 (配置时区)
    end:   Date   // 当天 23:59:59.999 (配置时区)
}

// 时区处理 (dayjs)
dayjs(date).tz(defaultTimezone).startOf('day')  // Asia/Shanghai 等

// RAGDiaryPlugin 使用:
timeRanges = timeParser.parse(userContent)
// [{ start: Date, end: Date }, ...]
// 可以有多个时间范围 (如 "今天和上周的记忆")
```

---

## 3. 与 RAGDiaryPlugin 的集成点 / Integration Points

```
RAGDiaryPlugin.processMessages():
  │
  ├── timeRanges = timeParser.extractTimeRanges(userContent)
  │   ↑ 第一步: 从用户输入提取时间意图
  │
  └── 传入 _processSingleSystemMessage()
      │
      ├── [[日记本::Time]] → _processRAGPlaceholder({ useTime: true, timeRanges })
      │   ↑ 时间感知双路召回 (见 02_rag_diary_plugin.md §8)
      │
      └── [[VCP元思考:技术思考]] → metaThinkingManager.processMetaThinkingChain()
          ↑ 递归推理链 (见本文档 §1.5)
```

---

> ⬆️ [README.md](./README.md) | ⬅️ [06_semantic_group_aimemo.md](./06_semantic_group_aimemo.md) | ➡️ [08_write_plugins.md](./08_write_plugins.md)
