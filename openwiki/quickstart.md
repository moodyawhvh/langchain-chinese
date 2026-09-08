---
type: "Getting Started"
title: "LangChain Repository Quick Start"
description: "Entry point for engineers: orient to the monorepo structure, run first tests, understand what to edit for common tasks, and route to major development areas."
tags: [quickstart, getting-started, monorepo, setup, development, first-steps, cli-reference]
verified:
  - by: openwiki/0.5.0
    at: 2026-09-03T15:18:34.589Z
sources:
  - id: openwiki-source-4d1645cb6317345817452838
    resource: repo://.pre-commit-config.yaml
  - id: openwiki-source-8e384445ccfbaf00747b3e18
    resource: repo://libs/core/langchain_core/__init__.py
  - id: openwiki-source-96c73d2c05223b0b46abdbe9
    resource: repo://libs/core/langchain_core/callbacks/__init__.py
  - id: openwiki-source-c52037e7b642f7ac5a7642a8
    resource: repo://libs/core/langchain_core/language_models/chat_models.py
  - id: openwiki-source-0f6ea1dd09fb4675ff4112b1
    resource: repo://libs/core/langchain_core/messages/__init__.py
  - id: openwiki-source-e7908d069731cffec228727e
    resource: repo://libs/core/langchain_core/output_parsers/__init__.py
  - id: openwiki-source-c46a1d181ab64e61460c84c6
    resource: repo://libs/core/langchain_core/prompts/__init__.py
  - id: openwiki-source-65071982a626569c8820a34b
    resource: repo://libs/core/langchain_core/runnables/__init__.py
  - id: openwiki-source-7c4bed110359f4f5c7847c8b
    resource: repo://libs/core/langchain_core/tools/__init__.py
  - id: openwiki-source-8f1875229ad4a704c8e20a06
    resource: repo://libs/core/Makefile
  - id: openwiki-source-3486a94e6eb23a78271a5bfb
    resource: repo://libs/core/pyproject.toml
  - id: openwiki-source-47db752fe27393d5d4825827
    resource: repo://libs/langchain_v1/langchain/__init__.py
  - id: openwiki-source-71e882e1ac9757ea8e959a7c
    resource: repo://libs/langchain_v1/langchain/agents/factory.py
  - id: openwiki-source-07e634f5cd5f00c636010306
    resource: repo://libs/langchain_v1/langchain/agents/middleware/__init__.py
  - id: openwiki-source-b09b1477098d69af6abaa5b4
    resource: repo://libs/langchain_v1/langchain/chat_models/__init__.py
  - id: openwiki-source-49fbcc45434b619b68220bf9
    resource: repo://libs/Makefile
  - id: openwiki-source-1e66a9da38565f8901e651f4
    resource: repo://libs/partners/openai/langchain_openai/__init__.py
  - id: openwiki-source-48ce5ee900993294d349b4e8
    resource: repo://libs/standard-tests/langchain_tests/__init__.py
generated: { by: "openwiki/0.5.0", at: "2026-09-03T15:18:34.589Z" }
---

> 🌐 本文档由 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 翻译,英文原版见原项目。
>
> ⚠️ 原文超过 10000 字符,本页翻译核心章节;命令、代码块与链接保持原样,完整细节请参考英文原版。

## 欢迎参与 LangChain 开发

LangChain 是一个智能体(agent)工程平台——一个用于构建 LLM 应用的框架,提供可组合的抽象、多家供应商集成以及编排原语。本页将带你了解 monorepo 结构、必要的环境搭建、常见开发任务,以及如何跳转到更深入的文档。

**初次接触本仓库?** 先看[安装与环境搭建](#安装与环境搭建),再通过[快速导航](#快速导航到主要领域)找到你要处理的内容。

## Monorepo 总览

LangChain 在 `/libs/` 下采用**三层架构**:

```
/libs/
├── core/              # langchain-core: 基础抽象(Runnable、BaseChatModel、tools、prompts、messages)
├── langchain_v1/      # langchain: 智能体编排、工厂、中间件
├── partners/          # 各供应商集成(OpenAI、Anthropic、Ollama 等)
├── standard-tests/    # 组件一致性共享测试套件
├── text-splitters/    # 文本切分工具
├── model-profiles/    # LLM 元数据与能力画像
└── Makefile           # monorepo 级构建目标
```

### 各层何时修改

| 层 | 适用场景 | 关键文件 |
|-------|----------------------|-----------|
| **core** | 新增或修改基础抽象、核心接口(Runnable、BaseChatModel、messages、tools、prompts)或回调。 | `libs/core/langchain_core/` |
| **langchain_v1** | 开发智能体工厂功能、中间件、模型初始化、聊天模型选择或高层编排。 | `libs/langchain_v1/langchain/agents/`、`libs/langchain_v1/langchain/chat_models/` |
| **partners/{name}** | 接入新的 LLM 供应商(OpenAI、Anthropic 等)、模型特性或供应商集成。 | `libs/partners/{provider}/` |

## 安装与环境搭建

### 克隆仓库

```bash
git clone https://github.com/langchain-ai/langchain.git
cd langchain
```

### 用 `uv` 安装依赖

monorepo 使用 `uv` 进行快速、确定的依赖解析。先安装一次:

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# 或通过 Homebrew
brew install uv
```

然后同步你所在包的全部依赖:

```bash
# 在任意 libs/ 子目录下,安装所有依赖组(test、lint、type、dev)
uv sync --all-groups

# 或只装需要的部分
uv sync --group test     # 运行测试用
uv sync --group lint     # ruff/mypy 用
```

### Pre-Commit 钩子

安装 git 钩子,自动保证代码质量:

```bash
pre-commit install

# 手动运行所有钩子
pre-commit run --all-files

# 只运行某个钩子
pre-commit run ruff --all-files
```

Pre-commit 钩子会执行:
- YAML/TOML 语法校验
- 文本规范化与行尾空白清理
- **按包粒度的格式化和 lint**(ruff、mypy)
- 各 `pyproject.toml` 之间的**版本一致性检查**

## 常见开发任务

### 运行单元测试

```bash
# 在任意包目录下(libs/core、libs/langchain_v1 等)
make test

# 运行指定测试文件
make test TEST_FILE=tests/unit_tests/agents/test_factory.py

# 监视模式(文件变更自动重跑)
make test_watch

# 运行扩展测试(带 @pytest.mark.requires 标记)
make extended_tests
```

**要点:**
- 测试默认带**套接字限制**(`--disable-socket`),防止意外联网
- 通过 pytest-xdist **并行执行**(`-n auto`)
- 会清除 LangSmith 追踪变量,保证测试隔离
- 测试路径与源码对应:`langchain_core/runnables/base.py` → `tests/unit_tests/runnables/test_base.py`

### 格式化与 Lint

```bash
# 格式化所有 Python 文件(ruff)
make format

# 检查 lint 问题(ruff、mypy)
make lint

# 仅类型检查(mypy)
make type

# 只格式化变更文件(与 main 做 git diff)
make format_diff
```

**使用的工具:**
- **ruff**:快速的 Python linter 和格式化工具(替代 black、isort、flake8)
- **mypy**:静态类型检查器
- 两者都通过 `uv run --group lint` 运行

### 提 PR 前的完整本地校验

在推送 PR 之前运行:

```bash
# 在你的包目录下
make format && make lint && make test
```

或一行搞定:

```bash
cd libs/core && make format lint test
```

## 快速导航到主要领域

用下面的表路由到详细文档:

| 任务 | 从这里开始 | 关键概念 |
|------|-----------|--------------|
| **构建智能体** | [Agent Factory](/openwiki/agent-factory.md) | create_agent、AgentState、中间件组合、图执行 |
| **新增 LLM 供应商** | [Adding a Chat Model Provider](/openwiki/partner-pattern.md) | ChatModel 实现、消息转换、供应商注册、标准测试 |
| **理解架构** | [Architecture Overview](/openwiki/architecture.md) | 三层设计、依赖流向、core/编排/partners 分工 |
| **使用聊天模型** | [Chat Model Interface](/openwiki/chat-models.md) | BaseChatModel 协议、流式、工具绑定、结构化输出 |
| **动态初始化模型** | [Model Initialization](/openwiki/model-initialization.md) | init_chat_model 工厂、provider:model 语法、回退链 |
| **组合组件(链、流水线)** | [Runnables & Composability](/openwiki/runnables.md)、[Composability](/openwiki/composability.md) | Runnable 协议、\| 运算符、分支、重试、回退 |
| **使用工具** | [Tools](/openwiki/tools.md) | BaseTool、schema 生成、工具调用、结果处理 |
| **流式输出** | [Streaming](/openwiki/streaming.md) | 逐 token 输出、跨组件流式传输 |
| **约束响应格式** | [Structured Output](/openwiki/structured-output.md) | JSON schema、响应校验、类型化输出 |
| **编写中间件** | [Agent Middleware](/openwiki/middleware.md) | 中间件类型、组合、自定义钩子 |
| **追踪智能体执行** | [Agent Execution Flow](/openwiki/agent-execution.md) | 运行时生命周期、循环控制、状态迁移 |
| **增加可观测性** | [Callbacks & Tracing](/openwiki/callbacks.md) | 回调管理器、LangSmith 集成、日志 |
| **编写单元/集成测试** | [Unit Testing](/openwiki/unit-tests.md)、[Integration Testing](/openwiki/integration-tests.md) | 测试结构、fixture、mock、VCR cassette |
| **使用提示词** | [Prompts](/openwiki/prompts.md) | 模板、few-shot、变量、图片处理 |
| **理解消息类型** | [Messages](/openwiki/messages.md) | AIMessage、ToolMessage、内容块、供应商转换 |
| **使用模型上下文协议** | [MCP Integration](/openwiki/mcp-integration.md) | MCP 服务器、工具适配器、elicitation |
| **查阅所有文件路径** | [Source Map](/openwiki/source-map.md) | 概念到路径的速查表、目录结构 |
| **查看 CI/CD 流程** | [CI/CD Workflows](/openwiki/ci-workflows.md) | GitHub Actions、测试、lint、发布流程 |
| **开发命令参考** | [Dev Commands](/openwiki/dev-commands.md) | make 目标详解、uv 语法、环境搭建 |

## 仓库结构速览

### 根目录

```
/
├── .github/              # GitHub Actions 工作流(CI/CD)
├── .pre-commit-config.yaml # pre-commit 钩子定义
├── .vscode/              # VS Code 配置
├── libs/                 # monorepo 主工作区
├── AGENTS.md             # 面向智能体的文档
├── CLAUDE.md             # 贡献指南(提 PR 前必读)
└── README.md             # 项目总览
```

### `/libs/` 内部

**core/** — 基础抽象(langchain-core 包)
```
core/
├── langchain_core/
│   ├── language_models/  # BaseChatModel 与语言模型契约
│   ├── messages/         # 消息类型与内容块
│   ├── runnables/        # Runnable 协议与运算符
│   ├── tools/            # BaseTool 与工具工具集
│   ├── prompts/          # 提示词模板与 few-shot
│   ├── callbacks/        # 回调管理器与处理器
│   └── output_parsers/   # 输出解析与校验
├── tests/unit_tests/     # 单元测试(不联网)
├── tests/integration_tests/ # 集成测试(调用真实 API)
├── Makefile              # 构建目标(test、lint、format)
└── pyproject.toml        # 包依赖与元数据
```

**langchain_v1/** — 智能体编排(langchain 包)
```
langchain_v1/
├── langchain/
│   ├── agents/
│   │   ├── factory.py    # create_agent 函数
│   │   ├── middleware/   # 可插拔中间件钩子
│   │   └── structured_output.py # 响应 schema
│   ├── chat_models/
│   │   └── base.py       # init_chat_model 工厂
│   ├── mcp/              # 模型上下文协议
│   └── ...
├── tests/unit_tests/
├── tests/integration_tests/
├── tests/cassettes/      # 用于 HTTP mock 的 VCR cassette
├── Makefile
└── pyproject.toml
```

**partners/** — 供应商集成
```
partners/
├── openai/               # ChatOpenAI、embeddings
├── anthropic/            # ChatAnthropic (Claude)
├── ollama/               # ChatOllama(本地模型)
├── groq/                 # ChatGroq
├── mistralai/            # ChatMistralAI
├── huggingface/          # HuggingFace 模型/embeddings
├── deepseek/             # ChatDeepSeek
└── ...(20+ 个其他供应商)
```

每个 partner 结构一致:
```
provider/
├── langchain_{provider}/
│   ├── __init__.py       # 导出 ChatModel 类
│   ├── chat_models/
│   │   └── base.py       # ChatModel 实现
│   └── data/             # 模型画像
├── tests/
│   ├── unit_tests/       # 标准测试 + 自定义测试
│   └── integration_tests/
├── pyproject.toml
├── Makefile
└── uv.lock
```

## 你的第一个 PR:完整流程

### 1. 挑任务

用上面的[快速导航](#快速导航到主要领域)表决定要做什么。首次贡献者建议:
- **简单**:加测试、修类型错误、改进文档
- **中等**:新增中间件钩子、扩展工具接口
- **困难**:新增供应商集成(参考 [Adding a Chat Model Provider](/openwiki/partner-pattern.md))

### 2. 阅读贡献指南

写代码前先读:
- **[CLAUDE.md](repo://CLAUDE.md)** — 规范、风格与 PR 要求
- **对应的 wiki 页面** — 你要改的领域的深入背景(见上表)

### 3. 配置你的包

```bash
cd libs/{core|langchain_v1|partners/provider}
uv sync --all-groups
pre-commit install
```

### 4. 修改代码

遵循代码库中已有的风格与模式。写类型标注;代码与测试同步提交。

### 5. 本地检查

```bash
make format lint test
```

全部通过后才能推送。

### 6. 提交并推送

```bash
git add .
git commit -m "简述本次改动"
git push origin your-branch
```

pre-commit 钩子会自动运行。失败就修复后重新提交。

### 7. 发起 Pull Request

将 PR 关联到相关 issue,并在描述中引用你读过的 wiki 页面。LangChain 团队会评审并反馈。

## 需要认识的关键文件

| 文件 | 用途 |
|------|---------|
| `CLAUDE.md` | 贡献指南、风格与规范 |
| `libs/Makefile` | monorepo 级 make 目标(lock、check-lock) |
| `libs/{core,langchain_v1,partners/*/Makefile` | 各包的 test、lint、format 目标 |
| `.pre-commit-config.yaml` | 代码质量 git 钩子 |
| `pyproject.toml`(每包) | 包元数据、依赖、构建配置 |

## 故障排查

### 测试因套接字报错失败
测试默认限制套接字。若确需联网:
- 把测试写到 `tests/integration_tests/`(见 [Integration Testing](/openwiki/integration-tests.md))
- 或本地解除限制:`uv run --group test pytest --disable-socket=false ...`

### 导入错误或版本不匹配
重新生成 lockfile:
```bash
cd libs
make lock
```

或单个包内:
```bash
cd libs/core
uv lock
```

### 类型检查失败
运行 mypy 查看详细错误:
```bash
make type
```

类型签名写法可参考 [Chat Models](/openwiki/chat-models.md) 或 [Runnables](/openwiki/runnables.md)。

### pre-commit 钩子拦截提交
pre-commit 会自动修复格式和部分问题。重新暂存再提交:
```bash
git add .
git commit -m "..."  # 再试一次
```

若 lint 仍失败,运行 `make lint` 看细节并手动修复。

## 常用命令速查

```bash
# 环境搭建
uv sync --all-groups          # 安装全部依赖
pre-commit install            # 安装 git 钩子

# 测试
make test                      # 运行单元测试
make test TEST_FILE=path/     # 运行指定测试文件
make test_watch               # 监视模式(自动重跑)
make integration_tests        # 运行集成测试

# 代码质量
make format                   # 格式化(ruff)
make lint                     # lint 检查(ruff、mypy)
make type                     # 仅类型检查(mypy)

# Lockfile 管理
cd libs && make lock          # 重新生成全部 lockfile
cd libs && make check-lock    # 校验 lockfile 是否最新

# 提 PR 前全量检查
make format && make lint && make test
```

## 下一步

1. **阅读 [CLAUDE.md](repo://CLAUDE.md)** 了解贡献规范
2. **按任务挑一个 wiki 页面**(见[快速导航](#快速导航到主要领域))
3. **克隆、配置环境,完成第一处修改**
4. **运行 `make format lint test`** 本地校验
5. **发起 PR** 并跟进评审

欢迎来到 LangChain!🚀
