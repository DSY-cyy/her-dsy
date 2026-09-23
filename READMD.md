# Hermes Agent 六步闯关学习笔记

> 基于 Hermes 官方文档整理，补充 OpenAI / Anthropic / Gemini / Hugging Face 大模型概念与代码示例。
> 适合作为课程作业笔记提交，重点标注命令与验收标准。

---

## Step 1 · Installation — 安装 Hermes Agent

### 学习目标
- 在本地或服务器安装好 Hermes Agent，理解安装器自动完成了哪些工作
- 让 `hermes` 命令全局可用（加入 PATH）
- 跑通官方自检 `hermes doctor`

### 核心知识点

| 组件 | 用途 | 安装位置 |
| --- | --- | --- |
| uv | Python 包管理器 + 虚拟环境生命周期管理 | `~/.local/share/hermes/` |
| Python 3.11+ | Hermes 本体运行环境（隔离 venv） | 隔离虚拟环境内 |
| Node.js v18+ | 浏览器自动化（Playwright）依赖 | 随安装器一起部署 |
| ripgrep (rg) | 项目内快速文件搜索 | 系统 PATH |
| ffmpeg | TTS / 音频格式转换 | 系统 PATH |
| hermes CLI | 全局命令入口 | `~/.local/bin` 或 `/usr/local/bin` |

**安装器原理**：Hermes 安装器是一个自解压脚本（curl / PowerShell），自动完成所有运行时依赖的安装。用户无需手动安装 Python、Node.js、ripgrep 或 ffmpeg。

**环境变量配置**：
- 安装后将二进制路径加入 shell PATH
- 普通用户默认：`~/.local/bin`
- root 模式：`/usr/local/bin`
- 重载 shell 后即可全局使用 `hermes` 命令

**平台安装命令**：

```bash
# Linux / macOS / WSL2 / Android(Termux)
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# Windows 原生（在 PowerShell 中运行）
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

> Windows 用户推荐先装 WSL2，再在 WSL2 中运行 curl 脚本；也可下载 Hermes Desktop 安装器。

### 操作命令 / 代码示例

```bash
# ── Windows PowerShell 安装 ──
iex (irm https://hermes-agent.nousresearch.com/install.ps1)

# ── 重载 shell（让 PATH 生效） ──
$ refreshenv          # 或关闭重新打开 PowerShell

# ── 版本验证 ──
hermes --version
# 预期输出：hermes-agent x.y.z

# ── 官方自检 ──
hermes doctor
# 检查依赖、模型配置、工具链。仅提示可忽略项即通过。

# ── 彻底重装（谨慎：会删除 ~/.hermes/ 记忆） ──
rm -rf ~/.hermes/ ~/.local/share/hermes/
# 再重新运行安装脚本
```

### 拓展：OpenAI CLI / Anthropic CLI 安装对比

| 工具 | 安装方式 | 依赖 | 环境隔离 |
| --- | --- | --- | --- |
| **Hermes Agent** | curl/PowerShell 一键安装脚本 | uv + Node.js + Python + ripgrep + ffmpeg（全部自动） | 安装器创建隔离 venv，用户无需手动配置 |
| **OpenAI CLI** | `pip install openai`（或官方 CLI 工具） | Python + pip，由用户自行管理虚拟环境 | 用户自行创建 venv，API Key 通过 `OPENAI_API_KEY` 环境变量传递 |
| **Anthropic CLI** | `npm install -g @anthropic-ai/claude-cli` | Node.js + npm | npm 全局安装，OAuth 或 API Key 通过环境变量/配置文件注入 |

**Hermes 的优势**：
- 一条命令完成所有依赖安装，用户零配置
- 统一抽象了多家 provider，无需为每个供应商安装单独 CLI
- 内置 Tool Gateway、会话管理、技能系统，功能一体化

### 验收清单
- [ ] 执行安装脚本且未报致命错误
- [ ] 重载 shell 后 `hermes --version` 正常输出版本号
- [ ] 运行 `hermes doctor` 自检通过（或仅提示可忽略项）
- [ ] 确认 `hermes` 二进制路径已加入当前 shell 的 PATH
- [ ] 查看 `~/.hermes/logs/` 目录，确认无持续报错

### 官方文档链接
- [官方安装文档](https://hermes-agent.nousresearch.com/docs/getting-started/installation)
- [官方学习路径](https://hermes-agent.nousresearch.com/docs/getting-started/learning-path)
- [GitHub 仓库](https://github.com/NousResearch/hermes-agent)

---

## Step 2 · Quickstart — 快速上手与首次对话

### 学习目标
- 启动 Hermes 会话，熟悉 TUI 与 CLI 两种界面
- 验证它能回复、能在需要时调用工具
- 理解多轮对话上下文不丢失的机制

### 核心知识点

**初始化流程**：
- 首次运行 `hermes` 会启动模型配置向导（`hermes model`）
- 提示选择 provider、OAuth 登录或粘贴 API Key
- 配置完成后，欢迎横幅显示当前模型、可用工具、已装 skill

**模型配置向导**：运行 `hermes model` 进入交互式向导
- 选择 provider 类型 → 完成 OAuth 登录 或 粘贴 API Key → 确认模型名称
- 向导将配置写入 `~/.hermes/config.yaml`

**TUI 与 CLI 两种界面**：

| 界面 | 命令 | 特点 |
| --- | --- | --- |
| CLI（经典） | `hermes` | 纯文本交互，适合脚本/SSH 场景 |
| TUI（推荐） | `hermes --tui` | 支持鼠标、覆盖层、快捷键、方向键滚动、Tab 切换面板 |

**大模型推理基本链路（Agent 工作原理）**：
1. **Prompt 构建**：系统提示 + 用户输入 + 工具描述（tool schema） + 历史对话拼成消息列表
2. **模型推理**：LLM 接收消息列表，基于工具描述判断是否需要调用工具。若需要，输出结构化的工具调用请求（tool_name + arguments）
3. **工具执行**：Hermes 解析模型输出，调用对应工具（read_file / run_shell / search_web 等），获取执行结果
4. **结果汇总**：工具返回的结果包装成 tool_result 消息送回模型，模型组织成最终回答

这一来一回可能多轮（模型可能调用多个工具或重试）。这就是 Agent 与普通聊天机器人的本质区别：**Agent 能「动手」，而聊天机器人只能「回答」**。

### 操作命令 / 代码示例

```bash
# ── 交互式模型配置向导 ──
hermes model
# 选择 provider → OAuth 登录 或 输入 API Key → 选定模型

# ── 一键登录（Nous Portal，覆盖 300+ 模型 + Tool Gateway） ──
hermes setup --portal
# 自动登录并开启 Tool Gateway（网页搜索、图像、TTS）

# ── 启动 TUI（推荐界面） ──
hermes --tui
# 欢迎横幅显示：模型、工具、skill、会话信息

# ── 首次工具调用测试 prompt（在 TUI/CLI 中输入） ──
Summarize this repo in 5 bullets and tell me what the main entrypoint is.
# 若模型判断需读文件，会看到 read_file / run_shell / search_web 调用记录

# ── 会话管理 ──
hermes --continue          # 或 hermes -c，恢复最近会话
hermes sessions            # 列出所有会话
hermes --session <id>     # 进入指定会话
hermes --new               # 强制开启新会话
```

### 验收清单
- [ ] `hermes` 或 `hermes --tui` 能启动并出现欢迎横幅
- [ ] 用一个具体 prompt 得到了正常回复
- [ ] 观察到 Hermes 在需要时调用了工具（终端 / 文件 / 搜索）
- [ ] `hermes --continue` 能恢复上次会话
- [ ] 用 `hermes sessions` 列出并理解会话管理机制

### 官方文档链接
- [官方会话文档](https://hermes-agent.nousresearch.com/docs/getting-started/using-hermes)
- [官方学习路径](https://hermes-agent.nousresearch.com/docs/getting-started/learning-path)
- [GitHub 仓库](https://github.com/NousResearch/hermes-agent)

---

## Step 3 · Providers & Model Configuration — 模型配置

### 学习目标
- 搞懂 Provider / Model / Base URL / API Key 四个概念的关系
- 至少配好一个可用的推理来源
- 理解本地模型为什么需要 ≥64K 上下文

### 核心知识点

**四要素关系**：

| 要素 | 定义 | 示例 |
| --- | --- | --- |
| **Provider** | 模型的来源 | Nous Portal、Anthropic、OpenAI、OpenRouter、DeepSeek、自定义端点 |
| **Model** | 具体的模型标识符 | `claude-sonnet-4-20250514`、`gpt-4o`、`deepseek-chat`、`llama3.1:70b` |
| **Base URL** | API 请求的 endpoint | 官方 provider 通常内置；自定义端点需显式指定，如 `http://localhost:11434`（Ollama） |
| **API Key** | 访问凭证（鉴权） | 通过环境变量（`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `DEEPSEEK_API_KEY`）或 `config.yaml` 的 `api_key` 字段注入 |

**配置方式对比**：

| Provider | 配置方式 | 适用场景 |
| --- | --- | --- |
| **Nous Portal** | `hermes model` → OAuth 登录 | 省心一站式，覆盖 300+ 模型，附带 Tool Gateway |
| **Anthropic (Claude)** | OAuth 或 `ANTHROPIC_API_KEY` | 代码能力强，稳定，适合重代码任务 |
| **OpenAI** | `OPENAI_API_KEY` | 兼容 GPT-4o / GPT-4 Turbo，Tool Calling 支持完善 |
| **OpenRouter** | `OPENROUTER_API_KEY` | 模型选择极多，便于横向比较和切换 |
| **DeepSeek** | `DEEPSEEK_API_KEY` | 性价比高，适合预算敏感场景 |
| **自定义端点** | `hermes model` → Custom endpoint | Ollama / vLLM / 私有模型；注意上下文 ≥64K |

**最低上下文要求 64K token**：
- Hermes 要把系统提示、工具描述、历史记录、待办任务同时塞进上下文
- 本地模型（Ollama / llama.cpp）务必把上下文设到至少 64K，否则可能拒绝启动或在长任务中崩溃
- 刚开始可以用云端大模型（Claude / GPT-4o / DeepSeek-V3）把流程跑通，再切本地模型降低成本

### 操作命令 / 代码示例

```bash
# ── 交互式模型配置向导（所有 provider 通用） ──
hermes model

# ── 五种配置方式速览 ──

# 1. Nous Portal（一键登录 + Tool Gateway）
hermes setup --portal

# 2. Anthropic（OAuth / API Key）
# 在向导中选择 Anthropic，完成 OAuth 或输入 ANTHROPIC_API_KEY

# 3. OpenAI（API Key）
# export OPENAI_API_KEY="sk-..."  或写入 config.yaml

# 4. DeepSeek（API Key）
# export DEEPSEEK_API_KEY="sk-..."

# 5. 自定义端点（Ollama 示例）
# config.yaml 中设置 base_url: http://localhost:11434
# 模型名如 llama3.1:70b，上下文务必 ≥64K

── config.yaml 片段 ──
model:
  provider: openai
  model: gpt-4o
  api_key: ${OPENAI_API_KEY}

model:
  provider: ollama
  model: llama3.1:70b
  base_url: http://localhost:11434
```

**拓展：OpenAI 兼容接口标准与 Tool Calling 支持差异**：

OpenAI Compatible API：自 OpenAI 发布 API 标准后，众多厂商（Ollama、vLLM、DeepSeek、Mistral、百度千帅等）都提供兼容 OpenAI 请求格式的 endpoint。Hermes 的自定义端点正是利用了这一标准 —— 通过 `base_url` + `API Key` 即可接入。

**Tool Calling / Function Calling 支持度（2025 年中）**：

| 提供商 | Tool Calling 支持 | 备注 |
| --- | --- | --- |
| **OpenAI (GPT-4o / GPT-4 Turbo)** | 完善 | 最早的 Function Calling 标准，支持多工具并行调用 |
| **Anthropic (Claude 3.5/4)** | 完善 | Tool Use 方式与 OpenAI 类似但消息格式略有不同，长上下文理解优秀 |
| **Gemini (Google)** | 支持 | Function Calling 支持，但工具描述格式和返回约定与 OpenAI 有差异，需注意适配 |
| **DeepSeek / 开源模型** | 大部分支持 | 通过 OpenRouter / 自定义端点，兼容 OpenAI function calling，但工具选择可靠性因模型而异 |
| **本地模型（Ollama / llama.cpp）** | 依赖模型本身 | 部分模型（如 Hermes-nomic、某些微调 Qwen）支持较好，但 64K 上下文是硬性要求 |

### 验收清单
- [ ] 跑过 `hermes model` 交互式向导并选定 Provider
- [ ] 能正常对话（说明模型与鉴权已通）
- [ ] 本地 / Ollama 模型已把上下文设到 ≥64K
- [ ] 对应 API Key 已写入环境变量或 config.yaml
- [ ] 确认所用模型的定价与上下文上限适合自己场景

### 官方文档链接
- [官方模型配置](https://hermes-agent.nousresearch.com/docs/integrations/providers)
- [官方学习路径](https://hermes-agent.nousresearch.com/docs/getting-started/learning-path)
- [GitHub 仓库](https://github.com/NousResearch/hermes-agent)

---

## Step 4 · CLI 常用命令

### 学习目标
- 熟悉 Hermes 的斜杠命令体系，能用 `/help` 查阅所有命令
- 掌握会话管理机制：创建、列出、恢复、切换会话

### 核心知识点

**斜杠命令体系**：以 `/` 开头的命令用于控制会话行为和查询状态，**不经过模型推理**，直接由 Hermes 解析执行。

| 命令 | 功能 |
| --- | --- |
| `/help` | 显示所有可用命令与快捷键 |
| `/model` | 查看或切换当前模型 |
| `/tools` | 列出当前可调用的工具（内置 + MCP） |
| `/status` | 显示会话状态：模型、工具数、上下文长度、内存占用 |
| `/context` | 查看当前上下文内容摘要（系统提示、历史、待办） |
| `/skills` | 列出已安装技能；`/skills create <name>` 手动创建空技能 |
| `/skills browse` | 浏览全部可用技能 |
| `<skill_name>` | 直接调用技能（例如 `/git-commit-helper`） |

**会话管理机制**：
- 每次 `hermes` 启动创建一个新会话（除非使用 `--continue` / `--session`）
- 会话记录保存在 `~/.hermes/sessions/`，包含完整对话历史、工具调用记录
- `hermes sessions` 列出所有存档会话
- `hermes --session <id>` 跳转指定会话
- `hermes --continue`（或 `-c`）恢复最近一次会话，适合断点续谈

### 操作命令 / 代码示例

```bash
── 核心斜杠命令（在 TUI/CLI 中输入） ──

/help          显示所有可用命令与快捷键
/model         查看或切换当前模型
/tools         列出当前可调用的工具（内置 + MCP）
/status        显示会话状态：模型、工具数、上下文长度、内存占用
/context       查看当前上下文内容摘要（系统提示、历史、待办）
/skills        列出已安装技能；/skills create <name> 手动创建空技能
/skills browse 浏览全部可用技能
<skill_name>   直接调用技能（例如 /git-commit-helper）

── 会话管理命令（终端中运行） ──

hermes               # 启动新会话（经典 CLI）
hermes --tui         # 启动新会话（TUI，推荐）
hermes --continue    # 或 hermes -c，恢复最近会话
hermes sessions      # 列出所有存档会话
hermes --session <id>  # 跳转进入指定会话
hermes --new         # 强制开启新会话（忽略最近会话）
```

**拓展：OpenAI / Anthropic 官方 CLI 的命令模式对比**：

| 维度 | Hermes Agent | OpenAI CLI | Anthropic CLI (claude) |
| --- | --- | --- | --- |
| **命令模式** | 斜杠命令 + 自然语言 prompt 混合，工具调用实时可见 | 一次性请求/响应，多轮需自行维护消息历史 | 交互式对话，支持多轮，上下文自动累积 |
| **会话管理** | 内置会话存档（`~/.hermes/sessions/`）、`hermes sessions`、`--continue`、`--session <id>` | 无内置会话存档，开发者自行实现 | 可以保存和恢复对话，但功能相对简陋 |
| **工具调用历史** | 可通过 `/tools`、`/context` 审查 | 无 | 不支持工具调用历史的细粒度审查 |
| **排错支持** | `/status`、`/context`、`hermes doctor`、`~/.hermes/logs/` | 有限 | 较少 |

**Hermes 的优势**：斜杠命令丰富（`/model` `/tools` `/status` `/context` `/skills`），且天然支持工具调用历史、会话存档、cross-session 记忆。对排错和复杂任务非常实用。

### 验收清单
- [ ] 熟悉并能使用 `/help` 查看命令列表
- [ ] 使用 `/model` 查看或切换当前模型
- [ ] 使用 `/tools` 列出可调用工具
- [ ] 使用 `/status` 查看会话状态
- [ ] 掌握 `hermes sessions` / `hermes --continue` / `hermes --session <id>` 会话管理

### 官方文档链接
- [官方 CLI 使用指南](https://hermes-agent.nousresearch.com/docs/zh-Hans/user-guide/cli)
- [官方学习路径](https://hermes-agent.nousresearch.com/docs/getting-started/learning-path)

---

## Step 5 · Configuration — 配置文件

### 学习目标
- 理解 `~/.hermes/` 目录结构及各子目录用途
- 掌握 `config.yaml` 普通配置与 `.env` 密钥配置的区别
- 学会用 `${VAR}` 语法在配置中引用环境变量
- 理解配置与密钥分离的安全最佳实践

### 核心知识点

**`~/.hermes` 目录结构**：

```
~/.hermes/
├── config.yaml           # 普通配置（模型、provider、MCP 服务器、connector）
├── .env                  # 密钥配置（API Key 等敏感信息，gitignore 保护）
├── memories/             # 记忆文件（USER.md, MEMORY.md, sessions/）
├── skills/               # 用户安装的技能目录
├── logs/                 # 运行日志，排错时查看
└── sessions/             # 会话存档
```

**config.yaml 普通配置**：
- 存放非敏感配置：模型选择、provider 设置、MCP 服务器定义、connector 配置、技能路径等
- 使用 YAML 语法，**缩进必须为两个空格**
- 支持 `${VAR_NAME}` 语法引用环境变量

**`.env` 密钥配置的区别**：
- API Key 等敏感信息不应明文写在 `config.yaml` 中
- 推荐使用 `.env` 文件或环境变量，并在 `config.yaml` 中用 `${VAR_NAME}` 语法引用
- 这样密钥与配置分离，便于 `.gitignore` 保护、方便在不同环境间切换

### 操作命令 / 代码示例

```yaml
── ~/.hermes/config.yaml 示例 ──

model:
  provider: openai
  model: gpt-4o
  api_key: ${OPENAI_API_KEY}

mcp_servers:
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]
  fetch:
    command: uvx
    args: ["mcp-server-fetch"]
  sqlite:
    command: uvx
    args: ["mcp-server-sqlite", "--db-path", "/home/user/notes.db"]

connectors:
  telegram:
    bot_token: ${TELEGRAM_BOT_TOKEN}
    allowed_users:
      - user1
      - user2
```

```bash
── ~/.hermes/.env 示例（不提交到 git） ──

OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
DEEPSEEK_API_KEY=sk-...
TELEGRAM_BOT_TOKEN=123456:ABC-DEF...
```

```bash
── 环境变量在 shell 中的设置 ──

# Linux/macOS/WSL
export OPENAI_API_KEY="sk-..."

# PowerShell
$ env:OPENAI_API_KEY = "sk-..."
```

**拓展：配置与密钥分离的安全最佳实践**：

1. **永远不要将 API Key 明文提交到代码仓库**。将 `.env` 添加到 `.gitignore`，`config.yaml` 中只保留 `${VAR}` 引用。
2. **使用环境变量注入**：在 CI/CD、Docker、devcontainer 中通过环境变量传递密钥，而非配置文件。
3. **文件权限保护**：`.env` 文件权限设为 600（仅所有者可读写）。在 Linux/macOS 上 `chmod 600 ~/.hermes/.env`。
4. **OAuth 优先于 API Key**：对于支持 OAuth 的 provider（Nous Portal、Anthropic），优先使用 OAuth 登录，避免长期密钥泄露风险。
5. **最小权限原则**：connector 配置中一定要设置 `allowed_users` 白名单。绝不要把 MCP filesystem 服务器指向 `~/.ssh` 或根目录。
6. **定期轮换密钥**：为每个环境创建独立密钥，定期更换。若怀疑泄露，立即在提供商控制台吊销并生成新密钥。
7. **日志脱敏**：确认 `~/.hermes/logs/` 中的日志不包含密钥明文。如有疑虑，检查日志并配置对应的日志过滤。

### 验收清单
- [ ] 了解 `~/.hermes/` 的目录结构及各子目录用途
- [ ] 能在 `config.yaml` 中正确配置模型 provider、MCP 服务器
- [ ] 掌握 `${VAR}` 语法在 `config.yaml` 中引用环境变量
- [ ] 将 `.env` 添加到 `.gitignore`，理解密钥与配置分离的意义
- [ ] 理解 connector `allowed_users` 白名单的安全必要性

### 官方文档链接
- [官方配置说明](https://hermes-agent.nousresearch.com/docs/zh-Hans/user-guide/configuration)
- [官方学习路径](https://hermes-agent.nousresearch.com/docs/getting-started/learning-path)

---

## Step 6 · Tools — 工具调用

### 学习目标
- 理解 Tool Calling / Function Calling 的基本原理
- 掌握 Hermes 内置工具：read_file、write_file、run_shell、search_web 等
- 完成一次完整工具调用实操：读取文件 + 执行 shell 命令 + 生成结果文件
- 了解 OpenAI Function Calling 与 Anthropic Tool Use 的协议差异

### 核心知识点

**Tool Calling 原理（三段论）**：
1. **工具描述注入**：系统提示中包含工具的名称、参数 schema（JSON Schema）、描述文字
2. **模型判断与请求**：LLM 接收消息列表后，判断是否需要调用外部工具。若需要，输出结构化的工具调用请求（`tool_name` + `arguments` JSON）
3. **执行与结果回送**：Hermes 解析模型输出，调用对应工具，获取执行结果。将结果包装成 `tool_result` 消息送回模型。模型综合所有信息生成最终回答

**Hermes 内置工具**：

| 工具 | 功能 |
| --- | --- |
| `read_file` / `read_text_file` | 读取本地文件内容，支持多种格式 |
| `write_file` / `write_text_file` | 向本地写入文件，可创建新文件或覆盖 |
| `run_shell` / `run_terminal` | 在本地 shell 执行命令，返回 stdout/stderr |
| `search_web` | 通过 Tool Gateway 或自定义搜索引擎搜索网络 |
| `browse` / `browser` | 启动浏览器访问页面、截图、提取 DOM 内容 |
| `edit_file` | 对现有文件进行局部编辑（删改指定行区间） |
| 其他 | TTS（语音合唱）、图像生成、代码执行等，取决于所接入的 Tool Gateway |

**完整实操案例：读取 + 统计 + 生成报告**

```
用户 prompt（在 hermes --tui 中输入）：
"读取当前项目的 README.md，统计所有 Python 文件的代码行数，并将结果写入 report.md。"

模型自主决定调用以下工具（终端中可见调用记录）：

1. read_file(path: "README.md")
   → 返回 README 内容

2. run_shell(command: "find . -name '*.py' -exec wc -l {} + | tail -1")
   → 返回 Python 文件总行数

3. write_file(path: "report.md", content: "## 项目总结\n- README: ...\n- Python 总行数: 1234\n")
   → report.md 创建成功
```

**拓展：OpenAI Function Calling / Anthropic Tool Use 官方标准对比**：

| 维度 | OpenAI Function Calling | Anthropic Tool Use |
| --- | --- | --- |
| **工具描述位置** | `tools` 数组（messages 之外），每个工具含 `type: "function"`、`function.name`、`function.description`、`function.parameters`（JSON Schema） | `tools` 数组，直接包含 `name`、`description`、`input_schema`（JSON Schema） |
| **模型返回** | `finish_reason: "tool_calls"`，`content` 中含 `tool_call` 对象列表（index、id、type、function.name、function.arguments JSON） | `content` 块类型为 `"tool_use"`，含 `id`、`type`、`name`、`input`（参数对象） |
| **结果回送** | 开发者解析 arguments、执行函数、构造 `tool_message` 送回 | 开发者执行后构造 `type: "tool_result"` 的 content 块送回，可选 `name`、`content`（结果文本）或 `is_error` 标志 |
| **多工具支持** | 支持多工具并行调用 | 支持多工具调用 |
| **协议兼容** | OpenAI 兼容 API 标准，被众多厂商采纳 | Anthropic 独有格式，差异较大 |

**其他厂商**（Gemini / DeepSeek / 本地模型）大体兼容 OpenAI 的 function calling 格式，但细节（如工具结果格式、多工具选择行为）可能略有不同。Hermes 的工具抽象层正是为了屏蔽这些差异。

### 操作命令 / 代码示例

```bash
── 实操案例：读取 + 统计 + 生成报告 ──

用户 prompt（在 hermes --tui 中输入）：
读取当前项目的 README.md，统计所有 Python 文件的代码行数，
并将结果写入 report.md。

模型自主决定调用以下工具（终端中可见调用记录）：

1. read_file(path: "README.md")
   → 返回 README 内容

2. run_shell(command: "find . -name '*.py' -exec wc -l {} + | tail -1")
   → 返回 Python 文件总行数

3. write_file(path: "report.md", content: "## 项目总结\n- README: ...\n- Python 总行数: 1234\n")
   → report.md 创建成功

── 查看已安装工具 ──
/tools
# 列出所有可用工具（内置 + MCP 提供的）
```

```bash
── 对比：OpenAI Function Calling 官方标准（curl 示例） ──

curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{
      "role": "user",
      "content": "What is the current weather in San Francisco?"
    }],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get current weather in a city",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {"type": "string"}
          },
          "required": ["city"]
        }
      }
    }]
  }'

# OpenAI 返回工具调用请求，开发者执行后再将结果送回模型。
```

```bash
── 对比：Anthropic Tool Use（curl 示例） ──

curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 1024,
    "tools": [{
      "name": "get_weather",
      "description": "Get current weather in a city",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": {"type": "string"}
        },
        "required": ["city"]
      }
    }],
    "messages": [{
      "role": "user",
      "content": "What is the current weather in San Francisco?"
    }]
  }'

# Anthropic 返回 tool_use 块，开发者执行后以 tool_result 块送回。
```

### 验收清单
- [ ] 理解 Tool Calling / Function Calling 的基本原理（工具描述 → 模型判断 → 调用请求 → 执行 → 结果汇总）
- [ ] 掌握 Hermes 内置工具：`read_file`、`write_file`、`run_shell`、`search_web`、`/tools` 列表查看
- [ ] 完成一次完整工具调用实操：读取文件 + 执行 shell 命令 + 生成结果文件
- [ ] 了解 OpenAI Function Calling 与 Anthropic Tool Use 的协议差异
- [ ] 理解 Hermes 的工具抽象层如何屏蔽不同 provider 的协议差异

### 官方文档链接
- [官方工具调用文档](https://hermes-agent.nousresearch.com/docs/zh-Hans/user-guide/features/tools)
- [官方学习路径](https://hermes-agent.nousresearch.com/docs/getting-started/learning-path)
- [OpenAI Function Calling 官方文档](https://platform.openai.com/docs/guides/function-calling)
- [Anthropic Tool Use 官方文档](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)

---

## 总结

通过以上六步，学习者应当能够：

1. **安装** Hermes Agent 并配置好运行环境
2. **配置**至少一个模型 provider 并成功对话
3. **熟悉** CLI/TUI 界面、斜杠命令和会话管理
4. **管理** `config.yaml` 和 `.env`，实现配置与密钥分离
5. **调用** Hermes 内置工具完成实际任务
6. **理解** Tool Calling 协议在不同提供商间的差异

> 提示：若遇到问题，按 `hermes doctor → hermes model → hermes setup` 的顺序排查。经验法则：先让一次完整对话跑通，再叠加 gateway / cron / skills / 语音等功能。

---

*笔记整理自 Hermes 官方文档（https://hermes-agent.nousresearch.com/docs）及参考样例（https://xiaolouchunyu.top/learning/hermes），补充 OpenAI / Anthropic / Gemini / Hugging Face 大模型概念与代码示例。*
