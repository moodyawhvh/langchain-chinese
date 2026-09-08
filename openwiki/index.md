---
okf_version: "0.2"
---

> 🌐 本文档由 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 翻译,英文原版见原项目。

# 文件列表

- [智能体执行流程与循环控制](agent-execution.md) - 追踪智能体从用户输入到模型调用、工具分发和循环终止条件的运行时生命周期,详解状态管理与中间件集成点。
- [创建一个基础智能体](agent-factory.md)
- [LangChain 系统架构](architecture.md) - 将 LangChain 框架高层拆解为三层:langchain-core(抽象)、langchain(编排与智能体)、partners(供应商集成),展示依赖关系、组件职责与扩展边界。
- [> Entering new SequentialChain chain...](callbacks.md)
- [聊天模型接口与生命周期](chat-models.md) - 讲解 BaseChatModel 协议、输入/输出处理、流式传输,以及与回调和模型画像的集成点。
- [CI/CD 工作流:GitHub Actions 与发布流程](ci-workflows.md)
- [字典语法创建 RunnableParallel](composability.md)
- [开发命令与本地环境搭建](dev-commands.md) - LangChain monorepo 中 uv、make、lint、test 和类型检查命令速查,包括环境搭建、pre-commit 钩子和测试流程。
- [集成测试:真实 API 测试与 VCR Cassette](integration-tests.md) - 如何编写调用真实模型 API 的集成测试并用 VCR cassette 录制以兼容 CI,包括环境搭建、cassette 管理和参数化模式。
- [Bearer token](mcp-integration.md)
- [消息类型与内容表示](messages.md) - 讲解消息抽象、面向多模态 LLM 输入输出的标准化内容块、消息层级,以及各供应商专用的块转换器。
- [中间件](middleware.md)
- [用 init_chat_model 初始化聊天模型](model-initialization.md) - 通过供应商字符串实例化聊天模型的工厂函数,提供统一配置与运行时模型切换。
- [使用异步方法(ainvoke、astream)](openai-provider.md)
- [新增聊天模型供应商](partner-pattern.md) - 将新 LLM 供应商接入 LangChain monorepo 的分步指南,包括包结构、ChatModel 实现、流式、函数调用与标准测试。
- [提示词模板与 Few-Shot 学习](prompts.md) - 提示词模板为聊天模型定义消息序列与变量替换模式;few-shot 学习通过动态挑选示例,以示范方式教模型做事。
- [LangChain 仓库快速上手](quickstart.md) - 工程师入口:熟悉 monorepo 结构、跑第一次测试、弄清常见任务该改哪里,并路由到各主要开发领域。
- [Runnable:核心组合层](runnables.md) - 解释 Runnable 协议,以及它如何通过 LangChain 表达式语言(LCEL)实现 LLM 组件的可组合链接。
- [Source Map:仓库文件组织](source-map.md) - 按主题定位代码的速查表,把 LangChain 概念映射到 monorepo 中的实现路径,涵盖核心抽象、智能体、中间件、partners 和配置文件。
- [流式:逐 Token 输出](streaming.md) - 流式在 LLM 组件与链中的工作方式、通过 AIMessageChunk 逐 token 交付、回调集成,以及内存/延迟权衡。
- [AutoStrategy(推荐)](structured-output.md)
- [形式一:无参数(名称取自函数)](tools.md)
- [单元测试:策略与模式](unit-tests.md) - 如何用 pytest、fixture、mock 以及 langchain-tests 提供的标准测试类,为 langchain-core 和 langchain 组件编写单元测试。
