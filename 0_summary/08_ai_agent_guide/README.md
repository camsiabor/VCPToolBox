# 08 AI Coding Agent 快速参考指南 / AI Coding Agent Quick Reference

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **目标读者 / Target**: AI Coding Agents & Human Developers  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [Agent 作业规则 / Agent Working Rules](#1-agent-作业规则--agent-working-rules)
2. [代码库快速定位 / Codebase Quick Navigation](#2-代码库快速定位--codebase-quick-navigation)
3. [常见任务模式 / Common Task Patterns](#3-常见任务模式--common-task-patterns)
4. [反模式警告 / Anti-Patterns Warning](#4-反模式警告--anti-patterns-warning)
5. [调试技巧 / Debug Tips](#5-调试技巧--debug-tips)
6. [变量系统速查 / Variable System Quick Reference](#6-变量系统速查--variable-system-quick-reference)
7. [插件协议速查 / Plugin Protocol Quick Reference](#7-插件协议速查--plugin-protocol-quick-reference)
8. [安全边界 / Security Boundaries](#8-安全边界--security-boundaries)

---

## 1. Agent 作业规则 / Agent Working Rules

### 🔴 绝对禁止 / Absolute Prohibitions

1. **不提交密钥 / Never commit secrets** — `config.env`、`Plugin/*/config.env` 包含真实密钥，永远不能提交
2. **不破坏 manifest / Never break manifests** — `plugin-manifest.json` 是插件加载的唯一契约，字段错误会导致整个插件无法加载
3. **不删除运行时数据 / Never delete runtime data** — `dailynote/`, `image/`, `Plugin/*/state/` 是 AI 记忆，删除不可恢复
4. **不添加 `shell: true` / Never add `shell: true`** — 除非有严格的输入验证和权限控制，否则是严重安全漏洞
5. **不擅自启用 `.block` 插件 / Never enable .block plugins without verification** — 先确认依赖与配置完整

### 🟡 重要提示 / Important Notes

1. **修改文件前备份 / Backup before modifying** — 特别是数据库文件和日记文件
2. **最小化改动 / Minimize changes** — 只改需要改的代码行
3. **CommonJS 模式 / CommonJS mode** — 本项目用 `require()`，不用 `import`（ESM）
4. **扁平根目录 / Flat root** — 核心文件都在根目录，不要假设有 `src/` 层级

---

## 2. 代码库快速定位 / Codebase Quick Navigation

### 按功能查找文件 / Find File by Function

| 我需要... / I need to... | 查看文件 / Look at file | 关键位置 / Key location |
|---|---|---|
| 修改 HTTP 路由 | `server.js` | `:306-813` |
| 修改插件调度逻辑 | `Plugin.js` | `:632-720` |
| 修改 WebSocket 逻辑 | `WebSocketServer.js` | `:400-465` |
| 修改 RAG 检索 | `KnowledgeBaseManager.js` | `:446-621` |
| 修改变量替换 | `modules/messageProcessor.js` | 全文 |
| 修改 Agent 映射 | `modules/agentManager.js` | 全文 |
| 添加管理面板端点 | `routes/adminPanelRoutes.js` | 全文 |
| 修改论坛功能 | `routes/forumApi.js` | 全文 |
| 修改 Rust 向量引擎 | `rust-vexus-lite/src/` | `lib.rs` |
| 修改管理面板前端 | `AdminPanel/js/` | 各功能模块 |

### 关键类与函数速查 / Key Classes & Functions

| 符号 / Symbol | 类型 / Type | 文件 / File | 作用 / Role |
|---|---|---|---|
| `PluginManager` | Class | `Plugin.js` | 插件注册与执行分发 |
| `KnowledgeBaseManager` | Class (Singleton) | `KnowledgeBaseManager.js` | RAG/向量库总控 |
| `ChatCompletionHandler` | Class | `modules/chatCompletionHandler.js` | 对话主流程编排 |
| `AgentManager` | Class | `modules/agentManager.js` | Agent 别名映射 |
| `initialize()` | Function | `WebSocketServer.js` | WS 服务器初始化 |
| `executePlugin()` | Method | `Plugin.js` | 插件执行入口 |
| `handleChatCompletion()` | Method | `modules/chatCompletionHandler.js` | 聊天处理入口 |
| `resolveAgentDir()` | Function | `server.js:27-66` | Agent 目录路径解析 |
| `startServer()` | Function | `server.js` | 最终启动门控 |

---

## 3. 常见任务模式 / Common Task Patterns

### 任务 A: 新建一个同步工具插件

```bash
# 1. 创建目录
mkdir Plugin/MyTool

# 2. 创建 manifest（见 02_plugin_system/README.md § 快速开发）
cat > Plugin/MyTool/plugin-manifest.json << 'EOF'
{
  "manifestVersion": "1.0.0",
  "name": "MyTool",
  "version": "1.0.0",
  "displayName": "我的工具",
  "description": "工具描述（AI 可见）",
  "author": "YourName",
  "pluginType": "synchronous",
  "entryPoint": { "type": "nodejs", "command": "node index.js" },
  "communication": { "protocol": "stdio", "timeout": 30000 },
  "capabilities": {
    "tools": [{ "name": "MyTool", "description": "...", "parameters": {} }]
  }
}
EOF

# 3. 实现逻辑（index.js 从 stdin 读参数，向 stdout 写结果）
# 4. 重启服务器
```

### 任务 B: 新增管理面板 API 端点

```javascript
// routes/adminPanelRoutes.js

// 添加新端点
router.get('/my-new-feature', async (req, res) => {
    try {
        const data = await doSomething();
        res.json({ success: true, data });
    } catch (err) {
        res.status(500).json({ success: false, error: err.message });
    }
});
```

### 任务 C: 调整 RAG 检索参数（无需重启）

```bash
# 编辑 rag_params.json，保存后自动热重载
vim rag_params.json

# 修改示例:
# tagInfluence: 0.3 → 0.5  (提高 Tag 引力)
# minSimilarity: 0.3 → 0.4 (提高最低相似度阈值)
```

### 任务 D: 添加新的 static 占位符

```json
// Plugin/MyStaticPlugin/plugin-manifest.json
{
  "pluginType": "static",
  "capabilities": {
    "systemPromptPlaceholders": [
      { "placeholder": "{{MyCustomData}}", "description": "我的自定义数据" }
    ]
  },
  "cron": "*/15 * * * *"
}
```

然后在 `config.env` 或 Agent 角色卡的 system prompt 中使用 `{{MyCustomData}}`。

### 任务 E: 重建向量索引

```bash
# 警告：耗时操作，勿在生产高峰期执行
# Warning: Time-consuming, do not run during production peak hours

node rebuild_vector_indexes.js

# 仅重建特定日记本索引
node rebuild_tag_index_custom.js
```

---

## 4. 反模式警告 / Anti-Patterns Warning

### ❌ 错误 1: 在根目录假设 `src/` 层级

```javascript
// 错误 / Wrong
require('./src/modules/something');

// 正确 / Correct
require('./modules/something');
```

### ❌ 错误 2: 使用 ESM import

```javascript
// 错误（本项目是 CommonJS）/ Wrong (this project is CommonJS)
import { something } from './module.js';

// 正确 / Correct
const { something } = require('./module.js');
```

### ❌ 错误 3: 在 plugin-manifest.json 中使用错误的 pluginType

```json
// 错误 / Wrong
{ "pluginType": "sync" }

// 正确（必须精确匹配）/ Correct (must exactly match)
{ "pluginType": "synchronous" }
```

有效值 / Valid values: `static`, `messagePreprocessor`, `synchronous`, `asynchronous`, `service`, `hybridservice`

### ❌ 错误 4: 假设工具调用使用 JSON Function Calling

```
// VCPToolBox 使用自定义 VCP 语法，不是 JSON Function Calling
// VCPToolBox uses custom VCP syntax, not JSON Function Calling

// 错误思维 / Wrong assumption:
// AI 输出 {"tool": "GoogleSearch", "params": {...}}

// 正确格式 / Correct format:
// <<<[TOOL_REQUEST]>>>
// toolName:「始」GoogleSearch「末」
// query:「始」search term「末」
// <<<[END_TOOL_REQUEST]>>>
```

### ❌ 错误 5: 直接删改 dailynote/ 文件

```bash
# 危险！这些是 AI 的记忆数据
# Dangerous! These are AI memory data
rm -rf dailynote/MyAgent/

# 正确做法：通过 AdminPanel 或 DailyNoteManager 插件管理
# Correct: Use AdminPanel or DailyNoteManager plugin
```

---

## 5. 调试技巧 / Debug Tips

### 查看运行日志

```bash
# 实时日志
tail -f DebugLog/archive/$(date +%Y-%m-%d)/Debug/*.log

# 或通过 AdminPanel 日志查看器
# http://localhost:5890/AdminPanel/ → 日志 Tab
```

### 测试插件逻辑

```bash
# 手动测试 stdio 插件
echo '{"query": "test input"}' | node Plugin/GoogleSearch/index.js

# 测试 Python 插件
echo '{"input": "test"}' | python Plugin/MyPyPlugin/main.py
```

### 验证 manifest 语法

```bash
node -e "
const fs = require('fs');
const m = JSON.parse(fs.readFileSync('Plugin/MyPlugin/plugin-manifest.json','utf8'));
console.log('✓ manifest valid:', m.name, m.pluginType);
"
```

### 检查插件是否加载

```javascript
// 在 server.js 启动后，访问管理面板查看插件列表
// http://localhost:5890/AdminPanel/ → 插件 Tab

// 或通过 API
GET /admin_api/plugins
Authorization: Basic base64(admin:password)
```

### 常用 grep 模式 / Common grep Patterns

```bash
# 查找所有 VCP 工具调用处理逻辑
grep -r "TOOL_REQUEST" --include="*.js" .

# 查找所有占位符定义
grep -r "systemPromptPlaceholders" Plugin/

# 查找特定插件的调用点
grep -r "executePlugin" --include="*.js" .

# 查找 WebSocket 消息类型
grep -r '"type"' WebSocketServer.js | head -30
```

---

## 6. 变量系统速查 / Variable System Quick Reference

VCPToolBox 支持四类自定义变量，在 system prompt 和 Agent 角色卡中使用：

| 变量类型 / Variable Type | 格式 / Format | 来源 / Source | 优先级 / Priority |
|---|---|---|---|
| `Agent` 变量 | `{{AgentName}}` | `Agent/<name>.txt` 或 `agent_map.json` | 最高 |
| `Tar` 变量 | `{{TarVarName}}` | 运行时动态设置 | 高 |
| `Var` 变量 | `{{VarName}}` | `TVStxt/<name>.txt` 外部文件 | 中 |
| `Sar` 变量 | `{{SarVarName}}` | 静态配置 | 低 |
| 固定变量 | `{{AllCharacterDiariesData}}` 等 | 静态插件注入 | 特殊 |

**变量覆盖顺序 / Override order**:  
`Tar` → `Var` → `Sar` → 固定变量

**示例 / Example**:
```
System prompt 中:
"你是{{Xiaoke}}，当前时间是{{CurrentTime}}，今日日记：{{AllCharacterDiariesData}}"

运行时解析:
- {{Xiaoke}} → 从 Agent/Xiaoke.txt 加载角色卡
- {{CurrentTime}} → 静态插件注入当前时间
- {{AllCharacterDiariesData}} → DailyNote 静态插件注入
```

---

## 7. 插件协议速查 / Plugin Protocol Quick Reference

### stdio 插件输入/输出格式 / stdio Plugin I/O Format

```javascript
// 输入（从 stdin 读取）/ Input (read from stdin)
{
  "param1": "value1",
  "param2": "value2",
  // 内置参数 / Built-in params:
  "_agentName": "Xiaoke",        // 调用此工具的 Agent 名称
  "_conversationId": "xxx",      // 当前对话 ID
  "_config": { /* 合并后的配置 */ }
}

// 输出（写入 stdout）/ Output (write to stdout)
{
  "success": true,               // 是否成功
  "result": "工具执行结果文本",   // 返回给 AI 的结果
  "error": null                  // 错误信息（成功时为 null）
}
```

### 分布式插件注册示例 / Distributed Plugin Registration Example

```json
// plugin-manifest.json
{
  "communication": {
    "protocol": "distributed",
    "nodeCapability": "ComfyUIGen"
  }
}

// 分布式节点注册时声明能力
{
  "type": "register",
  "capabilities": ["ComfyUIGen", "FluxGen"]
}
```

---

## 8. 安全边界 / Security Boundaries

### 代码修改安全检查清单 / Code Change Security Checklist

在提交代码前，检查以下安全边界：

- [ ] 没有将用户输入直接传给 `exec()` 或 `spawn(cmd, {shell: true})`
- [ ] 没有将用户输入直接用于文件路径（路径穿越攻击）
- [ ] 没有在日志中打印 API 密钥或 Bearer Token
- [ ] 新增的 API 端点都有适当的认证中间件
- [ ] 没有将 `config.env` 的内容暴露给未认证请求
- [ ] 插件回调 `callbackId` 使用了加密随机值

### 高风险区域 / High-Risk Areas

| 区域 / Area | 风险 / Risk | 注意 / Note |
|---|---|---|
| `LinuxShellExecutor` | ⚠️ 命令注入 | 13+ 验证器类，有严格输入过滤 |
| `/v1/human/tool` | ⚠️ 越权工具调用 | 确保 Bearer Token 认证 |
| `/plugin-callback/*` | ⚠️ 无认证回调 | 依赖 callbackId 随机性 |
| `expr.static('AdminPanel/')` | ⚠️ 文件泄露 | 确保 adminAuth 在前 |
| Docker root 用户 | ⚠️ 容器逃逸风险 | 已知权衡，生产需加固 |

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ⬅️ 上一节 / Prev: [07_config_deploy](../07_config_deploy/README.md)
