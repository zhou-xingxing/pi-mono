# Pi Monorepo 项目分析

## 项目概览

**Pi Monorepo** 是一个用于构建AI代理和管理LLM部署的TypeScript/Node.js monorepo项目，采用npm workspaces管理。

项目官网: https://pi.dev (由 exe.dev 捐赠域名)
GitHub: https://github.com/badlogic/pi-mono

---

## 包结构

| 包名 | 路径 | 功能描述 |
|------|------|----------|
| **@mariozechner/pi-ai** | `packages/ai/` | 统一多提供商LLM API，支持OpenAI、Anthropic、Google等20+提供商 |
| **@mariozechner/pi-agent-core** | `packages/agent/` | 有状态代理运行时，提供工具执行、事件流、状态管理 |
| **@mariozechner/pi-coding-agent** | `packages/coding-agent/` | 交互式终端编码代理CLI（核心产品） |
| **@mariozechner/pi-mom** | `packages/mom/` | Slack机器人，将消息委托给编码代理处理 |
| **@mariozechner/pi-tui** | `packages/tui/` | 终端UI库，支持差分渲染和无闪烁同步输出 |
| **@mariozechner/pi-web-ui** | `packages/web-ui/` | Web组件库，用于构建AI聊天界面 |
| **@mariozechner/pi-pods** | `packages/pods/` | GPU pod上的vLLM部署管理CLI |

---

## 核心架构层次

```
┌─────────────────────────────────────────────────────────────┐
│  应用层                                                       │
│  ├─ pi-coding-agent (CLI工具)                                │
│  ├─ pi-mom (Slack机器人)                                     │
│  └─ pi-pods (GPU部署管理)                                    │
├─────────────────────────────────────────────────────────────┤
│  UI层                                                        │
│  ├─ pi-tui (终端UI组件)                                      │
│  └─ pi-web-ui (Web组件)                                      │
├─────────────────────────────────────────────────────────────┤
│  代理层                                                      │
│  └─ pi-agent-core (Agent运行时、工具调用、状态管理)            │
├─────────────────────────────────────────────────────────────┤
│  LLM层                                                       │
│  └─ pi-ai (统一LLM API、多提供商支持)                         │
└─────────────────────────────────────────────────────────────┘
```

---

## pi-coding-agent 详解

这是项目的核心产品，一个极简但高度可扩展的终端编码代理。

### 主要功能
- **交互模式**: 完整的TUI界面，支持编辑器、命令、快捷键
- **会话管理**: 基于JSONL的树形结构，支持分支和压缩
- **多模式运行**: 交互式、打印模式、JSON模式、RPC模式
- **多提供商支持**: 20+ LLM提供商，支持OAuth和API Key认证

### 支持的提供商

**订阅制:**
- Anthropic Claude Pro/Max
- OpenAI ChatGPT Plus/Pro (Codex)
- GitHub Copilot
- Google Gemini CLI
- Google Antigravity

**API Key:**
- Anthropic, OpenAI, Azure OpenAI, Google Gemini, Google Vertex
- Amazon Bedrock, Mistral, Groq, Cerebras, xAI
- OpenRouter, Vercel AI Gateway, ZAI, MiniMax
- Hugging Face, Kimi For Coding, OpenCode Zen

### 扩展系统

扩展是TypeScript模块，可自定义工具、命令、快捷键、事件处理器和UI组件。

#### 扩展示例分类

| 类别 | 示例 |
|------|------|
| 生命周期与安全 | permission-gate, protected-paths, sandbox |
| 自定义工具 | todo, question, subagent, ssh |
| 命令与UI | plan-mode, preset, status-line, snake, doom-overlay |
| Git集成 | git-checkpoint, auto-commit-on-exit |
| 自定义Provider | custom-provider-anthropic, gitlab-duo |

### 技能系统 (Skills)

按需加载的能力包，遵循 [Agent Skills标准](https://agentskills.io)。通过 `/skill:name` 调用或让代理自动加载。

### Pi包系统

通过npm或git分享扩展、技能、提示模板和主题：

```bash
pi install npm:@foo/pi-tools
pi install git:github.com/user/repo
pi list
pi update
```

---

## pi-ai 详解

统一LLM API，仅包含支持工具调用（function calling）的模型。

### 支持的API
- `anthropic-messages`: Anthropic Messages API
- `google-generative-ai`: Google Generative AI API
- `openai-completions`: OpenAI Chat Completions API
- `openai-responses`: OpenAI Responses API
- `bedrock-converse-stream`: Amazon Bedrock Converse API
- 以及更多...

### 核心功能
- 自动模型发现
- 提供商配置
- Token和成本追踪
- 简单上下文持久化
- 跨提供商会话切换

---

## pi-agent-core 详解

有状态代理运行时，构建于 pi-ai 之上。

### 核心概念

#### AgentMessage vs LLM Message
- AgentMessage: 灵活类型，可包含自定义应用特定消息类型
- LLM仅理解 `user`, `assistant`, `toolResult`
- `convertToLlm` 函数在每次LLM调用前进行转换

#### 事件流
代理为UI更新发出事件，理解事件序列有助于构建响应式界面：

```
prompt("Hello")
├─ agent_start
├─ turn_start
├─ message_start   { message: userMessage }
├─ message_end     { message: userMessage }
├─ message_start   { message: assistantMessage }
├─ message_update  { message: partial... }
├─ message_end     { message: assistantMessage }
├─ turn_end        { message, toolResults: [] }
└─ agent_end       { messages: [...] }
```

### Steering 和 Follow-up

- **Steering**: 在工具运行时中断代理
- **Follow-up**: 在代理完成后排队工作

---

## pi-tui 详解

最小化终端UI框架，支持差分渲染和同步输出。

### 特性
- **差分渲染**: 三策略渲染系统，仅更新变化内容
- **同步输出**: 使用CSI 2026实现原子屏幕更新（无闪烁）
- **括号粘贴模式**: 正确处理大粘贴（>10行标记）
- **组件化**: 简单Component接口
- **内置组件**: Text, TruncatedText, Input, Editor, Markdown, Loader, SelectList等

### 内置组件

| 组件 | 描述 |
|------|------|
| `Text` | 多行文本，支持自动换行 |
| `TruncatedText` | 单行截断文本，用于状态行 |
| `Input` | 单行文本输入，水平滚动 |
| `Editor` | 多行编辑器，支持自动完成、文件完成 |
| `Markdown` | Markdown渲染，支持语法高亮 |
| `SelectList` | 交互式选择列表 |
| `Image` | 内联图片（Kitty/iTerm2协议） |

---

## pi-web-ui 详解

用于构建AI聊天界面的可重用Web UI组件。

### 特性
- **Chat UI**: 完整界面，包含消息历史、流式传输、工具执行
- **Tools**: JavaScript REPL、文档提取、Artifacts
- **Attachments**: PDF、DOCX、XLSX、PPTX、图片预览和文本提取
- **Storage**: IndexedDB支持的会话、API密钥和设置存储
- **CORS代理**: 浏览器环境自动代理处理

### 组件架构

```
┌─────────────────────────────────────────────────────┐
│                    ChatPanel                         │
│  ┌─────────────────────┐  ┌─────────────────────┐   │
│  │   AgentInterface    │  │   ArtifactsPanel    │   │
│  │  (messages, input)  │  │  (HTML, SVG, MD)    │   │
│  └─────────────────────┘  └─────────────────────┘   │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│              Agent (from pi-agent-core)              │
│  - State management (messages, model, tools)         │
│  - Event emission (agent_start, message_update, ...) │
│  - Tool execution                                    │
└─────────────────────────────────────────────────────┘
```

---

## pi-mom 详解

Slack机器人，由LLM驱动，可执行bash命令、读写文件。

### 特性
- **自管理**: 自动安装工具、配置凭证
- **Slack集成**: 响应@提及和DM
- **Docker沙箱**: 在容器中隔离运行（推荐）
- **持久工作区**: 所有历史、文件、工具存储在可控目录
- **事件系统**: 支持计划提醒和周期性任务

### 数据目录结构

```
./data/
  ├── MEMORY.md           # 全局记忆
  ├── settings.json       # 全局设置
  ├── skills/             # 全局自定义CLI工具
  ├── C123ABC/            # 每个Slack频道一个目录
  │   ├── MEMORY.md       # 频道特定记忆
  │   ├── log.jsonl       # 完整消息历史
  │   ├── context.jsonl   # LLM上下文
  │   ├── attachments/    # 用户共享的文件
  │   └── skills/         # 频道特定工具
```

---

## pi-pods 详解

在GPU pod上部署和管理LLM，自动配置vLLM。

### 支持的Provider
- **DataCrunch**: 最佳共享模型存储（NFS卷）
- **RunPod**: 良好的持久存储
- **Vast.ai**: 卷锁定到特定机器
- **AWS EC2**: 配合EFS使用

### 支持的模型
- **Qwen**: Qwen2.5-Coder, Qwen3-Coder
- **GPT-OSS**: GPT-OSS-20B, GPT-OSS-120B
- **GLM**: GLM-4.5, GLM-4.5-Air
- **自定义模型**: 通过 `--vllm` 参数

---

## 项目理念

1. **极端可扩展性**: 核心保持精简，功能通过扩展实现
2. **无内置功能**: 无MCP、无子代理、无计划模式、无权限弹窗 - 都可通过扩展添加
3. **适应工作流**: 让用户调整工具适应自己的工作方式，而非相反
4. **安全第一**: Docker沙箱、权限门控、路径保护

### 明确的"不做"清单

- **No MCP**: 构建带README的CLI工具（Skills），或构建添加MCP支持的扩展
- **No sub-agents**: 通过tmux生成pi实例，或构建自己的扩展
- **No permission popups**: 在容器中运行，或构建自己的确认流程
- **No plan mode**: 用扩展构建，或安装包
- **No built-in to-dos**: 使用TODO.md文件，或构建自己的扩展
- **No background bash**: 使用tmux，完全可观察，直接交互

---

## 开发信息

- **许可证**: MIT
- **开发状态**: 当前处于OSS假期（2026年2月16日重新开放PR）
- **构建命令**:
  ```bash
  npm install          # 安装所有依赖
  npm run build        # 构建所有包
  npm run check        # 检查、格式化、类型检查
  ./test.sh            # 运行测试
  ./pi-test.sh         # 从源码运行pi
  ```
- **Node版本**: 18+

---

## 文件来源

本分析基于以下README.md文件：
- `/README.md` - 项目根目录说明
- `/packages/ai/README.md` - pi-ai包说明
- `/packages/agent/README.md` - pi-agent-core包说明
- `/packages/coding-agent/README.md` - pi-coding-agent包说明
- `/packages/coding-agent/examples/README.md` - 示例概览
- `/packages/coding-agent/examples/extensions/README.md` - 扩展示例
- `/packages/coding-agent/examples/extensions/doom-overlay/README.md` - DOOM扩展示例
- `/packages/coding-agent/examples/extensions/plan-mode/README.md` - 计划模式扩展示例
- `/packages/coding-agent/examples/extensions/subagent/README.md` - 子代理扩展示例
- `/packages/coding-agent/examples/sdk/README.md` - SDK使用示例
- `/packages/mom/README.md` - pi-mom包说明
- `/packages/pods/README.md` - pi-pods包说明
- `/packages/tui/README.md` - pi-tui包说明
- `/packages/web-ui/README.md` - pi-web-ui包说明
- `/packages/web-ui/example/README.md` - Web UI示例
