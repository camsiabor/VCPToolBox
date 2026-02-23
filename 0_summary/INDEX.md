# VCPToolBox 知识库总目录 / Knowledge Base Index

> **用途 / Purpose**: 本目录是对 VCPToolBox 仓库进行理解与分析的文档集合，供 AI Coding Agent 和开发者快速定位系统知识。  
> **Purpose**: This directory is a curated collection of analysis and understanding documents for the VCPToolBox repository, helping AI Coding Agents and developers quickly locate system knowledge.

> **维护原则 / Maintenance Principle**: 只写文档，不改代码。Documents only — no code modifications.

---

## 🗺️ 文档导航 / Navigation

| 文档 / Document | 内容 / Content | 优先级 / Priority |
|---|---|---|
| [01_architecture/README.md](./01_architecture/README.md) | 系统整体架构、启动序列、核心三角 / Overall architecture, boot sequence, core triangle | ⭐⭐⭐ |
| [02_plugin_system/README.md](./02_plugin_system/README.md) | 插件生态、六类插件、manifest 协议、VCP 工具调用语法 / Plugin ecosystem, 6 types, manifest spec, VCP tool-call syntax | ⭐⭐⭐ |
| [03_memory_rag/README.md](./03_memory_rag/README.md) | 记忆系统、TagMemo 浪潮算法、EPA、残差金字塔、向量索引 / Memory system, TagMemo Wave, EPA, Residual Pyramid, vector index | ⭐⭐⭐ |
| [04_api_routes/README.md](./04_api_routes/README.md) | HTTP/SSE 端点、认证机制、中间件链、请求流程 / HTTP/SSE endpoints, auth, middleware, request flow | ⭐⭐⭐ |
| [05_distributed/README.md](./05_distributed/README.md) | 分布式 WebSocket 架构、节点注册、工具调度、文件传输 / Distributed WS architecture, node registration, tool dispatch, file transfer | ⭐⭐ |
| [06_frontend/README.md](./06_frontend/README.md) | AdminPanel、VCPChrome、OpenWebUISub / Admin panel, Chrome extension, user scripts | ⭐⭐ |
| [07_config_deploy/README.md](./07_config_deploy/README.md) | 配置系统、部署方式、Docker、环境变量 / Config system, deployment, Docker, env vars | ⭐⭐ |
| [08_ai_agent_guide/README.md](./08_ai_agent_guide/README.md) | AI Coding Agent 快速参考、常见任务模式、调试指南 / AI Coding Agent quick-reference, task patterns, debug guide | ⭐⭐⭐ |

**深度解析系列 / Deep Dive Series**

| 文档 / Document | 内容 / Content | 优先级 / Priority |
|---|---|---|
| [09_agent_calling/README.md](./09_agent_calling/README.md) | VCP 调用协议、ToolCallParser、ToolExecutor、Archery、流式/并发处理、与 LangChain/AutoGen 对比 / VCP call protocol, ToolCallParser, ToolExecutor, Archery mode, streaming, framework comparison | ⭐⭐⭐ |
| [10_agent_memory/README.md](./10_agent_memory/README.md) | 记忆四层架构、日记系统、AIMemo、LightMemo、ThoughtCluster、SQLite KV、与 MemGPT/Zep 对比 / 4-layer memory arch, diary system, AIMemo, ThoughtCluster, MemGPT/Zep comparison | ⭐⭐⭐ |
| [11_agent_rag/README.md](./11_agent_rag/README.md) | TagMemo 浪潮算法完整实现、EPA 加权PCA、残差金字塔、SVD去重、RAGDiaryPlugin、元思考系统、多级缓存、Rerank、与 LlamaIndex/Haystack 对比 / Full TagMemo algorithm, EPA, residual pyramid, SVD dedup, meta-thinking, caching, framework comparison | ⭐⭐⭐ |
| [12_agent_task_schedule/README.md](./12_agent_task_schedule/README.md) | Cron 调度、ScheduleManager、异步任务、Archery 后台触发、文件监听触发、与 BullMQ/Temporal/n8n 对比 / Cron scheduling, ScheduleManager, async tasks, file watcher triggers, framework comparison | ⭐⭐ |

---

## 🔖 快速查找 / Quick Lookup

### 按任务 / By Task

| 任务 / Task | 跳转 / Go To |
|---|---|
| 理解系统如何启动 / Understand system startup | [01_architecture § 启动序列](./01_architecture/README.md#启动序列--boot-sequence) |
| 开发一个新插件 / Develop a new plugin | [02_plugin_system § 快速开发](./02_plugin_system/README.md#快速开发指南--quick-dev-guide) |
| 优化 RAG 检索效果 / Optimize RAG retrieval | [03_memory_rag § TagMemo 算法](./03_memory_rag/README.md#tagmemo-浪潮算法--tagmemo-wave-algorithm) |
| 新增 API 端点 / Add a new API endpoint | [04_api_routes § 路由注册](./04_api_routes/README.md#路由注册--route-registration) |
| 部署分布式节点 / Deploy a distributed node | [05_distributed § 节点注册](./05_distributed/README.md#节点注册流程--node-registration) |
| 修改管理面板 / Modify admin panel | [06_frontend § AdminPanel](./06_frontend/README.md#adminpanel-管理面板) |
| 修改配置参数 / Change a config param | [07_config_deploy § 配置参数](./07_config_deploy/README.md#核心配置参数--core-config-params) |
| AI Agent 调试技巧 / AI Agent debug tips | [08_ai_agent_guide](./08_ai_agent_guide/README.md) |
| 理解 VCP 工具调用机制 / Understand VCP tool-call mechanism | [09_agent_calling](./09_agent_calling/README.md) |
| 理解记忆写入/读取流程 / Understand memory write/read flow | [10_agent_memory](./10_agent_memory/README.md) |
| 调优 RAG 检索效果 / Tune RAG retrieval quality | [11_agent_rag](./11_agent_rag/README.md) |
| 理解/扩展任务调度 / Understand/extend task scheduling | [12_agent_task_schedule](./12_agent_task_schedule/README.md) |

### 按模块 / By Module

| 核心文件 / Core File | 职责 / Responsibility | 对应文档 / Doc |
|---|---|---|
| `server.js` | HTTP 入口、启动编排 / HTTP entry, startup orchestration | [01_architecture](./01_architecture/README.md) |
| `Plugin.js` | 插件生命周期 / Plugin lifecycle | [02_plugin_system](./02_plugin_system/README.md) |
| `WebSocketServer.js` | 分布式 WS 骨架 / Distributed WS backbone | [05_distributed](./05_distributed/README.md) |
| `KnowledgeBaseManager.js` | 向量库/RAG 总控 / Vector DB / RAG control | [03_memory_rag](./03_memory_rag/README.md) |
| `EPAModule.js` | 语义空间定位 / Semantic space projection | [03_memory_rag](./03_memory_rag/README.md) |
| `ResidualPyramid.js` | 残差能量分解 / Residual energy decomposition | [03_memory_rag](./03_memory_rag/README.md) |
| `routes/` | Express 路由层 / Express routing layer | [04_api_routes](./04_api_routes/README.md) |
| `AdminPanel/` | 管理面板前端 / Admin panel frontend | [06_frontend](./06_frontend/README.md) |
| `rust-vexus-lite/` | Rust N-API 向量引擎 / Rust N-API vector engine | [03_memory_rag](./03_memory_rag/README.md) |
| `modules/` | 复用后端内部模块 / Reusable backend modules | [01_architecture](./01_architecture/README.md) |

---

## 📐 约定与规范 / Conventions

### 本目录的写作约定 / Writing Conventions for This Directory

1. **双语优先 / Bilingual First** — 所有文档中文与英文并列，便于国际协作。All docs are written in Chinese + English side by side.
2. **证据定位 / Evidence Anchoring** — 关键结论附原始文件路径与行号。Key conclusions are anchored with source file path + line numbers.
3. **只写分析，不改代码 / Analysis Only** — 本目录所有文件仅供理解，不影响运行时。All files in this directory are for understanding only, zero runtime impact.
4. **AI Agent 友好 / AI Agent Friendly** — 每节均有可机读的表格、代码块和跳转链接。Every section has machine-readable tables, code blocks, and jump links.
5. **分层组织 / Layered Organization** — 按子模块拆分，避免单文件过大。Split by sub-module to avoid bloated single files.

### 标注规范 / Annotation Conventions

| 标注 / Tag | 含义 / Meaning |
|---|---|
| ✅ 已确认 / Confirmed | 直接来自源码 / Directly from source code |
| 🔍 推断 / Inferred | 基于代码逻辑合理推断 / Reasonably inferred from code logic |
| ⚠️ 风险 / Risk | 潜在问题或需注意的边界条件 / Potential issues or boundary conditions |
| ❓ 待验 / Unverified | 需要运行时验证 / Needs runtime verification |
| 📁 位置 / Location | 文件路径 + 行号 / File path + line number |

---

## 📜 版本记录 / Version History

| 日期 / Date | 版本 / Version | 描述 / Description |
|---|---|---|
| 2026-02-23 | 1.0.0 | 初始创建，覆盖核心模块 / Initial creation, covering core modules |

---

> **参考文档 / See Also**: `docs/` 目录包含更早期、更底层的工程文档（英文为主）。`docs/` contains earlier low-level engineering docs (primarily Chinese).
