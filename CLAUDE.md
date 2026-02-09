# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cocos Creator MCP Server — a Cocos Creator 3.8+ editor extension that exposes an HTTP-based Model Context Protocol (MCP) server. AI assistants (Claude CLI, Cursor, etc.) connect to it to control the Cocos Creator editor via JSON-RPC 2.0 over HTTP.

## Build Commands

```bash
npm install        # Install dependencies (runs preinstall script via scripts/preinstall.js)
npm run build      # Compile TypeScript (tsc): source/ → dist/
npm run watch      # Watch mode for development
```

There is no test runner or linter configured. The project compiles TypeScript with strict mode, targeting ES2017/CommonJS.

## Architecture

### Runtime Environment

This plugin runs inside **Cocos Creator's Node.js environment** (CommonJS). It relies on global objects provided by the editor (`Editor`, `cc`). The `Editor` API is used for IPC, asset DB, scene operations. The `cc` runtime is used in `scene.ts` for direct scene graph manipulation.

### Core Modules

- **`source/main.ts`** — Plugin lifecycle (`load`/`unload`). Initializes `ToolManager` and `MCPServer`. Exports IPC methods called by the panel and editor message system. All `contributions.messages` in `package.json` map to methods exported here.

- **`source/mcp-server.ts`** — HTTP server using Node.js `http` module. Listens on `127.0.0.1:{port}`. Handles three endpoints:
  - `POST /mcp` — JSON-RPC 2.0 MCP protocol (tool discovery via `tools/list`, tool execution via `tools/call`)
  - `GET /health` — Health check
  - `POST /api/*` — Simple API wrapper
  Instantiates all 14 tool category classes and routes tool calls by name prefix.

- **`source/settings.ts`** — Reads/writes server settings to `{ProjectPath}/settings/mcp-server.json`.

- **`source/scene.ts`** — Methods registered under `contributions.scene` in `package.json`. These run in the scene process (not main), with direct access to `cc` runtime for node/component manipulation.

- **`source/types/index.ts`** — All shared TypeScript interfaces (`MCPServerSettings`, `ToolDefinition`, `ToolResponse`, `NodeInfo`, etc.).

### Tool System

Tools live in `source/tools/`, organized into 14 category files. Each class exposes:
- `getTools(): ToolDefinition[]` — Returns tool definitions with JSON Schema input specs
- `execute(toolName: string, args: any): Promise<ToolResponse>` — Executes a tool by name

Tool naming convention: `category_action_name` (e.g., `scene_get_current_scene`, `node_create_node`).

**Tool categories:** scene, node, component, prefab, project, debug, preferences, server, broadcast, assetAdvanced, referenceImage, sceneAdvanced, sceneView, validation.

**`source/tools/tool-manager.ts`** — Manages tool configurations (enable/disable individual tools). Persists to `{ProjectPath}/settings/tool-manager.json`. Supports multiple named configurations.

### Panel UI

- **`source/panels/default/index.ts`** — Vue 3 Composition API panel with two tabs: Server control (start/stop, port, settings) and Tool Management (enable/disable tools by category). Communicates with `main.ts` via `Editor.Message.request('cocos-mcp-server', ...)`.
- Templates in `static/template/default/`, styles in `static/style/default/`.

### Data Flow

```
AI Assistant → HTTP POST /mcp → MCPServer.handleHttpRequest()
  → JSON-RPC parse → tools/call → route to tool category class
  → tool.execute() → Editor API / Editor.Message.request (scene methods)
  → ToolResponse → JSON-RPC response → HTTP response
```

Tools that need scene-graph access call `Editor.Message.request('scene', 'execute-scene-script', ...)` which invokes methods in `scene.ts` running in the scene process.

### Localization

`i18n/en.js` and `i18n/zh.js` provide English and Chinese strings. Referenced via `i18n:cocos-mcp-server.*` keys in `package.json` and panel templates.

## Key Conventions

- Comments and log messages are often in Chinese (Mandarin).
- Tool classes follow a consistent pattern: constructor (no-op), `getTools()` returns definitions array, `execute()` switches on tool name.
- The `package.json` in this project is a **Cocos Creator extension manifest** (not a standard npm package). It defines `panels`, `contributions.messages`, `contributions.scene`, and `contributions.menu`.
- Server binds only to `127.0.0.1` (localhost). Default port is 3000.

## Adding a New Tool

1. Create or modify a tool file in `source/tools/`.
2. Add tool definitions in `getTools()` with JSON Schema `inputSchema`.
3. Handle execution in `execute()` switch.
4. If it's a new category, instantiate the class in `MCPServer.initializeTools()` (`mcp-server.ts`).
5. Run `npm run build`.
