# 07 配置系统与部署 / Configuration System & Deployment

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `config.env`, `docker-compose.yml`, `Dockerfile`, `rag_params.json`  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [配置文件清单 / Config File Inventory](#1-配置文件清单--config-file-inventory)
2. [核心配置参数 / Core Config Params](#2-核心配置参数--core-config-params)
3. [配置优先级 / Config Priority](#3-配置优先级--config-priority)
4. [快速部署指南 / Quick Deploy Guide](#4-快速部署指南--quick-deploy-guide)
5. [Docker 部署 / Docker Deployment](#5-docker-部署--docker-deployment)
6. [直接运行 / Direct Run](#6-直接运行--direct-run)
7. [前置依赖 / Prerequisites](#7-前置依赖--prerequisites)
8. [故障排查 / Troubleshooting](#8-故障排查--troubleshooting)

---

## 1. 配置文件清单 / Config File Inventory

| 文件 / File | 位置 / Location | 用途 / Purpose | 是否提交 git / In git |
|---|---|---|---|
| `config.env` | 根目录 / Root | 主配置，含 API 密钥 | ❌ 不提交（含密钥）|
| `config.env.example` | 根目录 | 主配置模板 | ✅ 提交 |
| `rag_params.json` | 根目录 | RAG 算法参数，热重载 | ✅ 提交（无密钥）|
| `ModelRedirect.json` | 根目录 | 模型名称重定向映射 | ✅ 提交模板 |
| `agent_map.json` | 根目录 | Agent 别名与文件映射 | ✅ 提交模板 |
| `ip_blacklist.json` | 根目录 | IP 黑名单 | ✅ 可提交 |
| `Plugin/*/config.env` | 插件目录 | 各插件专属配置 | ❌ 不提交（含密钥）|
| `Plugin/*/config.env.example` | 插件目录 | 插件配置模板 | ✅ 提交 |
| `docker-compose.yml` | 根目录 | Docker Compose 配置 | ✅ 提交 |
| `Dockerfile` | 根目录 | Docker 镜像构建 | ✅ 提交 |

### 首次部署初始化命令 / First-time Init Commands

```bash
cp config.env.example config.env
cp agent_map.json.example agent_map.json
cp ModelRedirect.json.example ModelRedirect.json
# 然后编辑 config.env 填入实际 API 密钥
# Then edit config.env with your actual API keys
```

---

## 2. 核心配置参数 / Core Config Params

> 📁 `config.env` (从 `config.env.example` 复制)

### 认证与安全 / Auth & Security

| 参数 / Parameter | 类型 / Type | 说明 / Description | 示例 / Example |
|---|---|---|---|
| `Key` | string | VCP API Bearer Token（**必须修改**）| `mySecretKey123` |
| `AdminUsername` | string | 管理面板用户名 | `admin` |
| `AdminPassword` | string | 管理面板密码 | `myAdminPass` |

### 上游 LLM API / Upstream LLM API

| 参数 / Parameter | 类型 / Type | 说明 / Description | 示例 / Example |
|---|---|---|---|
| `API_URL` | string | 上游 LLM API 基础 URL | `https://api.openai.com` |
| `API_KEY` | string | 上游 API 密钥 | `sk-xxx...` |
| `MODEL` | string | 默认模型名称 | `gpt-4o` |

### 嵌入模型 / Embedding Model

| 参数 / Parameter | 类型 / Type | 说明 / Description |
|---|---|---|
| `EMBEDDING_API_URL` | string | Embedding API URL（可与主 API 不同）|
| `EMBEDDING_API_KEY` | string | Embedding API 密钥 |
| `EMBEDDING_MODEL` | string | Embedding 模型名称（如 `text-embedding-3-small`）|
| `EMBEDDING_DIM` | number | 向量维度（必须与模型匹配）|

### 服务器配置 / Server Config

| 参数 / Parameter | 默认值 / Default | 说明 / Description |
|---|---|---|
| `PORT` | `5890` | HTTP 服务器端口 / HTTP server port |
| `DebugMode` | `false` | 调试模式（详细日志）/ Debug mode (verbose logging) |
| `MaxRetries` | `3` | 上游 API 请求失败重试次数 |

### Agent 系统 / Agent System

| 参数 / Parameter | 说明 / Description |
|---|---|
| `AGENT_DIR_PATH` | Agent 角色卡目录路径（默认为 `Agent/`）|
| `TVS_DIR_PATH` | TVStxt 变量文件目录路径（默认为 `TVStxt/`）|

### 知识库 / Knowledge Base

| 参数 / Parameter | 说明 / Description |
|---|---|
| `KNOWLEDGE_BASE_PATH` | SQLite 数据库路径（默认 `knowledge_base.sqlite`）|
| `DIARY_PATH` | 日记根目录路径（默认 `dailynote/`）|

---

## 3. 配置优先级 / Config Priority

```
高优先级 → 低优先级:
环境变量 (OS env) > config.env > 插件 config.env > manifest configSchema 默认值

High → Low priority:
OS Environment Variables > config.env > Plugin config.env > Manifest defaults
```

**示例 / Example**:
```
设置 PORT=8080 作为 OS 环境变量
同时 config.env 中 PORT=5890

最终使用 PORT=8080（OS 环境变量优先）
```

---

## 4. 快速部署指南 / Quick Deploy Guide

### 推荐路径 / Recommended Path

```
1. 克隆仓库 / Clone repo
   git clone https://github.com/camsiabor/VCPToolBox.git

2. 复制配置模板 / Copy config templates
   cp config.env.example config.env
   cp agent_map.json.example agent_map.json

3. 编辑主配置 / Edit main config
   vim config.env  # 填入 API_KEY、Key 等必填项

4. 构建 Rust 向量引擎 / Build Rust vector engine
   cd rust-vexus-lite && npm run build && cd ..

5. 安装依赖 / Install dependencies
   npm install
   pip install -r requirements.txt  # Python 插件依赖

6. 启动服务 / Start service
   node server.js             # 直接运行
   # 或 / or
   pm2 start server.js        # PM2 生产模式
```

---

## 5. Docker 部署 / Docker Deployment

> 📁 `Dockerfile`, `docker-compose.yml`

### 5.1 Docker Compose 快速启动 / Quick Start

```bash
# 编辑配置（必须先完成）
cp config.env.example config.env
vim config.env

# 构建并启动
docker-compose up --build -d

# 查看日志
docker-compose logs -f

# 停止
docker-compose down
```

### 5.2 docker-compose.yml 关键配置 / Key Config

```yaml
# docker-compose.yml (简化示意 / Simplified)
services:
  vcptoolbox:
    build: .
    ports:
      - "5890:5890"       # HTTP/WebSocket 端口
    volumes:
      - ./config.env:/app/config.env          # 主配置（挂载）
      - ./dailynote:/app/dailynote             # 日记数据（持久化）
      - ./Agent:/app/Agent                    # Agent 角色卡（持久化）
      - ./Plugin:/app/Plugin                  # 插件目录（挂载）
    restart: unless-stopped
    environment:
      - NODE_ENV=production
```

### 5.3 Dockerfile 关键内容 / Dockerfile Key Points

```dockerfile
# Dockerfile (简化示意)
FROM node:20-slim

# 安装 Rust（构建向量引擎）
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y

# 安装 Python 依赖
RUN pip install -r requirements.txt

# 构建 Rust N-API
RUN cd rust-vexus-lite && npm run build

# 使用 pm2-runtime 启动
CMD ["pm2-runtime", "server.js"]
```

⚠️ **已知风险 / Known Risk**: 容器默认以 **root 用户**运行，这是为解决卷挂载权限问题的已知权衡。生产环境建议评估安全加固。

### 5.4 数据持久化 / Data Persistence

以下目录需要持久化（容器重启后保留）：

| 目录 / Directory | 内容 / Content | 重要性 / Importance |
|---|---|---|
| `dailynote/` | AI 日记文件 | 🔴 重要（记忆核心）|
| `Agent/` | Agent 角色卡 | 🔴 重要 |
| `TVStxt/` | 外部变量文件 | 🟡 中等 |
| `Plugin/*/config.env` | 插件配置（含密钥）| 🔴 重要 |
| `*.sqlite` | 知识库数据库 | 🔴 重要 |
| `*.index` | 向量索引文件 | 🟡 可重建（慢）|
| `DebugLog/` | 日志存档 | 🟢 低（可丢失）|

---

## 6. 直接运行 / Direct Run

```bash
# 开发模式 / Development mode
node server.js

# PM2 生产模式 / PM2 production mode
npm install -g pm2
pm2 start server.js --name vcptoolbox
pm2 logs vcptoolbox
pm2 restart vcptoolbox

# PM2 开机自启 / PM2 auto-start on boot
pm2 startup
pm2 save
```

---

## 7. 前置依赖 / Prerequisites

| 依赖 / Dependency | 版本 / Version | 用途 / Purpose |
|---|---|---|
| Node.js | 18+ (推荐 20) | 主运行时 / Main runtime |
| npm | 8+ | 依赖管理 / Dependency management |
| Python | 3.9+ | Python 插件运行时 |
| Rust + Cargo | stable | 构建 rust-vexus-lite |
| node-gyp 依赖 | — | N-API 编译环境 |

### 快速检查 / Quick Check

```bash
node --version    # >= 18.0.0
python --version  # >= 3.9.0
rustc --version   # stable channel
cargo --version
```

---

## 8. 故障排查 / Troubleshooting

### 启动失败 / Startup Failure

| 症状 / Symptom | 可能原因 / Possible Cause | 解决方案 / Solution |
|---|---|---|
| `Cannot find module '...'` | npm 依赖未安装 | `npm install` |
| `ENOENT: config.env` | 配置文件不存在 | `cp config.env.example config.env` |
| `rust-vexus-lite/index.node not found` | Rust 组件未构建 | `cd rust-vexus-lite && npm run build` |
| `Port 5890 already in use` | 端口占用 | 修改 `PORT` 或 `kill` 占用进程 |
| `Invalid API key` | Key 配置错误 | 检查 `config.env` 中的 `Key` 字段 |

### 插件不加载 / Plugin Not Loading

```bash
# 检查 manifest 语法
node -e "JSON.parse(require('fs').readFileSync('Plugin/MyPlugin/plugin-manifest.json', 'utf8'))"

# 检查是否被禁用
ls Plugin/MyPlugin/plugin-manifest.json.block  # 存在则被禁用

# 查看启动日志
tail -f DebugLog/archive/$(date +%Y-%m-%d)/Debug/*.log
```

### RAG 检索效果差 / Poor RAG Retrieval

```bash
# 检查向量索引是否存在
ls *.index

# 重建向量索引（慎用：耗时较长）
node rebuild_vector_indexes.js

# 调整 rag_params.json 参数（无需重启）
# Adjust rag_params.json parameters (no restart needed)
```

### 日志位置 / Log Locations

```
DebugLog/
└── archive/
    └── YYYY-MM-DD/
        └── Debug/
            └── *.log   ← 每日日志归档
```

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ⬅️ 上一节 / Prev: [06_frontend](../06_frontend/README.md) | ➡️ 下一节 / Next: [08_ai_agent_guide](../08_ai_agent_guide/README.md)
