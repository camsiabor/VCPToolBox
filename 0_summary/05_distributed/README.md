# 05 分布式架构 / Distributed Architecture

> **所属目录 / Parent**: [0_summary/INDEX.md](../INDEX.md)  
> **核心文件 / Core Files**: `WebSocketServer.js`, `FileFetcherServer.js`  
> **版本 / Version**: VCP 6.4 · 2026-02-23

---

## 目录 / Table of Contents

1. [架构概览 / Architecture Overview](#1-架构概览--architecture-overview)
2. [六种 WebSocket 客户端类型 / Six WS Client Types](#2-六种-websocket-客户端类型--six-ws-client-types)
3. [节点注册流程 / Node Registration](#3-节点注册流程--node-registration)
4. [分布式工具执行 / Distributed Tool Execution](#4-分布式工具执行--distributed-tool-execution)
5. [文件传输机制 / File Transfer](#5-文件传输机制--file-transfer)
6. [ChromeBridge 集成 / ChromeBridge Integration](#6-chromebridge-集成--chromebridge-integration)
7. [心跳与连接管理 / Heartbeat & Connection Management](#7-心跳与连接管理--heartbeat--connection-management)
8. [消息协议 / Message Protocol](#8-消息协议--message-protocol)

---

## 1. 架构概览 / Architecture Overview

### 中文

VCP 分布式架构采用**星型网络拓扑**，由一个**主服务器 (VCPToolBox)** 和若干**分布式节点 (VCPDistributedServer)** 组成。主服务器通过 WebSocket 与所有节点保持长连接，实现远程工具调度和文件传输。

### English

VCP's distributed architecture uses a **star network topology** with one **master server (VCPToolBox)** and several **distributed nodes (VCPDistributedServer)**. The master server maintains persistent WebSocket connections with all nodes, enabling remote tool dispatching and file transfer.

```
                    ┌───────────────────────────┐
                    │       VCP 主服务器          │
                    │    (WebSocketServer.js)     │
                    │                             │
                    │  - 路由与调度 / Routing     │
                    │  - 插件管理 / Plugin mgmt   │
                    │  - 文件获取 / File fetch    │
                    └─────────────┬───────────────┘
                                  │  WebSocket
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │  分布式节点 A    │ │  分布式节点 B    │ │ ChromeObserver  │
    │  (GPU 服务器)   │ │  (文件服务器)   │ │  (浏览器扩展)   │
    │  VCPDistributed │ │  VCPDistributed │ │  VCPChrome      │
    └─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## 2. 六种 WebSocket 客户端类型 / Six WS Client Types

通过 WebSocket 连接 URL 路径区分客户端类型：

### 2.1 `VCPLog` — 日志客户端

```
连接路径 / Connect path: /VCPlog/VCP_Key=<key>
存储 / Storage: clients Map
用途 / Usage: 接收服务器广播的 VCP 工具调用日志
```

### 2.2 `VCPInfo` — 信息推送客户端

```
连接路径 / Connect path: /vcpinfo/VCP_Key=<key>
存储 / Storage: clients Map
专用函数 / Dedicated function: broadcastVCPInfo(data)
用途 / Usage: 接收系统状态、插件状态等推送信息
```

### 2.3 `VCPDistributedServer` — 分布式工具节点

```
连接路径 / Connect path: /vcp-distributed/VCP_Key=<key>
存储 / Storage: distributedClients Map
用途 / Usage: 承载可被主服务器远程调度的工具插件
```

**节点注册格式 / Node registration format**:
```json
{
  "type": "register",
  "nodeId": "gpu-server-01",
  "capabilities": ["ComfyUIGen", "FluxGen", "NovelAIGen"],
  "metadata": { "gpuModel": "RTX 4090", "vram": "24GB" }
}
```

### 2.4 `VCPBrowser` / `ChromeObserver` — 浏览器观察者

```
连接路径 / Connect path: /vcp-chrome-<type>/VCP_Key=<key>
用途 / Usage: VCPChrome 扩展与服务器的双向通道
```

### 2.5 `VCPChat` — 聊天客户端推送

```
连接路径 / Connect path: /vcpchat/VCP_Key=<key>
用途 / Usage: 向 VCPChat 前端推送实时消息
```

### 2.6 `VCPAdmin` — 管理客户端

```
连接路径 / Connect path: /vcpadmin/VCP_Key=<key>
用途 / Usage: 管理面板的实时状态推送（插件状态、系统指标等）
```

---

## 3. 节点注册流程 / Node Registration

```
分布式节点启动 / Distributed node startup
         │
         ▼
建立 WebSocket 连接 / Establish WS connection
  ws://master:5890/vcp-distributed/VCP_Key=<key>
         │
         ▼
发送注册消息 / Send registration message
  {
    "type": "register",
    "nodeId": "unique-node-id",
    "capabilities": ["PluginA", "PluginB"],
    "metadata": { ... }
  }
         │
         ▼
主服务器验证并记录节点 / Master validates and records node
  distributedClients.set(nodeId, { ws, capabilities, metadata })
         │
         ▼
主服务器确认注册 / Master confirms registration
  { "type": "register_ack", "status": "ok", "nodeId": "..." }
         │
         ▼
节点就绪，等待工具调度 / Node ready, waiting for tool dispatch
```

---

## 4. 分布式工具执行 / Distributed Tool Execution

当 AI 调用的工具插件带有 `"protocol": "distributed"` 时，执行流如下：

```
AI 输出 VCP 工具调用块
  <<<[TOOL_REQUEST]>>>
  toolName:「始」ComfyUIGen「末」
  <<<[END_TOOL_REQUEST]>>>
         │
         ▼
PluginManager 检测到 distributed 协议
         │
         ▼
WebSocketServer.dispatchToolToNode(toolName, params)
  ├── 查找具备该工具能力的节点 / Find node with capability
  ├── 选择节点（负载均衡 / Load balance）
  └── 发送执行请求 / Send execution request:
      {
        "type": "tool_execute",
        "requestId": "uuid-xxx",
        "toolName": "ComfyUIGen",
        "params": { ... }
      }
         │ WebSocket
         ▼
分布式节点 / Distributed node
  ├── 本地 spawn 子进程执行工具
  ├── 等待工具完成
  └── 发送结果回主服务器 / Send result back:
      {
        "type": "tool_result",
        "requestId": "uuid-xxx",
        "result": { ... },
        "success": true
      }
         │ WebSocket
         ▼
主服务器收到结果 / Master receives result
  ├── 通过 requestId 匹配等待的 Promise
  └── resolve(result) → 注入 AI 上下文
```

---

## 5. 文件传输机制 / File Transfer

> 📁 `FileFetcherServer.js`

### 问题背景 / Background

分布式节点执行工具后（如图像生成），产生的文件存在**节点本地**，主服务器和 AI 客户端无法直接访问。

### 解决方案 / Solution

`FileFetcherServer.js` 实现了**跨节点文件获取**机制：

```
AI 客户端请求图片 / AI client requests image
  GET /pw=key/images/generated_123.png (主服务器)
         │
         ▼
主服务器检查本地缓存 / Check local cache
  ├── 命中 → 直接返回 / Cache hit → Return directly
  └── 未命中 / Cache miss →
         ▼
向分布式节点发送文件获取请求 / Send file fetch request to node
  WS: { "type": "file_request", "filename": "generated_123.png" }
         │
         ▼
节点读取文件并分块传输 / Node reads file and streams chunks
  WS: { "type": "file_chunk", "data": "base64...", "seq": 1 }
  WS: { "type": "file_chunk", "data": "base64...", "seq": 2 }
  WS: { "type": "file_end", "filename": "..." }
         │
         ▼
主服务器重组文件并缓存 / Master reassembles and caches
         │
         ▼
返回给 AI 客户端 / Return to AI client
```

---

## 6. ChromeBridge 集成 / ChromeBridge Integration

> 📁 `VCPChrome/` (浏览器扩展), `WebSocketServer.js` (服务端处理)

### 功能 / Function

ChromeBridge 允许 AI Agent 通过 WebSocket **远程控制浏览器**，实现：
- 📸 截取当前页面内容（转换为 AI 可读 markdown）
- 🔗 获取页面链接列表
- ⌨️ 模拟键鼠操作
- 📋 读取剪贴板内容

### 连接模式 / Connection Mode

```
VCPChrome 扩展 (浏览器) ←── WebSocket ──► VCPToolBox (服务器)
                               /vcp-chrome-observer/VCP_Key=<key>
                               /vcp-chrome-control/VCP_Key=<key>
```

### 典型场景 / Typical Scenario

```
AI 调用 ChromeBridge 工具:
  <<<[TOOL_REQUEST]>>>
  toolName:「始」ChromeBridge「末」
  action:「始」getPageContent「末」
  <<<[END_TOOL_REQUEST]>>>

服务器通过 WS 发送命令给浏览器扩展
浏览器扩展执行页面内容采集
结果通过 WS 回传服务器
服务器将 Markdown 化的页面内容注入 AI 上下文
```

---

## 7. 心跳与连接管理 / Heartbeat & Connection Management

```javascript
// WebSocketServer.js (推断实现 / Inferred implementation)

// 客户端发送心跳 / Client sends heartbeat
ws.send(JSON.stringify({ type: 'ping' }));

// 服务器响应 / Server responds
if (message.type === 'ping') {
    ws.send(JSON.stringify({ type: 'pong' }));
}

// 断开处理 / Disconnect handling
ws.on('close', () => {
    distributedClients.delete(nodeId);
    // 清理该节点的所有等待 Promise
    // 如有进行中的工具调用，返回错误
});
```

| 配置 / Config | 默认值 / Default | 说明 / Description |
|---|---|---|
| 心跳间隔 / Heartbeat interval | 🔍 推断 30s | 客户端主动 ping |
| 连接超时 / Connection timeout | 🔍 推断 60s | 超时自动断开 |
| 重连策略 / Reconnect strategy | 节点自行实现 / Node-side | 指数退避推荐 / Exponential backoff recommended |

---

## 8. 消息协议 / Message Protocol

所有 WebSocket 消息均为 JSON 格式。

### 消息类型汇总 / Message Type Summary

| 方向 / Direction | 类型 / Type | 说明 / Description |
|---|---|---|
| 节点→主 / Node→Master | `register` | 节点注册 |
| 主→节点 / Master→Node | `register_ack` | 注册确认 |
| 主→节点 / Master→Node | `tool_execute` | 执行工具请求 |
| 节点→主 / Node→Master | `tool_result` | 工具执行结果 |
| 主→节点 / Master→Node | `file_request` | 请求文件 |
| 节点→主 / Node→Master | `file_chunk` | 文件数据块 |
| 节点→主 / Node→Master | `file_end` | 文件传输完成 |
| 双向 / Bidirectional | `ping` / `pong` | 心跳 |
| 主→客户端 / Master→Client | `vcp_log` | 工具调用日志 |
| 主→客户端 / Master→Client | `vcp_info` | 系统信息推送 |

### 消息格式示例 / Message Format Examples

```json
// 工具执行请求 / Tool execute request
{
  "type": "tool_execute",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "toolName": "ComfyUIGen",
  "params": {
    "prompt": "a beautiful sunset over mountains",
    "width": 1024,
    "height": 768
  }
}

// 工具执行结果 / Tool execute result
{
  "type": "tool_result",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "success": true,
  "result": {
    "imageUrl": "/pw=key/images/comfyui_output_001.png",
    "generationTime": 8.5
  }
}

// VCP 工具调用日志 / VCP tool call log
{
  "type": "vcp_log",
  "timestamp": "2026-02-23T02:54:27.656Z",
  "agent": "Xiaoke",
  "tool": "GoogleSearch",
  "params": { "query": "..." },
  "result": "...",
  "duration": 1234
}
```

---

> ⬆️ 返回 / Back: [INDEX.md](../INDEX.md) | ⬅️ 上一节 / Prev: [04_api_routes](../04_api_routes/README.md) | ➡️ 下一节 / Next: [06_frontend](../06_frontend/README.md)
