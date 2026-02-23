# VCP Ecosystem — Related Repositories

**Last updated:** 2026-02-23  
**Scope:** AI Coding Agent awareness document — read-only reference, no code changes

---

## Purpose

This document gives every AI Coding Agent working in this repository a clear picture of the
**broader VCP ecosystem**: which repositories exist, how they relate to each other, and what rules
govern work across them.

**Key rule for AI agents:** This fork (`camsiabor/*`) exists **solely** to provide understanding,
documentation, and analysis.  **Do NOT modify application code in this fork.**  All code changes
must originate from — or be synchronized back to — the upstream source repos owned by
[lioensky](https://github.com/lioensky).

---

## Repository Map

### Fork repositories (camsiabor — this organisation)

| Repository | URL | Role |
|------------|-----|------|
| **VCPToolBox** *(this repo)* | https://github.com/camsiabor/VCPToolBox | Core AI middleware, plugin runtime, RAG/memory system |
| **VCPDistributedServer** | https://github.com/camsiabor/VCPDistributedServer | Distributed node / WebSocket bridge server |
| **VCPChat** | https://github.com/camsiabor/VCPChat | Chat front-end client application |

### Upstream source repositories (lioensky — origin)

| Repository | URL | Role |
|------------|-----|------|
| **VCPToolBox** | https://github.com/lioensky/VCPToolBox | Upstream source for this repo |
| **VCPDistributedServer** | https://github.com/lioensky/VCPDistributedServer | Upstream source for VCPDistributedServer |
| **VCPChat** | https://github.com/lioensky/VCPChat | Upstream source for VCPChat |

---

## Fork vs. Upstream Relationship

```
lioensky/VCPToolBox  ──────fork──────►  camsiabor/VCPToolBox   (this repo)
lioensky/VCPDistributedServer ──fork──► camsiabor/VCPDistributedServer
lioensky/VCPChat      ──────fork──────► camsiabor/VCPChat
```

- The **camsiabor** forks shadow the **lioensky** originals 1-to-1.
- When the upstream changes, **pull from the upstream** (`git fetch upstream && git merge upstream/main`)
  to keep the fork in sync.  Do this before starting any analysis or documentation work.
- The forks should never diverge on application code; divergence is a signal to re-sync.

---

## Inter-Repository Architecture

```
┌──────────────────────────────────────────────────────────┐
│                      VCPChat                             │
│  (chat UI, user-facing front end)                        │
│  Port: configured per deployment                         │
└───────────────────────┬──────────────────────────────────┘
                        │ HTTP / SSE  (OpenAI-compatible API)
                        ▼
┌──────────────────────────────────────────────────────────┐
│                    VCPToolBox  ◄── THIS REPO             │
│  (core middleware: plugin runtime, RAG, memory,          │
│   tool execution, admin panel)                           │
│  Port: VCP_PORT (default 6005)                           │
└───────────────────────┬──────────────────────────────────┘
                        │ WebSocket  (tool execution bridge)
                        ▼
┌──────────────────────────────────────────────────────────┐
│               VCPDistributedServer                       │
│  (optional: remote tool nodes, file fetching,            │
│   cross-machine execution bridge)                        │
│  Port: WebSocket port configured in config.env           │
└──────────────────────────────────────────────────────────┘
```

**Data flow summary:**

1. **VCPChat** sends OpenAI-compatible chat completions requests to **VCPToolBox**.
2. **VCPToolBox** processes the request: injects system prompts, runs plugins, executes tools,
   queries the RAG/memory system, and streams the response back to VCPChat.
3. When a tool execution targets a remote node, **VCPToolBox** forwards the call over WebSocket to
   **VCPDistributedServer**, which runs the tool on the remote machine and streams results back.

---

## AI Agent Rules for This Ecosystem

### ✅ What agents SHOULD do in this fork

- Read source code to build understanding and documentation.
- Add or update files under `docs/` to improve agent-facing knowledge.
- Maintain `AGENTS.md` files at each directory level.
- Sync from upstream (`lioensky/*`) when the fork is behind.
- File issues or document findings about bugs / improvements found during analysis.

### ❌ What agents MUST NOT do in this fork

- **Do not modify application source code** (`.js`, `.py`, `.rs`, plugin files, config templates,
  etc.) in this fork unless explicitly instructed by a human maintainer.
- Do not push code changes that deviate from the upstream source code.
- Do not commit secrets, credentials, or personally identifiable information.
- Do not create files outside `docs/` or `AGENTS.md` files without explicit human instruction.

### 🔄 How to sync with upstream (when needed)

```bash
# One-time: add the upstream remote (run once per clone)
git remote add upstream https://github.com/lioensky/VCPToolBox.git

# Check current sync status
git fetch upstream
git log HEAD..upstream/main --oneline   # commits in upstream not yet in fork

# Merge upstream changes
git checkout main   # or the active branch
git merge upstream/main
git push origin main
```

Repeat the same pattern for `VCPDistributedServer` and `VCPChat` in their respective clone directories.

---

## Repository Summaries

### VCPToolBox (core middleware)

- **Tech stack:** Node.js (ESM/CJS mixed), Python plugins, Rust N-API (vector engine)
- **Key entry points:** `server.js` (HTTP/SSE), `Plugin.js` (plugin runtime),
  `WebSocketServer.js` (distributed bridge), `KnowledgeBaseManager.js` (RAG)
- **Plugin count:** ~79 active plugins across Node/Python/Rust
- **Memory system:** TagMemo Wave Algorithm (V3.7) + SQLite + Rust vector index
- **Admin UI:** embedded static SPA at `AdminPanel/`
- **Full docs:** see [`docs/DOCUMENTATION_INDEX.md`](./DOCUMENTATION_INDEX.md)

### VCPDistributedServer

- **Purpose:** Acts as a remote execution node; VCPToolBox connects to it via WebSocket to
  delegate tool calls to machines where those tools are physically present (e.g., desktop
  automation, camera, local shell).
- **Protocol:** Custom VCP WebSocket tool-bridge protocol (mirrors `WebSocketServer.js` in
  VCPToolBox).
- **When needed:** Only required when running tools on a machine separate from the VCPToolBox
  server.

### VCPChat

- **Purpose:** A dedicated chat front-end that speaks the OpenAI-compatible API exposed by
  VCPToolBox.
- **Connection:** Configured with `VCP_SERVER_URL` pointing at VCPToolBox.
- **Features:** Supports streaming, tool-call display, multi-agent conversation, and VCP-specific
  rendering (TagMemo visualization, media playback).
- **Setup guide:** see [`README For VCPChat.md`](../README%20For%20VCPChat.md) in this repo.

---

## Maintenance Checklist for AI Agents

When starting a new session in this repository:

- [ ] Check if the fork is behind upstream: `git fetch upstream && git log HEAD..upstream/main --oneline`
- [ ] If behind, sync before doing any analysis work.
- [ ] Read [`docs/DOCUMENTATION_INDEX.md`](./DOCUMENTATION_INDEX.md) for current system state.
- [ ] Confirm you are on the correct branch (`git branch --show-current`).
- [ ] Remember: **analysis and docs only** — no application code changes without human approval.
