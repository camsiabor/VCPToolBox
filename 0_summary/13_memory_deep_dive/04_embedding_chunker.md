# 04 EmbeddingUtils + TextChunker — 向量化与分块
# 04 EmbeddingUtils + TextChunker — Vectorization & Chunking

> **所属 / Parent**: [README.md](./README.md)  
> **源文件 / Sources**:  
> - `EmbeddingUtils.js` (161 行)  
> - `TextChunker.js` (约 120 行)

---

## 目录 / Table of Contents

1. [EmbeddingUtils — 批量嵌入工具](#1-embeddingutils--批量嵌入工具)
   - 1.1 [配置参数 / Config](#11-配置参数--config)
   - 1.2 [批次切分策略 / Batch Splitting](#12-批次切分策略--batch-splitting)
   - 1.3 [并发执行器 / Concurrent Executor](#13-并发执行器--concurrent-executor)
   - 1.4 [单批发送 _sendBatch / Single Batch Send](#14-单批发送-_sendbatch--single-batch-send)
   - 1.5 [错误处理 / Error Handling](#15-错误处理--error-handling)
2. [TextChunker — 智能文本分块](#2-textchunker--智能文本分块)
   - 2.1 [分块策略 / Chunking Strategy](#21-分块策略--chunking-strategy)
   - 2.2 [重叠窗口 / Overlap Window](#22-重叠窗口--overlap-window)
   - 2.3 [超长句子处理 / Long Sentence Handling](#23-超长句子处理--long-sentence-handling)
3. [集成配置示例 / Integration Config Example](#3-集成配置示例--integration-config-example)
4. [性能调优 / Performance Tuning](#4-性能调优--performance-tuning)

---

## 1. EmbeddingUtils — 批量嵌入工具

**📁 位置**: `EmbeddingUtils.js`

提供并发批量 Embedding API 调用能力，被 `KnowledgeBaseManager` 和 `RAGDiaryPlugin` 调用。

### 1.1 配置参数 / Config

```javascript
// 从 config.env 读取 / Read from config.env
const embeddingMaxToken = WhitelistEmbeddingModelMaxToken || 8000
const safeMaxTokens     = floor(embeddingMaxToken × 0.85)  // 85% 安全边界
const MAX_BATCH_ITEMS   = 100                               // Gemini/OpenAI 批次限制
const DEFAULT_CONCURRENCY = TAG_VECTORIZE_CONCURRENCY || 5  // 并发数

// 使用 tiktoken cl100k_base 精确计算 token 数
const encoding = get_encoding("cl100k_base")
```

### 1.2 批次切分策略 / Batch Splitting

**📁 位置**: `EmbeddingUtils.js:getEmbeddingsBatch()`

```
输入: texts (string[])

纯 CPU 操作，先把所有文本切分成 Batches:
  currentBatch = []
  currentBatchTokens = 0

  for (text of texts):
    textTokens = encoding.encode(text).length
    
    if (textTokens > safeMaxTokens):
      skip  ← 超长单文本直接跳过（TextChunker 应在前面处理）
    
    isTokenFull = (currentBatchTokens + textTokens > safeMaxTokens)
    isItemFull  = (currentBatch.length >= MAX_BATCH_ITEMS)
    
    if (isTokenFull || isItemFull):
      batches.push(currentBatch)
      currentBatch = [text]
      currentBatchTokens = textTokens
    else:
      currentBatch.push(text)
      currentBatchTokens += textTokens
  
  if (currentBatch.length > 0): batches.push(currentBatch)
```

### 1.3 并发执行器 / Concurrent Executor

**📁 位置**: `EmbeddingUtils.js` — Worker Pool Pattern

```javascript
// 使用 Worker Pool 模式，不是简单的 Promise.all
// 优点: 防止同时发送 N 个请求（可能触发 429 限流）

const results = new Array(batches.length)  // 预分配，保证顺序
let cursor = 0  // 原子游标（JS 单线程安全）

const worker = async (workerId) => {
    while (true) {
        const batchIndex = cursor++        // 取下一个任务
        if (batchIndex >= batches.length): break
        
        results[batchIndex] = await _sendBatch(
            batches[batchIndex], config, batchIndex + 1
        )
    }
}

// 启动 DEFAULT_CONCURRENCY 个 Worker
await Promise.all(
    Array.from({ length: DEFAULT_CONCURRENCY }, (_, i) => worker(i))
)

// 展平结果（过滤失败的 batch）
return results.filter(r => r).flat()
```

**Worker Pool vs Promise.all**:
```
Promise.all:  同时发送 N 个请求 → 可能触发 429
Worker Pool:  最多同时 5 个请求 → 平滑流量
```

### 1.4 单批发送 _sendBatch / Single Batch Send

```javascript
async _sendBatch(batchTexts, config, batchNumber) {
    // 请求格式: OpenAI Embedding API 兼容
    POST config.apiUrl + '/v1/embeddings'
    {
        "model": config.model,
        "input": batchTexts    // string[]
    }
    
    // 响应解析
    // 支持格式: { data: [{ index: 0, embedding: float[] }, ...] }
    return data.data
        .sort((a, b) => a.index - b.index)  // 按 index 排序（保证顺序）
        .map(item => item.embedding)
}
```

**重试策略**:
```
最多 3 次重试
429 限流: 等待 5000ms × attempt 后重试
其他 4xx/5xx: 记录详细错误信息（model 名称、错误码）
失败时: 指数退避 (1000ms × 2^attempt)
```

### 1.5 错误处理 / Error Handling

详细的错误诊断输出，方便 AI Coding Agent 调试：

```javascript
// 错误类型及对应提示
if (data.error) {
    console.error(`  Error Code: ${errorCode}`)
    console.error(`  Error Message: ${errorMsg}`)
    console.error(`  Hint: Check if embedding model "${config.model}" is available`)
}
if (!data.data) {
    console.error(`Response keys: ${Object.keys(data).join(', ')}`)  // 帮助诊断
}
if (!Array.isArray(data.data)) {
    console.error(`data type: ${typeof data.data}`)
}
```

---

## 2. TextChunker — 智能文本分块

**📁 位置**: `TextChunker.js`

将日记原文切分成适合 Embedding 的文本块（chunks），是向量化前的必要预处理。

### 2.1 分块策略 / Chunking Strategy

```
目标: 每个 chunk ≤ safeMaxTokens (= embeddingMaxToken × 85%)
重叠: overlapTokens = safeMaxTokens × 10%

算法:
  sentences = text.split(/(?<=[。？！.!?\n])/g)
  // 正则: 在中英文标点/换行后分割，保留分隔符
  
  currentChunk = ""
  currentTokens = 0
  
  for (sentence of sentences):
    sentenceTokens = tiktoken.encode(sentence).length
    
    if (sentenceTokens > maxTokens):
      flush currentChunk
      forceSplitLongText(sentence)  ← 超长句子强制切分
      continue
    
    if (currentTokens + sentenceTokens > maxTokens):
      chunks.push(currentChunk)
      ★ 创建重叠部分 (backward scan):
        向前扫描 sentences[i-1], [i-2], ...
        累加 tokens 直到超过 overlapTokens
        overlapChunk = 这些句子的拼接
      currentChunk = overlapChunk + sentence
      currentTokens = overlapTokens + sentenceTokens
    else:
      currentChunk += sentence
      currentTokens += sentenceTokens
```

### 2.2 重叠窗口 / Overlap Window

重叠设计确保语义边界处的内容不会因分块而丢失上下文：

```
文本: "...sentence_N-2. sentence_N-1. || sentence_N. sentence_N+1..."
       ┌──────────── Chunk A ──────────────┐
                             ┌──── Overlap ────┐
                             ┌──────── Chunk B ──────────┐

Chunk A: ..., sentence_N-2, sentence_N-1
Chunk B: sentence_N-2, sentence_N-1, sentence_N, sentence_N+1, ...
        (重叠部分: sentence_N-2 + sentence_N-1)
```

**参数配置**:
```bash
# config.env
WhitelistEmbeddingModelMaxToken=8000    # 模型最大 token
# safeMaxTokens = 6800 (85%)
# overlapTokens = 680 (10%)
```

### 2.3 超长句子处理 / Long Sentence Handling

```javascript
// 如果单个句子超过 safeMaxTokens，强制按 token 切分
function forceSplitLongText(text, maxTokens, overlapTokens) {
    const chars = text.split('')
    let currentChunk = ""
    let currentTokens = 0
    const chunks = []
    
    for (char of chars):
        charTokens = tiktoken.encode(char).length
        
        if (currentTokens + charTokens > maxTokens):
            chunks.push(currentChunk)
            // 保留最后 overlapTokens 的内容作为重叠
            overlap = getLastNTokens(currentChunk, overlapTokens)
            currentChunk = overlap + char
            currentTokens = overlapTokens + charTokens
        else:
            currentChunk += char
            currentTokens += charTokens
    
    return chunks
}
```

---

## 3. 集成配置示例 / Integration Config Example

```bash
# config.env — 关键 Embedding 参数

# Embedding 模型
WhitelistEmbeddingModel=google/gemini-embedding-001
WhitelistEmbeddingModelMaxToken=8000  # gemini-embedding 最大 8192

# 向量维度 (必须与模型一致!)
VECTORDB_DIMENSION=3072               # gemini-embedding-001 = 3072

# 并发配置
TAG_VECTORIZE_CONCURRENCY=5           # 同时最多 5 个 Embedding 请求

# API 配置
API_Key=your-api-key
API_URL=https://generativelanguage.googleapis.com/v1beta/openai
```

**常用模型维度参考**:

| 模型 | 维度 |
|---|---|
| `google/gemini-embedding-001` | 3072 |
| `text-embedding-3-small` | 1536 |
| `text-embedding-3-large` | 3072 |
| `text-embedding-ada-002` | 1536 |
| `BAAI/bge-m3` | 1024 |

---

## 4. 性能调优 / Performance Tuning

### 调整批次大小

```bash
# 增大 Token 上限（注意: 必须与模型限制一致）
WhitelistEmbeddingModelMaxToken=16000   # 适用于支持更长输入的模型

# 增大并发数（注意: 可能触发 API 限流）
TAG_VECTORIZE_CONCURRENCY=10           # 高吞吐场景

# 减小安全边界（慎用）
# safeMaxTokens = floor(maxToken × ratio)
# ratio = 0.85 (写死在代码中，如需调整需修改源码)
```

### 监控 Embedding 性能

```bash
# 日志输出格式
grep "\[Embedding\] Prepared" logs.txt
# [Embedding] Prepared 8 batches. Executing with concurrency: 5...

grep "\[Embedding\] ✅ Batch" logs.txt
# [Embedding] ✅ Batch 3 completed (45 items).

grep "\[Embedding\].*429" logs.txt
# [Embedding] Batch 2 rate limited (429). Retrying in 10s...
```

### 调整 TextChunker 参数

```bash
# 如果检索结果经常缺少上下文（上下句被切断）:
# → 增大 overlapTokens
# 如果 chunk 太长导致语义稀释:
# → 减小 maxTokens (修改 safeMaxTokens 的比例系数)
```

---

> ⬆️ [README.md](./README.md) | ⬅️ [03_epa_residual_result.md](./03_epa_residual_result.md) | ➡️ [05_context_vector_manager.md](./05_context_vector_manager.md)
