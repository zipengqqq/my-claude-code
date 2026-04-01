# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是 `@anthropic-ai/claude-code` CLI 工具的**还原 TypeScript 源码树**，从 npm 包的 source map 中提取。它是一个基于 **TypeScript + React (Ink)** 的终端 AI 编程助手，运行在 **Bun** 上。源码仅供学习研究，版权归 Anthropic 所有。

## 开发命令

```bash
bun install            # 安装依赖（需要 Bun >= 1.3.5，Node.js >= 24）
bun run dev            # 启动 CLI（运行 src/dev-entry.ts）
bun run start          # dev 的别名
bun run version        # 验证 CLI 启动并打印版本号
```

**没有配置测试套件、linter 或格式化工具。** 手动验证方式：用 `bun run dev` 启动，用 `bun run version` 冒烟测试，然后手动操作变更涉及的路径。

## 构建配置

- **运行时**：Bun，使用 `bun:bundle` 实现编译时特性开关和死代码消除
- **TypeScript**：`strict: false`，ESNext target/module，`moduleResolution: "bundler"`，`jsx: "react-jsx"`
- **路径别名**：`src/*` → `./src/*`（导入时直接用 `src/...` 路径）
- **纯 ESM**（`package.json` 中 `"type": "module"`）
- **所有导入使用 `.js` 扩展名**（尽管实际是 `.ts` 文件，这是 Bun 的约定）：`import { X } from './utils/foo.js'`
- **没有 ESLint/Prettier 配置** —— 请匹配周围文件的代码风格

## 编码规范

- 很多文件不写分号；使用单引号
- 变量/函数用 `camelCase`，React 组件和管理类用 `PascalCase`
- 命令文件夹用 `kebab-case`（如 `src/commands/install-slack-app/`）
- `react-jsx` 转换 —— 不需要显式 `import React from 'react'`
- 部分文件有 `// biome-ignore-all assist/source/organizeImports` 注释 —— 不要重排这些文件的导入顺序
- 这是还原的源码树 —— 偏好最小化、可审计的改动

## 架构

### 启动流程

```
dev-entry.ts → entrypoints/cli.tsx → entrypoints/init.ts → setup.ts → main.tsx
```

- `dev-entry.ts`：验证工作区完整性，扫描缺失的导入，转发到 CLI
- `entrypoints/cli.tsx`：处理快速路径参数（`--version`、`--help`），动态导入以加速启动
- `entrypoints/init.ts`：带记忆化的初始化 —— 配置、环境变量、OpenTelemetry、认证、代理
- `setup.ts`：设置工作目录、hooks 配置、worktree 创建、会话记忆、后台任务
- `main.tsx`：连接 React/Ink 终端 UI，渲染 REPL

### 核心模块

| 模块 | 职责 |
|------|------|
| `QueryEngine.ts` | 核心编排引擎 —— 处理查询、管理工具调用、消息压缩、上下文窗口 |
| `query.ts` | 实际 API 调用，处理流式传输、用量追踪、通过异步生成器实现重试逻辑 |
| `Tool.ts` | 工具基础抽象 —— 定义 `Tool` 接口、`Tools` 类型、`ToolUseContext` |
| `tools.ts` | 工具注册表 —— 组装全部 53+ 个工具，使用 `feature()` 标志做死代码消除 |
| `commands.ts` | 斜杠命令注册表 —— 87+ 个命令，三种类型：`prompt`、`local`、`local-jsx` |
| `Task.ts` | 任务系统类型 —— 状态、任务句柄、代理协调 |
| `App.tsx` | 顶层 React 组件，包裹交互式会话 |

### 关键子系统

| 目录 | 用途 |
|------|------|
| `src/tools/` | 53+ 个工具实现（Bash、FileEdit、Agent、MCP 等） |
| `src/commands/` | 87+ 个斜杠命令 |
| `src/services/api/` | Anthropic API 客户端、认证、重试逻辑 |
| `src/services/mcp/` | 模型上下文协议（MCP）—— 连接管理器、传输层、OAuth |
| `src/services/compact/` | 消息/上下文压缩服务 |
| `src/services/analytics/` | 事件追踪（GrowthBook + OpenTelemetry） |
| `src/components/` | 148+ 个终端 UI 组件（React + Ink） |
| `src/hooks/` | 87+ 个自定义 React hooks |
| `src/state/` | 应用状态管理（React context + store） |

### 核心抽象

**特性开关**（`feature()` 来自 `bun:bundle`）：约 50 个编译时标志用于死代码消除（如 `BUDDY`、`KAIROS`、`ULTRAPLAN`、`COORDINATOR_MODE`、`BRIDGE_MODE`）。外部构建会裁剪大部分功能。

**用户类型**（`USER_TYPE` 环境变量）：`'ant'`（Anthropic 内部，全功能）vs `'external'`（外部用户，裁剪版）。整个代码库有 200+ 处条件检查。

**GrowthBook**：远程 A/B 测试和运行时特性开关（如 `tengu_kairos`、`tengu_ultraplan_model`）。

**MACRO 系统**：构建时常量（`MACRO.VERSION`、`MACRO.BUILD_TIME`）通过 `globalThis.MACRO` 注入。

**异步生成器流式传输**：核心架构模式 —— `async function*` 生成器将 API 响应、工具结果和消息通过查询管道进行流式传输。`yield*` 委托给子生成器。

**命令类型**：`Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)` —— 鉴别联合类型，`type` 字段决定命令的执行方式。

**权限系统**：三层模型（`alwaysAllow`、`alwaysDeny`、`alwaysAsk`），多种模式（default、bypass、plan）。

## 特性开关控制的子系统

以下子系统由编译时特性开关控制，在外部构建中不可用：

- **`src/buddy/`** —— 虚拟宠物系统（`BUDDY`）
- **`src/assistant/` + `src/proactive/` + `src/services/autoDream/`** —— 持久助手模式（`KAIROS`）
- **`src/coordinator/`** —— 多代理编排（`COORDINATOR_MODE`）
- **`src/bridge/`** —— WebSocket 远程控制（`BRIDGE_MODE`）
- **`src/voice/`** —— 语音交互（`VOICE_MODE`）
- **`src/vim/`** —— Vim 按键模拟（始终激活）
