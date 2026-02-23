# 08 写入插件详解 — DailyNoteWrite, LightMemo, ThoughtCluster, AgentDream
# 08 Write Plugins — DailyNoteWrite, LightMemo, ThoughtCluster, AgentDream

> **所属 / Parent**: [README.md](./README.md)  
> **源文件 / Sources**:  
> - `Plugin/DailyNoteWrite/daily-note-write.js` (416 行)  
> - `Plugin/LightMemo/LightMemo.js` (688 行)  
> - `Plugin/ThoughtClusterManager/ThoughtClusterManager.js` (约 200 行)  
> - `Plugin/AgentDream/AgentDream.js` (约 500 行)

---

## 目录 / Table of Contents

1. [DailyNoteWrite — 主力日记写入插件](#1-dailynotewrite--主力日记写入插件)
   - 1.1 [调用协议 / Invocation Protocol](#11-调用协议--invocation-protocol)
   - 1.2 [Tag 处理管道 / Tag Processing Pipeline](#12-tag-处理管道--tag-processing-pipeline)
   - 1.3 [文件写入细节 / File Write Details](#13-文件写入细节--file-write-details)
   - 1.4 [文件格式 / File Format](#14-文件格式--file-format)
2. [LightMemo — BM25+向量混合检索与快速记录](#2-lightmemo--bm25向量混合检索与快速记录)
   - 2.1 [BM25 算法实现 / BM25 Algorithm](#21-bm25-算法实现--bm25-algorithm)
   - 2.2 [三阶段检索流程 / Three-Stage Retrieval](#22-三阶段检索流程--three-stage-retrieval)
   - 2.3 [Jieba 中文分词 / Jieba Tokenization](#23-jieba-中文分词--jieba-tokenization)
   - 2.4 [语义组 Token 扩展 / Semantic Group Token Expansion](#24-语义组-token-扩展--semantic-group-token-expansion)
   - 2.5 [混合评分策略 / Hybrid Scoring](#25-混合评分策略--hybrid-scoring)
3. [ThoughtClusterManager — 思维簇主题记忆](#3-thoughtclustermanager--思维簇主题记忆)
4. [AgentDream — 梦境记忆整合引擎](#4-agentdream--梦境记忆整合引擎)
   - 4.1 [梦境触发机制 / Dream Trigger](#41-梦境触发机制--dream-trigger)
   - 4.2 [记忆采集与联想 / Memory Seeding & Association](#42-记忆采集与联想--memory-seeding--association)
   - 4.3 [梦境操作类型 / Dream Operations](#43-梦境操作类型--dream-operations)
   - 4.4 [审批门 / Approval Gate](#44-审批门--approval-gate)
   - 4.5 [存储结构 / Storage Structure](#45-存储结构--storage-structure)
5. [四种写入插件对比 / Comparison](#5-四种写入插件对比--comparison)

---

## 1. DailyNoteWrite — 主力日记写入插件

**📁 位置**: `Plugin/DailyNoteWrite/daily-note-write.js`  
**插件类型**: `asynchronous` (子进程执行，stdin/stdout 通信)

### 1.1 调用协议 / Invocation Protocol

AI 通过 VCP 工具协议触发，数据通过 stdin 传入（JSON 格式）：

```javascript
// Plugin.js 调用方式
const child = spawn('node', ['daily-note-write.js'], { cwd: pluginDir })
child.stdin.write(JSON.stringify({ maidName, dateString, contentText }))
child.stdin.end()

// 返回 (stdout)
{ "status": "success", "message": "Diary saved to /path/to/file.md" }
// 或
{ "status": "error", "message": "..." }
```

### 1.2 Tag 处理管道 / Tag Processing Pipeline

**📁 位置**: `daily-note-write.js:290` — `processTagsInContent()`

```
processTagsInContent(contentText):

  ① detectTagLine(contentText):
     检查最后一行是否匹配 /^Tag:\s*.+/i
     返回: { hasTag, lastLine, contentWithoutLastLine }

  ② 路径 A: 已有 Tag 行:
     fixedTag = fixTagFormat(detection.lastLine)
     // fixTagFormat 规范化:
     //   - 统一前缀: "Tag: "（大小写规范）
     //   - 替换中文标点: ，→ ", "  、→ ", "  ：→ ""
     //   - 规范化空格
     processedContent = contentWithoutLastLine + '\n' + fixedTag

  ③ 路径 B: 没有 Tag 行:
     generatedTag = await generateTagsWithAI(contentText)
     // AI 生成 Tag (最多 3 次重试)
     
     if (generatedTag):
       extractedTag = extractTagFromAIResponse(generatedTag)
       // 提取 [[Tag: xxx]] 格式
       fixedTag = fixTagFormat(extractedTag)
       processedContent = contentText + '\n' + fixedTag
     else:
       processedContent = contentText  // 无 Tag，原样保留
```

**`generateTagsWithAI()` 调用流程**:
```javascript
// 1. 读取 TagMaster 提示词文件 (TagMasterPrompt.txt)
// 2. POST ${API_URL}/v1/chat/completions
//    {
//      model: TAG_MODEL,
//      messages: [
//        { role: "system", content: tagMasterPrompt },
//        { role: "user", content: "请为以下日记内容生成Tag:\n" + content }
//      ]
//    }
// 3. 从响应中提取 [[Tag: ...]] 格式
// 4. 指数退避重试 (1s, 2s, 4s)
```

### 1.3 文件写入细节 / File Write Details

**📁 位置**: `daily-note-write.js:323` — `writeDiary()`

```
writeDiary(maidName, dateString, contentText):

  ① 处理 [tag]name 格式的 maidName:
     "[知识库]量子力学笔记" →
       folderName = "知识库"    (作为目录名)
       actualMaidName = "量子力学笔记" (作为文件头)
     
     无中括号:
       folderName = maidName
       actualMaidName = maidName

  ② 路径规范化 (sanitizePathComponent):
     移除 \/:*?"<>| 等非法字符
     移除控制字符 \x00-\x1f
     Trim 首尾空格和点号
     空结果 → "Untitled"

  ③ 构建文件路径:
     datePart = dateString (替换 .\/\s- 为 -)
     timeStr = HH_MM_SS (防冲突时间戳)
     fileName = "{datePart}-{timeStr}.md"
     filePath = dailynote/{sanitizedFolderName}/{fileName}

  ④ 创建目录 + 写入:
     fs.mkdir(dirPath, { recursive: true })
     fileContent = "[{datePart}] - {actualMaidName}\n{processedContent}"
     fs.writeFile(filePath, fileContent)
```

### 1.4 文件格式 / File Format

```markdown
[2026-02-23] - 小可
今天在图书馆看了量子力学导论...
量子纠缠真的很奇妙，两个粒子之间的神秘联系...

[[更多感想]]

Tag: 量子力学, 物理, 学习, 图书馆
```

**关键设计**:
- 首行格式 `[YYYY-MM-DD] - 角色名` 是 RAGDiaryPlugin 中的标准头格式
- Tag 行必须在最后一行
- 文件名包含时间戳，同一天可以写多篇日记

---

## 2. LightMemo — BM25+向量混合检索与快速记录

**📁 位置**: `Plugin/LightMemo/LightMemo.js`  
**插件类型**: `synchronous` (在服务进程内直接执行)

### 2.1 BM25 算法实现 / BM25 Algorithm

**📁 位置**: `LightMemo.js:9` — `class BM25Ranker`

```javascript
// BM25 (Best Match 25) — 信息检索领域的经典算法
// 参数: k1=1.5 (词频饱和因子), b=0.75 (文档长度归一化因子)

calculateIDF(allDocTokens):
  for (term in vocabulary):
    docFreq = 含有 term 的文档数
    idf[term] = ln((N - docFreq + 0.5) / (docFreq + 0.5) + 1)
    // N = 总文档数

score(queryTokens, docTokens, avgDocLength, idfScores):
  bm25 = 0
  for (term of queryTokens):
    if (term in docTokens):
      tf = 词频 (term 在 doc 中出现次数)
      docLen = docTokens.length
      // BM25 公式:
      bm25 += idf[term] × (tf × (k1 + 1)) / 
              (tf + k1 × (1 - b + b × docLen / avgDocLen))
  return bm25
```

### 2.2 三阶段检索流程 / Three-Stage Retrieval

**📁 位置**: `LightMemo.js:142` — `handleSearch()`

```
Stage 1: BM25 关键词初筛
  queryTokens = jieba.cut(query)       中文分词
  expandedTokens = _expandQueryTokens(queryTokens)  语义组扩展
  candidates = _gatherCandidateChunks(maid/folder)   收集所有 chunk
  BM25 打分 → 过滤 score>0 → 取前 k×5 个
  
  如果 BM25 结果 < k:
    补充向量相似度 Top-k×2 作为兜底候选

Stage 2: 向量精排
  queryVector = getSingleEmbedding(query)
  
  if (tag_boost > 0):   ← 可选 TagMemo V3 增强
    queryVector = vectorDBManager.applyTagBoost(queryVector, tag_boost)
  
  for (candidate of topByKeyword):
    vectorScore = cosineSimilarity(candidate.chunkVector, queryVector)
  
  混合评分:
    hasBM25 = candidate.bm25Score > 0
    hybridScore = normalizedBM25 × 0.6 + vectorScore × 0.4   (有 BM25)
    hybridScore = vectorScore × 1.0                            (纯向量兜底)

Stage 3: (可选) Rerank
  if (rerank=true && rerankConfig 已配置):
    → 外部 Rerank API 二次排序 (同 RAGDiaryPlugin._rerankDocuments)
```

### 2.3 Jieba 中文分词 / Jieba Tokenization

```javascript
// 使用 @node-rs/jieba (Rust 绑定，速度快)
this.jiebaInstance = Jieba.withDict(dict)

_tokenize(text):
  words = this.jiebaInstance.cut(text, false)  // 精确模式
  
  // 过滤停用词:
  stopWords = new Set(['的', '了', '是', '在', '我', '有', '和', '就', ...])
  
  // 保留有意义的词:
  return words.filter(w => 
    w.length >= 2 &&           // 最少 2 字符（过滤单字）
    !stopWords.has(w) &&
    !/^\s+$/.test(w)           // 过滤纯空格
  )
```

### 2.4 语义组 Token 扩展 / Semantic Group Token Expansion

**📁 位置**: `LightMemo.js:_expandQueryTokens()`

```javascript
_expandQueryTokens(tokens) {
    const expandedTokens = new Set()
    
    // 对每个 token 检查是否匹配语义组
    for (const token of tokens) {
        for (const [groupName, groupData] of semanticGroupManager.groups) {
            const allWords = [...groupData.words, ...(groupData.auto_learned || [])]
            if (allWords.some(w => w.toLowerCase().includes(token.toLowerCase()))) {
                // 命中语义组，添加该组所有词作为扩展
                allWords.forEach(w => expandedTokens.add(w))
            }
        }
    }
    return Array.from(expandedTokens)
}
```

### 2.5 混合评分策略 / Hybrid Scoring

```
候选 chunk 的最终评分:

动态权重:
  有 BM25 分数 (关键词匹配): BM25权重=0.6, 向量权重=0.4
  无 BM25 分数 (纯向量兜底): BM25权重=0.0, 向量权重=1.0

混合分 = normalizedBM25 × bmWeight + vectorScore × vecWeight
       (normalizedBM25 = min(1.0, bm25Score / 10))

排序输出格式:
  📍 [2026-02-20 - 日记名]
  内容片段... (最多 300 字)
  🏷️ Tags: 量子力学, 物理
  📊 相关度: 0.85 [BM25+向量] / 0.73 [向量] / ...
```

---

## 3. ThoughtClusterManager — 思维簇主题记忆

**📁 位置**: `Plugin/ThoughtClusterManager/ThoughtClusterManager.js`  
**插件类型**: `synchronous`

### API 参考

```javascript
// 工具调用 1: 创建思维簇新条目
CreateClusterFile({
    clusterName: "量子力学簇",    // 必须以 '簇' 结尾
    content: "今天学到了..."      // Markdown 内容 (自动添加 Tag)
})
// 写入: dailynote/量子力学簇/<timestamp>.md

// 工具调用 2: 编辑已有条目
EditClusterFile({
    clusterName: "量子力学簇",
    targetText:  "原始文本内容",    // 精确匹配（前 80 字符定位）
    replacementText: "替换后的内容"
})

// 工具调用 3: 读取簇内容
ReadClusterFile({
    clusterName: "量子力学簇",
    maxEntries: 10,               // 最近 N 条
    searchQuery: "薛定谔"         // 可选：关键词过滤
})
```

**命名规范**: 目录名必须以 `簇` 结尾，这是系统识别思维簇目录的标志。

**存储结构**:
```
dailynote/
└── 量子力学簇/
    ├── 2026-02-23T14-30-00-000Z.md   ← ISO 时间戳文件名
    └── 2026-02-20T09-15-00-000Z.md
```

---

## 4. AgentDream — 梦境记忆整合引擎

**📁 位置**: `Plugin/AgentDream/AgentDream.js`  
**📁 详细文档**: `AgentDream.md`  
**插件类型**: `service` (后台服务)

### 4.1 梦境触发机制 / Dream Trigger

```javascript
// AgentDream 通过以下方式触发:
// 方式 1: 定时触发 (cron-style)
schedule = AgentDreamSchedule || '0 3 * * *'  // 默认: 每天凌晨3点

// 方式 2: 手动触发 (AdminPanel API)
POST /api/admin/agent-dream/trigger

// 方式 3: 对话中通过 VCP 工具调用触发
// (仅 AgentDream 启用时)
```

### 4.2 记忆采集与联想 / Memory Seeding & Association

```
梦境流程:

① 记忆种子采集 (Memory Seeding)
  randomSeedFile = 从 dailynote/ 随机选取 1 个 .md 文件
  seedContent = readFile(randomSeedFile)
  seedVector = getSingleEmbedding(seedContent)

② 语义联想 (Semantic Association)
  relatedMemories = vectorDBManager.search(
    diaryName: null,     // 全局搜索所有日记本
    queryVector: seedVector,
    k: 5
  )
  // 找到 5 条语义相关的记忆片段

③ 提示词构建
  dreamPrompt = renderTemplate('dreampost.txt', {
    memorySeeds: seedContent,
    relatedMemories: formatResults(relatedMemories),
    characterPersonality: agentConfig.personality,
    dreamInstructions: "请生成以下格式的梦境感悟..."
  })

④ AI 生成梦境叙事
  POST LLM API { system: dreamerPersona, user: dreamPrompt }
  dreamNarrative = response.content
  
  ← 梦境叙事包含意识流叙事 + 嵌入的操作指令
```

### 4.3 梦境操作类型 / Dream Operations

AI 在梦境叙事中使用 VCP 工具调用语法嵌入操作指令：

```
DiaryMerge (合并相似日记):
  <<<[TOOL_REQUEST]>>>
  tool_name:「始」DiaryMerge「末」,
  sourceDiaries:「始」2026-02-10.md, 2026-02-11.md「末」,
  targetDiary:「始」2026-02-10-merged.md「末」,
  mergedContent:「始」合并后的内容「末」
  <<<[END_TOOL_REQUEST]>>>

DiaryDelete (删除过时/错误记忆):
  <<<[TOOL_REQUEST]>>>
  tool_name:「始」DiaryDelete「末」,
  targetDiary:「始」2026-01-01.md「末」,
  reason:「始」内容已过时，已被更新的记忆取代「末」
  <<<[END_TOOL_REQUEST]>>>

DreamInsight (创造新感悟日记):
  <<<[TOOL_REQUEST]>>>
  tool_name:「始」DailyNoteWrite「末」,
  maidName:「始」小可「末」,
  dateString:「始」2026-02-23「末」,
  contentText:「始」梦境感悟：从量子力学的角度看...
  Tag: 梦境感悟, 量子力学, 元认知「末」
  <<<[END_TOOL_REQUEST]>>>
```

### 4.4 审批门 / Approval Gate

```
梦境操作的执行需要人工审批:

① 生成 JSON 操作日志:
   dream_logs/<timestamp>.json = {
     dreamId: "uuid",
     createdAt: "ISO8601",
     narrative: "梦境叙事全文",
     operations: [
       { type: "DiaryMerge", sources: [...], preview: "..." },
       { type: "DiaryDelete", target: "...", reason: "..." },
       { type: "DreamInsight", content: "...", tags: [...] }
     ],
     status: "pending"  ← 等待审批
   }

② AdminPanel 展示等待审批的操作列表
   GET /api/admin/dream-logs?status=pending

③ 管理员确认/拒绝
   PUT /api/admin/dream-logs/:dreamId/approve
   → 执行实际操作 (文件合并/删除/写入)
   → 触发 KnowledgeBaseManager 重建索引
   
   PUT /api/admin/dream-logs/:dreamId/reject
   → 标记为 rejected，不执行任何操作
```

### 4.5 存储结构 / Storage Structure

```
dream_logs/
├── 2026-02-23T03-00-00-000Z.json   ← 等待审批的操作
└── 2026-02-20T03-00-00-000Z.json   ← 已执行/已拒绝

dailynote/
└── 小可/
    └── 2026-02-23-03_05_00.md      ← 审批通过后写入的梦境感悟
```

---

## 5. 四种写入插件对比 / Comparison

| 特性 | DailyNoteWrite | LightMemo | ThoughtCluster | AgentDream |
|---|---|---|---|---|
| 主要功能 | 日记写入 | 快速检索+记录 | 主题聚类存储 | 梦境整合记忆 |
| 触发者 | AI 主动调用 | AI 主动调用 | AI 主动调用 | 定时/手动 |
| 写入目录 | `dailynote/<角色>/` | `dailynote/<角色>/` | `dailynote/<X>簇/` | `dailynote/<角色>/` |
| Tag 处理 | ✅ AI 自动生成/规范化 | ❌ 不写入 | 🔍 可选 | ✅ AI 生成 |
| 人工审批 | ❌ 直接写入 | ❌ 直接检索 | ❌ 直接写入 | ✅ 必须审批 |
| 文件格式 | Markdown + 头信息 | — | Markdown | Markdown |
| 检索集成 | 触发 chokidar | 使用现有索引 | 触发 chokidar | 触发 chokidar |
| 适合场景 | 事件/情感记录 | 精确关键词搜索 | 知识/经验积累 | 记忆清理整合 |

---

> ⬆️ [README.md](./README.md) | ⬅️ [07_meta_thinking_time_parser.md](./07_meta_thinking_time_parser.md)
