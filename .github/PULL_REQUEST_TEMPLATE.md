> 🌐 本文档由 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 翻译,英文原版见原项目。

Fixes #

---

<!-- 把 `Fixes #xx` 关键词保留在最顶部并填上 issue 编号 —— 合并时会自动关闭该 issue。用 1-2 句改动说明替换本注释。不要加 `# Summary` 标题;说明就是摘要。 -->

完整贡献指南:https://docs.langchain.com/oss/python/contributing/overview

> **所有贡献必须使用英文。** 见[语言政策](https://docs.langchain.com/oss/python/contributing/overview#language-policy)。

如果你在这里粘贴一大段明显的 AI 生成描述,你的 PR 可能被忽略或直接关闭!

感谢为 LangChain 做贡献!按以下步骤操作,你的 PR 才会被视为可评审状态。

1. PR 标题:格式为 TYPE(SCOPE): DESCRIPTION

  - 示例:
    - fix(anthropic): resolve flag parsing error
    - feat(core): add multi-tenant support
    - test(openai): update API usage tests
  - 允许的 TYPE 和 SCOPE 取值:https://github.com/langchain-ai/langchain/blob/master/.github/workflows/pr_lint.yml#L15-L33

2. PR 描述:

  - 用 1-2 句话让人容易理解这次改动:谁受益、遇到什么问题、如何解决。比起冗长摘要,更推荐简洁的用户故事。
  - 顶部的 `Fixes #xx` 行对外部贡献是**必需的** —— 填上 issue 编号并保留关键词。它把你的 PR 关联到已批准的 issue,并在合并时自动关闭。
  - 如有破坏性变更,请清楚说明。
  - 如果本 PR 依赖另一个 PR 先合并,请在描述中写上 "Depends on #PR_NUMBER"。

## Release note
<!-- 全新功能或改变行为的 bugfix 必填。用发布说明口吻陈述用户可见的变化。杂务、重构或纯测试改动可省略本节。 -->

3. 在你所改包的根目录运行 `make format`、`make lint` 和 `make test`。

  - 这三项在 CI 中不通过,我们不会评审你的 PR。

4. 你是如何验证代码可用的?

附加要求:

  - 所有外部 PR 必须关联一个解决方案已获维护者批准的 issue 或 discussion,且你必须被分配到该 issue。未经事先批准的 PR 会被关闭。
  - 除非确有必要,PR 不要同时改动多个包。
  - 未经维护者明确许可,不要更新 `uv.lock` 文件,也不要往 `pyproject.toml` 里加依赖(包括可选依赖)。

## 社交账号(可选)
<!-- 如果想在发布公告中获得致谢,在下面填写你的社交账号 -->
Twitter: @
LinkedIn: https://linkedin.com/in/
