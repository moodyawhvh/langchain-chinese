# LangChain monorepo 全局开发指南

> 🌐 本文档由 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 翻译,英文原版见原项目。
>
> ⚠️ 原文超过 10000 字符,本页翻译核心章节;CI/LangSmith 等长尾细节做了精简,完整内容见英文原版。代码块与链接保持原样。

本文档提供理解 LangChain Python 项目所需的背景,并协助开发工作。

<corridor>

## Corridor 安全分析

当 Corridor 的 `analyzePlan` 工具可用时,先制定计划并在生成或修改代码前用该工具分析;写代码前先落实其给出的安全建议。

</corridor>

## 项目架构与背景

### Monorepo 结构

这是一个使用 `uv` 的 Python monorepo,包含多个独立版本化的包。

```txt
langchain/
├── libs/
│   ├── core/             # `langchain-core` 原语与基础抽象
│   ├── langchain/        # `langchain-classic`(遗留,不加新功能)
│   ├── langchain_v1/     # 积极维护的 `langchain` 包
│   ├── partners/         # 第三方集成
│   │   ├── openai/       # OpenAI 模型与 embeddings
│   │   ├── anthropic/    # Anthropic (Claude) 集成
│   │   ├── ollama/       # 本地模型支持
│   │   └── ...(LangChain 团队维护的其他集成)
│   ├── text-splitters/   # 文档分块工具
│   ├── standard-tests/   # 集成共享测试套件
│   ├── model-profiles/   # 模型配置画像
├── .github/              # CI/CD 工作流与模板
├── .vscode/              # VSCode 标准配置与推荐扩展
└── README.md             # LangChain 项目信息
```

- **核心层**(`langchain-core`):基础抽象、接口与协议。用户通常不需要直接接触这一层。
- **实现层**(`langchain`):具体实现与高层公共工具。
- **集成层**(`partners/`):第三方服务集成。注意本 monorepo 并未涵盖所有 LangChain 集成;部分集成在独立仓库维护,例如 `langchain-ai/langchain-google` 与 `langchain-ai/langchain-aws`。这些仓库通常与本 monorepo 克隆在同一层级,需要时可直接浏览 `../langchain-google/` 引用其代码。
- **测试层**(`standard-tests/`):面向 partner 集成的标准化集成测试。

### 开发工具与命令

- `uv` – 快速的 Python 包安装与依赖解析器(替代 pip/poetry)
- `make` – 常用开发命令的任务执行器,可用命令见各 `Makefile`
- `ruff` – 快速的 Python linter 与格式化工具
- `mypy` – 静态类型检查
- `pytest` – 测试框架

本 monorepo 用 `uv` 管理依赖,本地开发采用可编辑安装:`[tool.uv.sources]`。

`libs/` 下每个包都有自己的 `pyproject.toml` 和 `uv.lock`。

跑测试前,先执行以下命令配置所有包:

```bash
# 所有依赖组
uv sync --all-groups

# 或只装某一组:
uv sync --group test
```

```bash
# 运行单元测试(不联网)
make test

# 运行指定测试文件
uv run --group test pytest tests/unit_tests/test_specific.py
```

```bash
# Lint 检查
make lint

# 格式化
make format

# 类型检查
uv run --group lint mypy .
```

#### 环境与依赖管理

monorepo 中所有环境与依赖操作一律使用 `uv`,不要直接调用 `pip`、`poetry` 或 `conda`。

- 让 `uv` 管理解释器和虚拟环境 —— `uv sync` 和 `uv run` 无需手动 `source .venv/bin/activate`。不要在包目录外随意建虚拟环境。
- 每个包通过自己的 `pyproject.toml` 声明支持的 Python 范围;不要固定全局 Python 版本。需要指定解释器时,以包的 `requires-python` 为准,不要假设系统 Python。
- 依赖一律通过 `uv sync`(可选 `--group <name>` / `--all-groups`)显式安装,绝不隐式安装。
- 不要在同一会话中混用环境;非必要不加新依赖 —— 确需添加时,说明理由(近期发布/提交、采用情况)。

#### 关键配置文件

- pyproject.toml:主工作区配置与依赖组
- uv.lock:锁定依赖,保证构建可复现
- Makefile:开发任务

#### PR 与提交标题

遵循 Conventional Commits。允许的类型与 scope 见 `.github/workflows/pr_lint.yml`。所有标题必须带 scope,无一例外 —— 主包 `langchain` 也不例外。

- `type(scope):` 之后的正文以小写字母开头,除非首词是专有名词(如 `Azure`、`GitHub`、`OpenAI`)或命名实体(类、函数、方法、参数或变量名)。
- 命名实体用反引号包裹以代码样式渲染;专有名词不加修饰。
- 标题简短达意 —— 细节留给正文。

示例:

```txt
feat(langchain): add new chat completion feature
fix(core): resolve type hinting issue in vector store
chore(anthropic): update infrastructure dependencies
feat(langchain): `ls_agent_type` tag on `create_agent` calls
fix(openai): infer Azure chat profiles from model name
```

#### 分支命名

分支前缀格式为 `<github-username>/<scope>/<short-description>`:

- `<github-username>` — 作者的 GitHub 用户名(如 `mdrxy`)。
- `<scope>` — 与 Conventional Commit 标题一致的 scope(`core`、`langchain`、partner 名、`infra`、`docs` 等)。
- `<short-description>` — kebab-case,简短,结尾不带斜杠。

示例:

```txt
mdrxy/anthropic/normalize-tool-call-ids
mdrxy/core/vector-store-type-hints
mdrxy/infra/agents-md-branch
```

#### PR 描述

描述本身就是摘要 —— 不要再加 `# Summary` 标题。

- 当 PR 关闭某个 issue 时,最顶部单独一行写关闭关键词,然后是水平分隔线和正文:

  ```txt
  Closes #123

  ---

  <其余描述>
  ```

  只有 `Closes`、`Fixes`、`Resolves` 会在合并时自动关闭对应 issue;`Related:` 等标注仅作信息说明,不会关闭任何东西。

- 解释"为什么":谁受益、遇到什么问题、如何解决。比起冗长摘要,更推荐简洁的用户故事。
- 为可能不熟悉该领域的读者写作。避免圈内黑话,用语对公众友好 —— 有助于可读性。
- **不要**引用行号;文件一改行号就失效。
- 尽量不贴完整文件路径或文件名,改为按名称引用受影响的符号、类或子系统。
- 类、函数、方法、参数和变量名用反引号包裹。
- 对于全新功能或改变行为的 bugfix,PR 描述应包含 `## Release note` 小节,用发布说明口吻陈述用户可见的变化。
- 多数情况下无需单独的"Test plan"/"Testing"小节;仅当测试覆盖不明显、有风险或值得说明时才提测试。
- 指出本次改动中需要重点评审的部分。
- 附一句简短声明,说明本贡献有 AI 智能体参与。

## 核心开发原则

### 保持公共接口稳定

关键:导出/公共方法务必保持函数签名、参数位置与名称,不做破坏性变更。
任何函数签名变更都要向开发者示警,无论看起来是否具有破坏性。

**修改任何公共 API 之前:**

- 确认函数/类是否在 `__init__.py` 中导出
- 查看测试和示例中的既有用法
- 新参数使用仅限关键字参数:`*, new_param: str = "default"`
- 实验性特性用 docstring 警告明确标注(使用 MkDocs Material 提示块,如 `!!! warning`)

自问:"这个改动会不会让上周还在用旧版的用户代码挂掉?"

### 代码质量标准

所有 Python 代码必须带类型标注和返回类型。

```python title="示例"
def filter_unknown_users(users: list[str], known_users: set[str]) -> list[str]:
    """Single line description of the function.

    Any additional context about the function can go here.

    Args:
        users: List of user identifiers to filter.
        known_users: Set of known/valid user identifiers.

    Returns:
        List of users that are not in the `known_users` set.
    """
```

- 变量命名要有描述性、见名知义。
- 遵循你所改代码库中的既有模式。
- 在合理的情况下,把复杂函数(>20 行)拆成更小、职责单一的函数。

### 测试要求

每个新功能或 bugfix 必须有单元测试覆盖。

- 单元测试:`tests/unit_tests/`(禁止网络调用)
- 集成测试:`tests/integration_tests/`(允许网络调用)
- 测试框架为 `pytest`;拿不准时参考已有测试。
- 测试文件结构应与源码结构镜像对应。

**清单:**

- [ ] 新逻辑被故意破坏时测试会失败
- [ ] 覆盖正常路径
- [ ] 覆盖边界情况与错误场景
- [ ] 对外部依赖使用 fixture/mock
- [ ] 测试确定性(无 flaky 测试)
- [ ] 测试套件能否捕获新逻辑被破坏的情况?

### 安全与风险评估

- 禁止对用户可控输入使用 `eval()`、`exec()` 或 `pickle`
- 正确的异常处理(不用裸 `except:`),错误信息用 `msg` 变量
- 提交前删除不可达/被注释的代码
- 留意竞态条件与资源泄漏(文件句柄、套接字、线程)
- 确保资源正确清理(文件句柄、连接)

### 文档标准

所有公共函数使用 Google 风格 docstring,带 Args 小节。

```python title="示例"
def send_email(to: str, msg: str, *, priority: str = "normal") -> bool:
    """Send an email to a recipient with specified priority.

    Any additional context about the function can go here.

    Args:
        to: The email address of the recipient.
        msg: The message body to send.
        priority: Email priority level.

    Returns:
        `True` if email was sent successfully, `False` otherwise.

    Raises:
        InvalidEmailError: If the email address format is invalid.
        SMTPConnectionError: If unable to connect to email server.
    """
```

- 类型写在函数签名里,不写进 docstring
  - 有默认值时,除非有后处理或按条件赋值,否则不要在 docstring 里重复默认值。
- 描述聚焦"为什么"而非"是什么"
- 写全参数、返回值与异常
- 描述简洁而清晰
- 使用美式英语拼写(如 "behavior" 而非 "behaviour")
- 不要用 Sphinx 风格的双反引号(`` ``code`` ``);docstring 和注释中的行内代码用单反引号(` `code` `)。

#### 文档与示例中的模型引用

docstring 和示例代码中引用 LLM 时,一律使用最新的正式发布(GA)模型。除非该模型没有 GA 等价物,避免使用预览版或 beta 标识。过时的模型名会显得代码陈旧,也会让用户困惑。

写或改模型引用前,先对照供应商官方文档核实当前模型 ID。不要依赖记忆或缓存的模型名 —— 它们很快过时。

修改代码中**已发布的默认参数值**(如类构造器里 `model=` kwarg 的默认值)可能构成破坏性变更 —— 见上文"保持公共接口稳定"。此条针对文档与示例,不涉及代码默认值。

模型*画像数据*(能力标志、上下文窗口)请使用下述 `langchain-profiles` CLI。

## 模型画像

模型画像用 `libs/model-profiles` 中的 `langchain-profiles` CLI 生成。`--data-dir` 必须指向包含 `profile_augmentations.toml` 的目录,而不是包的顶层目录。

```bash
# 在 libs/model-profiles 下运行
cd libs/model-profiles

# 刷新本仓库内某 partner 的画像
uv run langchain-profiles refresh --provider openai --data-dir ../partners/openai/langchain_openai/data

# 刷新外部仓库中某 partner 的画像(需要 echo y 确认)
echo y | uv run langchain-profiles refresh --provider google --data-dir /path/to/langchain-google/libs/genai/langchain_google_genai/data
```

本仓库内有画像的 partner 示例:

- `libs/partners/openai/langchain_openai/data/`(provider: `openai`)
- `libs/partners/anthropic/langchain_anthropic/data/`(provider: `anthropic`)
- `libs/partners/perplexity/langchain_perplexity/data/`(provider: `perplexity`)

当 `--data-dir` 位于 `libs/model-profiles` 工作目录之外时,必须加 `echo y |` 管道。

## CI/CD 基础设施

### 发布流程

每个 partner 包独立发布。完整流程:

1. **版本号 PR。** 创建一个 PR,各改一行:
   - `langchain_<partner>/_version.py` — `__version__`
   - `pyproject.toml` — `version`
   - `uv.lock` — 在包目录运行 `uv lock`。若 diff 含无关变化(例如不同本地 Python 版本导致的环境标记行),回退它们,只保留本次发布包的 `version = "..."` 行

   标题遵循 Conventional Commits:`release(<partner>): <version>`(如 `release(openrouter): 0.2.6`)。分支名用 `release/<partner>-<version>`。

   补丁号还是次版本号按仓库既有惯例:在 `0.x` 系列内,修复与新增功能都走补丁号(如 `session_id` 字段 → 0.2.1→0.2.2,`parallel_tool_calls` → 0.2.3→0.2.4)。

2. **合并 PR** 到 `master`。

3. **触发发布工作流。** 对 "🚀 Package Release" 工作流(`_release.yml`,文件 ID `63880841`)执行 `gh workflow run`:

   ```bash
   gh workflow run 63880841 --repo langchain-ai/langchain \
     -f working-directory=<partner> -f release-version=<version>
   ```

   `working-directory` 用工作流下拉框里的 partner 短名(如 `openrouter`,不是 `libs/partners/openrouter`)。

4. **其余全部由工作流自动完成** —— **不要**手动创建 GitHub release 或 tag。PyPI 发布成功后,`mark-release` 任务(使用 `ncipollo/release-action`)会创建 GitHub release、tag 和发布说明;发布说明正文由上一个 tag 到 HEAD 之间的提交历史自动生成。

   监控运行:

   ```bash
   gh run view <run-id> --repo langchain-ai/langchain
   ```

   完整任务链:build → release-notes → pre-release-checks → TestPyPI publish → PyPI publish → tag GitHub release。

### PR 标签与 lint

**标题 lint**(`.github/workflows/pr_lint.yml`)

**自动打标:**

- `.github/workflows/pr_labeler.yml` – 统一 PR 打标器(规模、文件、标题、外部/内部、贡献者层级)
- `.github/workflows/pr_labeler_backfill.yml` – 对已开 PR 手动补打标签
- `.github/workflows/auto-label-by-package.yml` – 按 package 给 issue 打标
- `.github/workflows/tag-external-issues.yml` – issue 外部/内部分类

### 集成测试追踪(LangSmith)

定时与手动触发的集成测试(`integration_tests.yml`)会把每次运行追踪到 LangSmith,失败可回链到发起的 Actions run。(`_release.yml` 也会跑集成测试,但目前未配置 LangSmith 追踪。)

CI 设置的环境变量包括 `LANGSMITH_API_KEY`(认证)、`LANGSMITH_TRACING: "true"`(开启追踪)、`LANGSMITH_PROJECT`(默认 `scheduled-testing-py`,可用仓库变量覆盖;改项目请改 GitHub 仓库变量,不要硬编码进工作流)、`LANGSMITH_TAGS`(标识本次运行的逗号分隔标签)和 `LANGSMITH_METADATA`(由 "Build LangSmith Metadata" 步骤构建的 JSON,含 run id、commit SHA 等)。

**追踪桥接插件:** LangSmith SDK 原生并不从环境读取 `LANGSMITH_TAGS` / `LANGSMITH_METADATA`。`libs/standard-tests/langchain_tests/_langsmith_plugin.py` 的 pytest 插件补上了这一环:在整个测试会话期间进入 `langsmith.run_helpers.tracing_context`。它仅在 `GITHUB_ACTIONS=true` 时激活,本地开发不受影响;通过 `pytest11` 入口点在依赖 `langchain-tests` 的包中自动发现。

**单元测试隔离:** 单元测试绝不联网、不发追踪。`libs/core` Makefile 的 `make test` 用 `env -u` 在运行 pytest 前清除追踪变量(`LANGCHAIN_TRACING_V2`、`LANGCHAIN_API_KEY`、`LANGSMITH_API_KEY`、`LANGSMITH_TRACING`、`LANGCHAIN_PROJECT`)。此外,`libs/core/tests/unit_tests/runnables/conftest.py` 有会话级 autouse fixture,显式关闭 runnable 单元测试的追踪,结束后恢复原环境。

### 向 CI 接入新 partner

新增 partner 包时,更新以下文件:

- `.github/ISSUE_TEMPLATE/*.yml` – 加入 package 下拉选项
- `.github/dependabot.yml` – 加入依赖更新条目
- `.github/scripts/pr-labeler-config.json` – 加入文件规则和 scope→label 映射
- `.github/workflows/_release.yml` – 按需加入 API key secret
- `.github/workflows/auto-label-by-package.yml` – 加入包标签
- `.github/workflows/check_diffs.yml` – 加入变更检测
- `.github/workflows/integration_tests.yml` – 加入集成测试配置
- `.github/workflows/pr_lint.yml` – 加入允许的 scope

## GitHub Actions 与工作流

本仓库要求 action 固定到完整长度的 commit SHA;用 tag 会失败。用 `gh` CLI 查询,并确认不是需要解引用的附注 tag 对象。

## 附加资源

- **文档:** https://docs.langchain.com/oss/python/langchain/overview ,源码在 https://github.com/langchain-ai/docs 或 `../docs/`。优先本地安装并用文件搜索工具;必要时按 `.mcp.json` 的定义使用 docs MCP 服务器以编程方式访问。
- **贡献指南:** [Contributing Guide](https://docs.langchain.com/oss/python/contributing/overview)

<!-- OPENWIKI:START -->

## OpenWiki

本仓库有自动生成的 `openwiki/` 证据索引。它是可选的按需背景,不是必读的启动材料。

- 以源码和测试为准。简报中的未知项与评审项是待验证缺口,不是自动生效的要求。
- 优先选择能证明所改行为的最小静默验证;保留完整的失败输出。

定时的 OpenWiki GitHub Actions 工作流会刷新仓库 wiki。除非明确要求,不要手改生成的 OpenWiki 页面;优先更新源码/文档,让 OpenWiki 重新生成。

<!-- OPENWIKI:END -->
