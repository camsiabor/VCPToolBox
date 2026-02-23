# 03 EPA + ResidualPyramid + ResultDeduplicator — 语义分析三件套
# 03 EPA + ResidualPyramid + ResultDeduplicator — Semantic Analysis Trio

> **所属 / Parent**: [README.md](./README.md)  
> **源文件 / Sources**:  
> - `EPAModule.js` (485 行)  
> - `ResidualPyramid.js` (391 行)  
> - `ResultDeduplicator.js` (195 行)

---

## 目录 / Table of Contents

1. [三件套关系概览 / Overview of the Trio](#1-三件套关系概览--overview-of-the-trio)
2. [EPAModule — 嵌入投影分析](#2-epamodule--嵌入投影分析)
   - 2.1 [加权 PCA 初始化 / Weighted PCA Init](#21-加权-pca-初始化--weighted-pca-init)
   - 2.2 [向量投影 project() / Vector Projection](#22-向量投影-project--vector-projection)
   - 2.3 [跨域共振检测 / Cross-Domain Resonance](#23-跨域共振检测--cross-domain-resonance)
   - 2.4 [Power Iteration 算法 / Power Iteration Algorithm](#24-power-iteration-算法--power-iteration-algorithm)
   - 2.5 [EPA 缓存机制 / EPA Cache](#25-epa-缓存机制--epa-cache)
3. [ResidualPyramid — 残差金字塔](#3-residualpyramid--残差金字塔)
   - 3.1 [Gram-Schmidt 正交投影 / Orthogonal Projection](#31-gram-schmidt-正交投影--orthogonal-projection)
   - 3.2 [金字塔分析流程 / Pyramid Analysis](#32-金字塔分析流程--pyramid-analysis)
   - 3.3 [特征提取 / Feature Extraction](#33-特征提取--feature-extraction)
   - 3.4 [握手差值分析 / Handshake Analysis](#34-握手差值分析--handshake-analysis)
4. [ResultDeduplicator — 结果智能去重](#4-resultdeduplicator--结果智能去重)
   - 4.1 [SVD 潜在主题提取 / SVD Latent Topics](#41-svd-潜在主题提取--svd-latent-topics)
   - 4.2 [残差选择算法 / Residual Selection](#42-残差选择算法--residual-selection)
5. [三件套在 TagMemo 中的协作 / Collaboration in TagMemo](#5-三件套在-tagmemo-中的协作--collaboration-in-tagmemo)
6. [数学公式汇总 / Math Formula Summary](#6-数学公式汇总--math-formula-summary)
7. [Rust 加速路径 / Rust Acceleration Paths](#7-rust-加速路径--rust-acceleration-paths)
8. [调试 Checklist / Debug Checklist](#8-调试-checklist--debug-checklist)

---

## 1. 三件套关系概览 / Overview of the Trio

```
查询向量 q
    │
    ▼
EPAModule.project(q)
    └─ 输出: logicDepth(L), entropy(H), resonance(R), dominantAxes
    │  (理解"这个查询在哪个语义世界，意图有多明确")
    │
    ▼
ResidualPyramid.analyze(q)
    └─ 输出: levels[{tags, energyExplained}], features{coverage, novelty, tagMemoActivation}
    │  (理解"查询的语义能量被哪些 Tag 分层解释")
    │
    ▼
KnowledgeBaseManager._applyTagBoostV3(q, tagBoost, coreTags)
    └─ 利用 EPA + Pyramid 的输出动态调整 Tag 权重，融合出增强查询向量
    │
    ▼
KNN 搜索 → 候选结果集 candidates
    │
    ▼
ResultDeduplicator.deduplicate(candidates, q)
    └─ SVD 潜在主题分析 + 残差迭代选择 = 多样性最优结果子集
```

---

## 2. EPAModule — 嵌入投影分析

**📁 位置**: `EPAModule.js`  
**名称含义**: **E**mbedding **P**rojection **A**nalysis

### 2.1 加权 PCA 初始化 / Weighted PCA Init

**📁 位置**: `EPAModule.js:33` — `initialize()`

```
初始化触发条件: KnowledgeBaseManager.initialize() 调用
目标: 用所有已知 Tag 构建语义空间的正交基底

算法步骤:

① SQL 查询: SELECT id, name, vector FROM tags WHERE vector IS NOT NULL
   要求: ≥8 个 Tag（否则跳过）

② 鲁棒 K-Means 聚类 (k=32)
   目的: 从大量 Tag 中提取代表性质心，减少噪音 Tag 的影响
   
   function _clusterTags(tags, k):
     随机初始化 k 个质心
     迭代 maxIter=50 次:
       分配每个 Tag 到最近质心
       重新计算质心 (带权重: 质心所含 Tag 数)
     返回: { vectors: centroids, weights, labels }

③ 加权中心化 PCA (_computeWeightedPCA):
   3a. 计算全局加权平均向量 mean = Σ(v_i × w_i) / Σw_i
   3b. 构建中心化缩放矩阵: x_i = sqrt(w_i) × (v_i - mean)
   3c. 构建 Gram 矩阵 G (n×n): G_ij = dot(x_i, x_j)
   3d. Power Iteration 提取前 K 个特征向量/值
   3e. 将 n 维特征向量映射回 dim 维原始空间

④ 选择主成分数 K
   累计解释方差 ≥ 95% 时停止，且 K ≥ 8

⑤ 保存:
   orthoBasis[K], basisMean, basisEnergies, basisLabels
   → kv_store WHERE key='epa_basis_cache' (JSON base64)
```

### 2.2 向量投影 project() / Vector Projection

**📁 位置**: `EPAModule.js:72` — `project(vector)`

```
输入: queryVector (Float32Array, dim=3072)
输出: { projections, probabilities, entropy, logicDepth, dominantAxes }

步骤:
  ① 优先尝试 Rust VexusIndex.project() (速度更快)
     参数: (vec, flattenedBasis, basisMean, K)
     输出: { projections, probabilities, entropy, totalEnergy }
  
  如果 Rust 失败，JS Fallback:
  ② 去中心化: centeredVec = vec - basisMean
  ③ 计算投影系数: projections[k] = dot(centeredVec, orthoBasis[k])
  ④ 计算能量: totalEnergy = Σ(projections[k]^2)
  ⑤ Softmax 概率: p[k] = projections[k]^2 / totalEnergy
  ⑥ Shannon 熵: entropy = -Σ p[k] × log(p[k])
  
  ⑦ 逻辑深度: logicDepth = 1 - entropy / ln(K)
     [0 = 完全分散在所有轴, 1 = 集中在单一轴]
  
  ⑧ 主导轴标签: dominantAxes 按 probability 降序排列
     dominantAxes[0].label = queryWorld (语义世界名称)
```

**直觉理解 / Intuitive understanding**:
```
低熵 (H→0): 查询能量集中在 1-2 个主成分轴 → 意图明确 → logicDepth 高
高熵 (H→1): 查询能量分散在所有轴 → 语义模糊 → logicDepth 低

例子:
  "量子力学导论" → H=0.2, L=0.8 → 意图明确，物理学领域
  "今天" → H=0.9, L=0.1 → 意图模糊
  "量子计算在金融中的应用" → H=0.5, L=0.5 → 跨域共振
```

### 2.3 跨域共振检测 / Cross-Domain Resonance

**📁 位置**: `EPAModule.js:detectCrossDomainResonance()`

```javascript
detectCrossDomainResonance(vector) {
    const projection = this.project(vector)
    
    // 找到投影能量 ≥ 0.1 的活跃轴
    const activeAxes = projection.probabilities.filter(p => p >= 0.1)
    
    // 共振强度 = 活跃轴数量 × 各轴平均投影量级
    const resonance = activeAxes.length * mean(activeAxes)
    
    return { resonance, activeAxisCount: activeAxes.length }
}
```

**共振强度解读**:
- `resonance ≈ 0.1`: 单一领域聚焦 → TagMemo 保守增强
- `resonance ≈ 0.5`: 跨 2-3 个领域 → TagMemo 中度增强
- `resonance > 1.0`: 高度跨域 → TagMemo 激进增强

### 2.4 Power Iteration 算法 / Power Iteration Algorithm

**📁 位置**: `EPAModule.js:_powerIteration()`

```
目的: 提取 Gram 矩阵的第 k 个最大特征向量（对应第 k 个主成分）

算法:
  随机初始化向量 v (n维)
  循环 100 次:
    w = G × v                        矩阵-向量乘法
    Rayleigh Quotient: λ = v·w       近似特征值
    ★ Re-orthogonalization:          防止收敛到已有主成分
       for (prevV of existingBasis):
           w -= dot(w, prevV) × prevV
    归一化: v = w / ||w||
    if |λ - λ_prev| < 1e-6: break   收敛判定

返回: { vector: v, value: λ }
```

**关键优化 / Key optimization**: Re-orthogonalization (Gram-Schmidt 对已有特征向量) 解决了传统 Power Iteration 中 Deflation 精度丢失的问题。

### 2.5 EPA 缓存机制 / EPA Cache

```javascript
// 存储位置: kv_store WHERE key='epa_basis_cache'
// 缓存失效条件: Tag 数量变化时（由 tags 表 COUNT 检测）

// 缓存格式 (JSON)
{
    basis: ["<base64(float32array)>", ...],  // K 个基底向量
    mean:  "<base64(float32array)>",          // 全局平均向量
    energies: [0.23, 0.18, ...],             // 特征值列表
    labels: ["物理学", "技术", ...],           // 轴标签
    timestamp: 1234567890,
    tagCount: 5432                            // 用于失效检测
}
```

---

## 3. ResidualPyramid — 残差金字塔

**📁 位置**: `ResidualPyramid.js`

### 3.1 Gram-Schmidt 正交投影 / Orthogonal Projection

**📁 位置**: `ResidualPyramid.js:_computeOrthogonalProjection()`

```
输入: vector v, tags [{id, name, vector}]

① 正交化 Tag 向量集合 (Gram-Schmidt):
   orthogonalBasis = []
   for (tag of tags):
       u = tag.vector
       for (e of orthogonalBasis):  // 去除在已有基底方向上的分量
           u = u - dot(u, e) × e
       if ||u|| > ε: 
           normalize u
           orthogonalBasis.push(u)
           basisCoefficients[i] = dot(original_tag_vec, u)

② 计算投影 P (v 在 Tag 张成子空间上的投影):
   P = Σ(dot(v, e_k) × e_k)

③ 计算残差 R:
   R = v - P

返回: { projection P, residual R, orthogonalBasis, basisCoefficients }
```

**物理直觉**: 残差 `R` 是查询向量中**不能被这批 Tag 解释**的部分，代表语义上的"新颖度"。

### 3.2 金字塔分析流程 / Pyramid Analysis

**📁 位置**: `ResidualPyramid.js:analyze()`

```
analyze(queryVector):

初始化:
  currentResidual = queryVector
  originalEnergy = ||queryVector||^2
  pyramid = { levels: [], totalExplainedEnergy: 0 }

循环 (最多 maxLevels=3 次):
  Level L:
  ① VexusIndex.search(currentResidual, topK=10) → 找最相关 Tags
     // 注意: 每次搜的是"剩余残差"，不是原始查询！
  
  ② 获取 Tag 向量: SQL SELECT FROM tags WHERE id IN (...)
  
  ③ Gram-Schmidt 正交投影:
     { projection, residual, basisCoefficients } =
       _computeOrthogonalProjection(currentResidual, rawTags)
  
  ④ 能量分析:
     currentEnergy = ||currentResidual||^2
     residualEnergy = ||residual||^2
     energyExplainedByLevel = (currentEnergy - residualEnergy) / originalEnergy
  
  ⑤ 握手分析:
     handshakes = _computeHandshakes(currentResidual, rawTags)
  
  ⑥ 记录本层数据:
     pyramid.levels[L] = {
         level: L,
         tags: [{ id, name, similarity, contribution, handshakeMagnitude }],
         projectionMagnitude, residualMagnitude,
         energyExplainedByLevel,
         handshakeFeatures: _analyzeHandshakes(handshakes)
     }
     pyramid.totalExplainedEnergy += energyExplainedByLevel
  
  ⑦ 更新残差: currentResidual = residual
  
  终止条件:
    residualEnergy / originalEnergy < minEnergyRatio (=0.1)
    即 Tag 已解释了 ≥90% 的语义能量

返回: pyramid + features = _extractPyramidFeatures(pyramid)
```

### 3.3 特征提取 / Feature Extraction

**📁 位置**: `ResidualPyramid.js:_extractPyramidFeatures()`

```javascript
features = {
    depth: pyramid.levels.length,  // 金字塔深度 (1~3)
    
    coverage: min(1.0, pyramid.totalExplainedEnergy),
    // Tag 可解释的能量占比 (0=未覆盖, 1=完全覆盖)
    
    novelty: (residualRatio × 0.7) + (directionalNovelty × 0.3),
    // 新颖度: 未被 Tag 解释的能量 + 方向一致性加权
    
    coherence: level0.handshakeFeatures.patternStrength,
    // 相干度: 第一层 Tag 的方向一致性
    
    tagMemoActivation: coverage × coherence × (1 - noiseSignal),
    // ★ TagMemo 激活强度 (0~1)
    // 高覆盖 + 高相干 + 低噪音 = 高激活
    
    expansionSignal: novelty  // 是否需要探索新 Tag
}
```

**TagMemo 激活强度解读**:
```
tagMemoActivation 高 (>0.5):
  → Tag 已经找到了正确的"语义邻域"，激活 TagMemo 增强效果好
  
tagMemoActivation 低 (<0.2):
  → 查询太新颖 (Tag 覆盖不足) 或 Tag 很杂乱 (噪音高) → 弱化 TagMemo
```

### 3.4 握手差值分析 / Handshake Analysis

**📁 位置**: `ResidualPyramid.js:_analyzeHandshakes()`

```
握手差值 = 查询向量与每个 Tag 向量之间的差向量 (delta)
delta_i = query - tag_i

分析指标:
  directionCoherence: ||mean(normalize(delta_i))||
  // 所有差值方向的一致性
  // 高 → 查询在所有 Tag 的"外部同一方向" (新领域)
  // 低 → 查询被 Tag 包围在"中间" (已知领域细节)
  
  patternStrength: mean(pairwise_cosine_sim(delta_i, delta_j))
  // Tag 之间的方向相似性
  // 高 → Tag 属于同一类 (相干)
  // 低 → Tag 来自不同方向 (发散)
  
  noiseSignal: (1 - directionCoherence) × (1 - patternStrength)
  // 噪音信号: 方向杂乱 + Tag 之间也乱 = 噪音
```

---

## 4. ResultDeduplicator — 结果智能去重

**📁 位置**: `ResultDeduplicator.js`

### 4.1 SVD 潜在主题提取 / SVD Latent Topics

**📁 位置**: `ResultDeduplicator.js:deduplicate()`

```
输入: candidates (搜索结果数组), queryVector

① 过滤无向量的结果 (vector || _vector 字段)
   if (candidates.length ≤ 5): 直接返回（数量太少无需去重）

② 构建临时 clusterData 结构
   vectors = candidates.map(c => c.vector)
   weights = [1, 1, 1, ...]  // 等权重

③ 调用 EPAModule._computeWeightedPCA(clusterData)
   // 注意：这里不用全局 Tag PCA，而是对本次候选结果做即时 PCA
   // 目的：找出这批结果本身的潜在主题分布
   { U: topics, S: energies } = epa._computeWeightedPCA(clusterData)

④ 过滤弱主题 (累积能量 > 95% 后停止)
   significantTopics = topics (覆盖 95% 能量的主成分)
```

### 4.2 残差选择算法 / Residual Selection

```
目标: 选择一组结果，覆盖 queryVector 的最大语义多样性

① Anchor 选择:
   计算每个候选与 queryVector 的余弦相似度
   选择相似度最高的作为 Anchor (第一名)
   initialBasis = [anchor.vector]

② 迭代残差选择 (maxRounds = maxResults - 1):
   for (round of rounds):
     for (candidate not yet selected):
       // 计算候选在已选集合之外的"新信息量"
       { residual } = _computeOrthogonalProjection(
           candidate.vector, currentBasis
       )
       noveltyEnergy = ||residual||^2
       
       // 综合评分: 新信息量 × 原始相关度
       score = noveltyEnergy × (candidate.score + 0.5)
     
     // 选择 score 最高的候选
     nextBest = argmax(score)
     
     // 终止条件: 新信息量微乎其微
     if (maxProjectedEnergy < 0.01): break
     
     selectedResults.push(nextBest)
     currentBasis.push(nextBest.vector)  // 扩充正交基

返回: selectedResults (多样性最优子集)
```

**算法本质**: 这是一种基于正交投影的贪心 **MMR (Maximal Marginal Relevance)** 变体，但使用了更严格的向量正交化而非简单的相似度减法。

---

## 5. 三件套在 TagMemo 中的协作 / Collaboration in TagMemo

```
_applyTagBoostV3(q, baseTagBoost, coreTags):

  epaResult = EPA.project(q)
    └─ L = logicDepth, H = entropy, R = resonance.resonance, queryWorld
  
  pyramid = ResidualPyramid.analyze(q)
    └─ features.tagMemoActivation, features.coverage
  
  // EPA 与 Pyramid 联合决策 TagMemo 强度:
  A_mul = 0.5 + tagMemoActivation × 1.0          // 激活倍率
  dynamicBoostFactor = L(1+ln(1+R)) / (1+H×0.5) × A_mul
  effectiveTagBoost = baseTagBoost × clamp(dynamicBoostFactor, 0.3, 2.0)
  
  // EPA 的 queryWorld 用于语言门控:
  isNoiseTag = isTechnicalNoise(tag) && !isTechnicalWorld(queryWorld)
  langPenalty = if(isNoiseTag) 0.05~0.1 else 1.0
  
  // 构建上下文向量 (含 Pyramid 的 Tag 列表)
  ...
  
  // 最终融合
  fused = normalize((1-β) × q + β × contextVec)
  
  // ResultDeduplicator 在搜索之后运行:
  candidates = KNN_search(fused)
  unique = ResultDeduplicator.deduplicate(candidates, q)
```

---

## 6. 数学公式汇总 / Math Formula Summary

### EPA 公式

```
logicDepth L = 1 - H / ln(K)

H = -Σᵢ p_i ln(p_i)     (Shannon 熵)
p_i = (q·e_i)² / Σⱼ(q·e_j)²   (投影能量占比)
K = 正交基维数 (主成分数)

resonance R = |activeAxes| × mean(p_i for p_i ≥ 0.1)
```

### TagMemo 动态 Boost 公式

```
dynamicBoostFactor = [L × (1 + ln(1+R)) / (1 + H × 0.5)] × A_mul

A_mul = 0.5 + tagMemoActivation × 1.0
      ∈ [0.5, 1.5]

effectiveTagBoost = baseTagBoost × clamp(dynamicBoostFactor, boostMin, boostMax)
                    (boostMin=0.3, boostMax=2.0 from rag_params.json)
```

### 动态 Beta (TagWeight) 公式

```
β_input = L × ln(2 + R) - S × noise_penalty
β = sigmoid(β_input)
finalTagWeight = β × (weightMax - weightMin) + weightMin
                 (weightMin=0.05, weightMax=0.45)
```

### 残差金字塔能量公式

```
E_level_L = (||r_L||² - ||r_{L+1}||²) / ||q||²

coverage = Σ E_level_L    (所有层解释的总能量比)
novelty  = (1 - coverage) × 0.7 + directionCoherence × 0.3
```

### 最终融合公式

```
fused = normalize(
    (1 - β_eff) × q_original
    + β_eff × normalize(Σᵢ wᵢ × t_vec_i)
)

wᵢ = contribution_i × 0.7^level_i × langPenalty_i × coreBoost_i
```

---

## 7. Rust 加速路径 / Rust Acceleration Paths

| JS 方法 | Rust 等价方法 | 加速比 |
|---|---|---|
| `EPA.project()` (JS fallback) | `VexusIndex.project()` | ~5-10x |
| `ResidualPyramid._computeHandshakes()` | `VexusIndex.computeHandshakes()` | ~3-5x |
| `VexusIndex.search()` | Rust HNSW | ~100x (vs JS暴力搜索) |

**Rust 方法调用规范**:
```javascript
// EPA 投影
const result = vexusIndex.project(
    Buffer.from(vec.buffer, vec.byteOffset, vec.byteLength),         // 查询向量
    Buffer.from(flattenedBasis.buffer, ...),                          // K×dim 展平基底
    Buffer.from(basisMean.buffer, ...),                               // dim 均值向量
    K                                                                  // 基底维数
)
// 返回: { projections: number[], probabilities: number[], entropy: number, totalEnergy: number }

// 握手分析
const result = vexusIndex.computeHandshakes(
    queryBuffer,       // dim 维查询
    flattenedTags,     // n×dim 展平 Tag 向量
    n                  // Tag 数量
)
// 返回: { magnitudes: number[], directions: number[] (n×dim) }
```

---

## 8. 调试 Checklist / Debug Checklist

### EPA 调试

```bash
# EPA 初始化成功标志
grep "\[EPA\] 💾 Loaded basis from cache" logs.txt
grep "\[EPA\] ✅ Basis saved to cache" logs.txt

# EPA 初始化失败
grep "\[EPA\] ❌" logs.txt
# 原因1: Tags < 8 个 (需要先积累足够多的日记)
# 原因2: 所有 Tag 向量零向量 (Embedding API 失败)

# EPA 分析输出
grep "\[TagMemo-V3.7\] World=" logs.txt
# 输出: World=物理学, Depth=0.732, Resonance=0.423
```

### ResidualPyramid 调试

```bash
# 残差金字塔分析失败 (搜索失败)
grep "\[Residual\] Search failed" logs.txt

# 金字塔层数
grep "Coverage=" logs.txt
# Coverage 低 (<0.3): Tag 库覆盖不足，需积累更多日记
# Coverage 高 (>0.8): TagMemo 效果好，可提高 tagWeight
```

### ResultDeduplicator 调试

```bash
grep "\[ResultDeduplicator\]" logs.txt
# 输出: Starting deduplication for N candidates
# 输出: Identify M significant latent topics
# 输出: Selected X / N diverse results
# 输出: Remaining candidates provide negligible novelty. Stopping.
```

---

> ⬆️ [README.md](./README.md) | ⬅️ [02_rag_diary_plugin.md](./02_rag_diary_plugin.md) | ➡️ [04_embedding_chunker.md](./04_embedding_chunker.md)
