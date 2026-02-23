# 仓库生态与协作约定 / Repository Ecosystem & Collaboration Agreement

> **所属目录 / Parent**: [0_summary/INDEX.md](./INDEX.md)  
> **版本 / Version**: 1.0.0 · 2026-02-23  
> **⚠️ 本文件对 AI Coding Agent 具有强制约束力 / This file is binding for AI Coding Agents**

---

## 目录 / Table of Contents

1. [仓库生态地图 / Repository Ecosystem Map](#1-仓库生态地图--repository-ecosystem-map)
2. [本 Fork 的核心约定 / This Fork's Core Agreement](#2-本-fork-的核心约定--this-forks-core-agreement)
3. [上游同步策略 / Upstream Sync Strategy](#3-上游同步策略--upstream-sync-strategy)
4. [AI Coding Agent 操作规范 / AI Coding Agent Operating Rules](#4-ai-coding-agent-操作规范--ai-coding-agent-operating-rules)
5. [各仓库职责说明 / Per-Repo Responsibility](#5-各仓库职责说明--per-repo-responsibility)
6. [同步操作快速参考 / Sync Quick Reference](#6-同步操作快速参考--sync-quick-reference)

---

## 1. 仓库生态地图 / Repository Ecosystem Map

### 上游源码仓库（lioensky）/ Upstream Source Repos (lioensky)

| 仓库 / Repo | URL | 说明 / Description |
|---|---|---|
| VCPToolBox (源) | https://github.com/lioensky/VCPToolBox | VCP 核心中间层 — 权威代码来源 / VCP core middleware — authoritative code source |
| VCPDistributedServer (源) | https://github.com/lioensky/VCPDistributedServer | 分布式节点服务 / Distributed node service |
| VCPChat (源) | https://github.com/lioensky/VCPChat | VCP 官方聊天前端 / VCP official chat frontend |

### 本 Fork 仓库（camsiabor）/ This Fork (camsiabor)

| 仓库 / Repo | URL | 用途 / Purpose |
|---|---|---|
| **VCPToolBox (fork)** | https://github.com/camsiabor/VCPToolBox | 📖 理解与分析文档 / Understanding & analysis docs |
| VCPDistributedServer (fork) | https://github.com/camsiabor/VCPDistributedServer | 📖 理解与分析文档 / Understanding & analysis docs |
| VCPChat (fork) | https://github.com/camsiabor/VCPChat | 📖 理解与分析文档 / Understanding & analysis docs |

### 仓库关系图 / Repository Relationship Diagram

```
上游源码（lioensky）/ Upstream Source (lioensky)
┌─────────────────────────────────────────────────────────────────┐
│  lioensky/VCPToolBox ◄──── 权威代码，持续演进 / Authoritative   │
│  lioensky/VCPDistributedServer                                  │
│  lioensky/VCPChat                                               │
└────────────────────────┬────────────────────────────────────────┘
                         │ git fork + periodic pull sync
                         ▼
本 Fork（camsiabor）/ This Fork (camsiabor)
┌─────────────────────────────────────────────────────────────────┐
│  camsiabor/VCPToolBox                                           │
│    ├── [上游代码，不修改 / Upstream code, unmodified]           │
│    └── 0_summary/   ← 仅此目录由本 fork 新增和维护             │
│                        Only this dir is added & maintained here │
│  camsiabor/VCPDistributedServer                                 │
│  camsiabor/VCPChat                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 本 Fork 的核心约定 / This Fork's Core Agreement

### 🔴 强制约定（AI Coding Agent 必须遵守）/ Mandatory Rules

```
┌────────────────────────────────────────────────────────────────┐
│  本 FORK 的唯一目的 / THE SOLE PURPOSE OF THIS FORK:           │
│                                                                │
│  📖 对 VCPToolBox 生态进行理解、分析和文档化                   │
│  📖 Understand, analyze, and document the VCPToolBox ecosystem │
│                                                                │
│  ❌ 严禁修改任何非 0_summary/ 的文件                           │
│  ❌ NEVER modify any file outside of 0_summary/               │
│                                                                │
│  ✅ 所有贡献必须位于 0_summary/ 目录下                         │
│  ✅ All contributions MUST reside under 0_summary/            │
└────────────────────────────────────────────────────────────────┘
```

### 理由 / Rationale

1. **上游独立演进** — lioensky 的仓库持续更新，本 fork 不应产生合并冲突
2. **文档与代码解耦** — 分析文档独立于代码，任何人可查阅而无需担心文档引入 bug
3. **易于同步** — 由于代码区域完全未修改，`git pull upstream` 永远干净，无冲突
4. **可信度** — 分析文档引用的是上游的权威代码，不是本地修改的版本

---

## 3. 上游同步策略 / Upstream Sync Strategy

### 3.1 何时同步 / When to Sync

| 触发条件 / Trigger | 建议操作 / Recommended Action |
|---|---|
| lioensky 发布新版本 / lioensky releases new version | 执行完整同步 / Full sync |
| 文档内容与实际代码出现不一致 / Docs diverge from actual code | 先同步代码，再更新文档 / Sync code first, then update docs |
| 需要分析新功能 / Need to analyze new feature | 先同步代码，再写分析文档 / Sync code first, then write docs |
| 定期维护（建议每月一次）/ Regular maintenance (monthly) | 执行完整同步 / Full sync |

### 3.2 同步步骤 / Sync Steps

```bash
# Step 1: 添加上游远程（首次操作，已有则跳过）
# Add upstream remote (first time only, skip if already exists)
git remote add upstream https://github.com/lioensky/VCPToolBox.git

# Step 2: 确认远程列表
git remote -v
# 应看到:
# origin   https://github.com/camsiabor/VCPToolBox.git (fetch/push)
# upstream https://github.com/lioensky/VCPToolBox.git  (fetch/push)

# Step 3: 拉取上游最新代码（不自动合并）
git fetch upstream

# Step 4: 查看上游有哪些新提交
git log HEAD..upstream/main --oneline

# Step 5: 合并上游到本地分支
# 由于本 fork 不修改代码，此步骤不会产生冲突
git merge upstream/main

# Step 6: 如果文档需要更新，在 0_summary/ 下修改文档后提交
# Update docs in 0_summary/ if needed, then commit

# Step 7: 推送到本 fork
git push origin main
```

### 3.3 同步后文档维护 / Post-Sync Doc Maintenance

同步后，检查以下内容是否需要更新文档：

```
检查清单 / Checklist:
□ 新增了哪些插件？（更新 02_plugin_system/README.md）
□ 核心模块有哪些变化？（更新 01_architecture/README.md）
□ 配置项有哪些变化？（更新 07_config_deploy/README.md）
□ RAG 系统有哪些优化？（更新 11_agent_rag/README.md）
□ API 路由有哪些变化？（更新 04_api_routes/README.md）
□ ChangeLog.md 有哪些新条目？
```

---

## 4. AI Coding Agent 操作规范 / AI Coding Agent Operating Rules

### 4.1 允许的操作 / Permitted Operations

```
✅ 在 0_summary/ 下创建新的分析文档
✅ 修改 0_summary/ 下的现有文档
✅ 在 0_summary/ 下创建新的子目录
✅ 读取任意源码文件（用于分析理解）
✅ 执行 git fetch upstream / git merge upstream/main（同步操作）
✅ 更新 0_summary/INDEX.md 的导航链接
```

### 4.2 禁止的操作 / Prohibited Operations

```
❌ 修改 0_summary/ 以外的任何文件
❌ 修改 server.js, Plugin.js, KnowledgeBaseManager.js 等源码文件
❌ 修改 Plugin/ 目录下的任何文件
❌ 修改 AdminPanel/ 目录下的任何文件
❌ 修改 routes/, modules/, rust-vexus-lite/ 目录
❌ 修改 package.json, Dockerfile, docker-compose.yml
❌ 修改 README.md, VCP.md, ChangeLog.md 等根目录文档
❌ 提交包含 API 密钥或敏感信息的文件
```

### 4.3 文档质量标准 / Documentation Quality Standards

所有在 `0_summary/` 下创建的文档必须满足：

1. **双语** — 中文 + 英文并列（参考现有文档格式）
2. **证据定位** — 关键结论附原始文件路径和行号
3. **结构化** — 使用表格、代码块、ASCII 架构图
4. **可追溯** — 每个文档顶部注明基于哪个版本/提交的代码
5. **跨链接** — 相关文档之间建立双向链接

### 4.4 发现代码问题时的处理 / Handling Discovered Code Issues

```
场景: AI Agent 在分析源码时发现了 bug 或安全漏洞

正确做法 / Correct approach:
  1. 在 0_summary/ 下创建问题报告文档
  2. 详细描述问题位置、原因和影响
  3. 提出修复建议（文字描述，不直接修改代码）
  4. 在 INDEX.md 中标记 ⚠️

错误做法 / Wrong approach:
  ❌ 直接修改源码文件
  ❌ 在代码文件中添加注释
```

---

## 5. 各仓库职责说明 / Per-Repo Responsibility

### VCPToolBox

| 方面 / Aspect | 上游 lioensky/VCPToolBox | 本 Fork camsiabor/VCPToolBox |
|---|---|---|
| 代码维护 / Code maintenance | ✅ 是 | ❌ 否 |
| 功能开发 / Feature development | ✅ 是 | ❌ 否 |
| Bug 修复 / Bug fixes | ✅ 是 | ❌ 否 |
| 架构分析文档 / Architecture docs | ❌ 否 | ✅ 是 (0_summary/) |
| 深度技术解析 / Deep technical analysis | ❌ 否 | ✅ 是 (0_summary/) |
| 框架对比分析 / Framework comparison | ❌ 否 | ✅ 是 (0_summary/) |

### VCPDistributedServer

分布式节点服务，部署在远程机器上，通过 WebSocket 连接主服务器。

- **上游源码**: https://github.com/lioensky/VCPDistributedServer
- **本 Fork**: https://github.com/camsiabor/VCPDistributedServer
- **分析文档位置**: 本 fork 的 `0_summary/05_distributed/` 中包含分布式架构分析

> 🔍 **待创建**: `0_summary/13_distributed_server/README.md` — 专门针对 VCPDistributedServer 的深度分析

### VCPChat

VCP 专属聊天前端，提供优化的 VCP 工具调用可视化体验。

- **上游源码**: https://github.com/lioensky/VCPChat
- **本 Fork**: https://github.com/camsiabor/VCPChat
- **分析文档位置**: 本 fork 的 `0_summary/06_frontend/` 中有部分提及

> 🔍 **待创建**: `0_summary/14_vcpchat/README.md` — 专门针对 VCPChat 的深度分析

---

## 6. 同步操作快速参考 / Sync Quick Reference

### 首次设置（仅需执行一次）/ First-Time Setup (Run Once)

```bash
# 1. 克隆本 fork
git clone https://github.com/camsiabor/VCPToolBox.git
cd VCPToolBox

# 2. 添加三个上游远程
git remote add upstream-vcptoolbox https://github.com/lioensky/VCPToolBox.git
git remote add upstream-distributed https://github.com/lioensky/VCPDistributedServer.git
git remote add upstream-vcpchat https://github.com/lioensky/VCPChat.git

# 验证
git remote -v
```

### 日常同步（VCPToolBox）/ Daily Sync (VCPToolBox)

```bash
git fetch upstream-vcptoolbox
git merge upstream-vcptoolbox/main
# 由于本 fork 只在 0_summary/ 添加文件，此步骤无冲突
git push origin main
```

### 检查上游变更摘要 / Check Upstream Changes Summary

```bash
# 查看上游新提交（不合并）
git fetch upstream-vcptoolbox
git log HEAD..upstream-vcptoolbox/main --oneline

# 查看上游变更的文件列表
git diff HEAD upstream-vcptoolbox/main --name-only | grep -v "^0_summary"
```

---

## 版本记录 / Version History

| 日期 / Date | 版本 / Version | 变更 / Change |
|---|---|---|
| 2026-02-23 | 1.0.0 | 初始创建 / Initial creation |

---

> ⬆️ 返回 / Back: [INDEX.md](./INDEX.md)
