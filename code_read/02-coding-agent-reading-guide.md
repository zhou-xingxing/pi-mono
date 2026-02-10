# packages/coding-agent 源码阅读指南

## 项目概述

pi-coding-agent 是整个 Pi Monorepo 的核心产品，是一个极简但高度可扩展的终端编码代理 CLI 工具。它基于 `pi-agent-core` 和 `pi-tui` 构建，提供了交互式 TUI 界面、会话管理、扩展系统等功能。

---

## 源码阅读顺序（按调用链和依赖关系）

### 第一阶段：入口与启动流程（建立整体认知）

| 顺序 | 文件 | 说明 |
|:---:|------|------|
| 1 | `src/cli.ts` | CLI入口，仅设置进程标题并调用 main() |
| 2 | `src/main.ts` | **核心入口**，参数解析、模式分发、AgentSession创建 |
| 3 | `src/config.ts` | 配置常量、版本号、路径定义（getAgentDir, VERSION） |
| 4 | `src/index.ts` | 包的公开API导出（了解整体功能暴露） |

**关键理解**：从 cli.ts → main.ts 的调用链，了解程序如何根据参数进入不同模式（interactive/print/rpc）。

---

### 第二阶段：核心会话层（理解Agent生命周期）

| 顺序 | 文件 | 说明 |
|:---:|------|------|
| 5 | `src/core/agent-session.ts` | **核心类 AgentSession**，提供 prompt() / continue() 方法，是上层与Agent运行时的桥梁 |
| 6 | `src/core/session-manager.ts` | 会话持久化管理，JSONL格式、分支管理、会话恢复 |
| 7 | `src/core/settings-manager.ts` | 设置管理（思考级别、重试策略、图像设置、终端设置） |
| 8 | `src/core/auth-storage.ts` | 认证凭证存储（API Key / OAuth 凭证管理） |
| 9 | `src/core/model-registry.ts` | 模型注册表，支持多Provider的模型发现和解析 |
| 10 | `src/core/model-resolver.ts` | 模型解析逻辑，默认模型配置 |

**关键理解**：AgentSession 如何包装 pi-agent-core 的 Agent 类，添加会话持久化和扩展支持。

---

### 第三阶段：扩展系统（理解插件机制）

| 顺序 | 文件 | 说明 |
|:---:|------|------|
| 11 | `src/core/extensions/types.ts` | **类型定义中心**，ExtensionAPI、事件类型、工具定义类型 |
| 12 | `src/core/extensions/index.ts` | 扩展模块导出，类型守卫函数（is*ToolResult） |
| 13 | `src/core/extensions/loader.ts` | 扩展加载器，TS/JS文件的发现、编译（jiti）、加载 |
| 14 | `src/core/extensions/runner.ts` | **扩展运行器**，ExtensionRunner 类，事件分发、生命周期管理 |
| 15 | `src/core/extensions/wrapper.ts` | 工具包装器，扩展如何拦截/修改工具调用和结果 |

**关键理解**：扩展系统的事件驱动模型，以及工具拦截的实现机制。

---

### 第四阶段：内置工具实现（理解工具如何工作）

| 顺序 | 文件 | 说明 |
|:---:|------|------|
| 16 | `src/core/tools/index.ts` | 工具导出，codingTools 数组 |
| 17 | `src/core/tools/read.ts` | **读文件工具**，带行范围支持和截断 |
| 18 | `src/core/tools/write.ts` | **写文件工具**，支持创建新文件 |
| 19 | `src/core/tools/edit.ts` | **编辑工具核心**，字符串替换和diff应用 |
| 20 | `src/core/tools/edit-diff.ts` | 统一diff格式解析和应用 |
| 21 | `src/core/tools/bash.ts` | **Bash执行工具**，命令执行和安全检查 |
| 22 | `src/core/tools/grep.ts` | 文本搜索工具（基于ripgrep） |
| 23 | `src/core/tools/ls.ts` | 目录列表工具 |
| 24 | `src/core/tools/find.ts` | 文件查找工具（基于fd） |
| 25 | `src/core/tools/truncate.ts` | 输出截断逻辑 |
| 26 | `src/core/tools/path-utils.ts` | 路径处理工具函数 |
| 27 | `src/core/bash-executor.ts` | Bash命令执行实现，spawn/wait逻辑 |

**关键理解**：每个工具的 execute 方法签名，以及它们如何与 ExtensionRunner 配合。

---

### 第五阶段：资源管理（技能、提示、包）

| 顺序 | 文件 | 说明 |
|:---:|------|------|
| 28 | `src/core/skills.ts` | **Skills系统**，Agent Skills标准实现，SKILL.md解析 |
| 29 | `src/core/prompt-templates.ts` | 提示模板加载和管理 |
| 30 | `src/core/resource-loader.ts` | **统一资源加载器**，扩展/技能/提示/主题的加载协调 |
| 31 | `src/core/package-manager.ts` | **Pi包管理**，npm/git包的安装、更新、移除 |

**关键理解**：Skill和Prompt Template的文件格式，以及 Pi Package 的 manifest 结构。

---

### 第六阶段：上下文压缩与会话优化

| 顺序 | 文件 | 说明 |
|:---:|------|------|
| 32 | `src/core/compaction/index.ts` | 压缩功能导出 |
| 33 | `src/core/compaction/compaction.ts` | **上下文压缩核心**，智能摘要算法 |
| 34 | `src/core/compaction/branch-summarization.ts` | 分支摘要生成，多轮对话压缩 |
| 35 | `src/core/compaction/utils.ts` | 压缩工具函数 |

**关键理解**：如何处理长会话的上下文窗口限制。

---

### 第七阶段：交互模式与TUI（可选，深入了解UI）

| 顺序 | 文件 | 说明 |
|:---:|------|------|
| 36 | `src/modes/index.ts` | 运行模式导出（interactive/print/rpc） |
| 37 | `src/modes/interactive/interactive-mode.ts` | **交互模式主逻辑**，TUI初始化、事件循环 |
| 38 | `src/modes/interactive/components/index.ts` | UI组件导出 |
| 39 | `src/modes/interactive/components/model-selector.ts` | 模型选择器组件 |
| 40 | `src/modes/interactive/components/session-selector.ts` | 会话选择器组件 |
| 41 | `src/modes/interactive/theme/theme.ts` | 主题系统 |

**关键理解**：InteractiveMode 如何整合 pi-tui 和 AgentSession。

---

### 第八阶段：CLI工具和杂项

| 顺序 | 文件 | 说明 |
|:---:|------|------|
| 42 | `src/cli/args.ts` | **命令行参数解析**，所有flags和选项定义 |
| 43 | `src/core/slash-commands.ts` | **Slash命令处理**，/login /model /settings 等 |
| 44 | `src/core/system-prompt.ts` | **系统提示构建**，合并默认提示、AGENTS.md、Skills |
| 45 | `src/core/messages.ts` | 消息格式转换（AgentMessage ↔ LLM Message） |
| 46 | `src/core/diagnostics.ts` | 诊断信息收集和显示 |
| 47 | `src/core/timings.ts` | 性能计时 |
| 48 | `src/core/exec.ts` | 进程执行工具 |

---

## 快速阅读路径（2-3小时）

如果只想快速理解核心架构，按这个精简顺序：

```
1. cli.ts → main.ts → config.ts
   └─ 了解程序入口和启动流程

2. core/agent-session.ts (重点★)
   └─ 理解 AgentSession.prompt() 和 continue() 方法

3. core/extensions/types.ts + runner.ts
   └─ 理解扩展系统的类型定义和事件分发

4. core/tools/read.ts + edit.ts + bash.ts
   └─ 理解内置工具的实现模式

5. core/session-manager.ts
   └─ 理解会话持久化和分支管理

6. modes/interactive/interactive-mode.ts (概览)
   └─ 了解交互模式如何整合各组件
```

这样可以建立起从 "CLI入口 → Agent会话 → 扩展系统 → 工具执行 → 持久化 → UI" 的完整认知链条。

---

## 关键调用链追踪

### 1. 启动流程

```
cli.ts#main()
  └── main.ts#main()
      ├── parseArgs() 解析参数
      ├── resolveConfig() 解析配置
      ├── createAgentSession() 创建会话
      └── 根据模式分发:
          ├── InteractiveMode.run() 交互模式
          ├── runPrintMode() 打印模式
          └── runRpcMode() RPC模式
```

### 2. 一次用户输入的处理流程

```
InteractiveMode
  └── Editor.onSubmit(text)
      └── AgentSession.prompt(text)
          ├── ExtensionRunner.emit('input') 扩展预处理
          ├── Agent.loop() (来自 pi-agent-core)
          │   ├── stream() 调用 LLM (来自 pi-ai)
          │   ├── 处理 tool_calls
          │   └── 执行 tools
          │       └── ExtensionRunner.emit('tool_call')
          │           └── 内置 Tool.execute()
          └── 保存到 SessionManager
```

### 3. 扩展加载流程

```
main.ts
  └── discoverAndLoadExtensions()
      ├── loader.ts#loadExtension() 加载单个扩展
      │   ├── 使用 jiti 编译 TS
      │   └── 执行扩展工厂函数
      └── runner.ts#ExtensionRunner 管理扩展实例
```

---

## 目录结构速览

```
packages/coding-agent/src/
├── cli.ts                    # 入口
├── main.ts                   # 主逻辑
├── config.ts                 # 配置
├── index.ts                  # 公开API
├── core/                     # 核心逻辑
│   ├── agent-session.ts      # Agent会话（核心）
│   ├── session-manager.ts    # 会话持久化
│   ├── settings-manager.ts   # 设置管理
│   ├── auth-storage.ts       # 认证存储
│   ├── model-registry.ts     # 模型注册表
│   ├── extensions/           # 扩展系统
│   │   ├── types.ts          # 类型定义
│   │   ├── loader.ts         # 扩展加载
│   │   ├── runner.ts         # 扩展运行器
│   │   └── wrapper.ts        # 工具包装
│   ├── tools/                # 内置工具
│   │   ├── read.ts
│   │   ├── write.ts
│   │   ├── edit.ts
│   │   ├── bash.ts
│   │   ├── grep.ts
│   │   ├── ls.ts
│   │   └── find.ts
│   ├── compaction/           # 上下文压缩
│   ├── skills.ts             # Skills系统
│   ├── prompt-templates.ts   # 提示模板
│   ├── resource-loader.ts    # 资源加载
│   ├── package-manager.ts    # 包管理
│   └── ...
├── modes/                    # 运行模式
│   ├── interactive/          # 交互模式（TUI）
│   │   ├── interactive-mode.ts
│   │   ├── components/       # UI组件
│   │   └── theme/            # 主题
│   ├── rpc/                  # RPC模式
│   └── print/                # 打印模式
├── cli/                      # CLI辅助
│   └── args.ts               # 参数解析
└── utils/                    # 工具函数
    └── ...
```

---

## 下一步分析建议

1. **深入扩展系统**：阅读 `examples/extensions/` 中的示例，对照 `core/extensions/` 的源码
2. **理解Agent循环**：结合 `pi-agent-core` 的 `Agent` 类源码
3. **TUI渲染机制**：结合 `pi-tui` 的组件系统理解交互模式
4. **添加新工具**：参考现有工具实现，创建自定义工具

---

*本指南基于源码结构分析，具体源码细节需结合实际阅读更新。*
