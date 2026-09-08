<div align="center">

# langchain 中文文档

[![原项目](https://img.shields.io/badge/原项目-langchain--ai--langchain-blue?style=flat-square&logo=github)](https://github.com/langchain-ai/langchain)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](https://opensource.org/licenses/MIT)
[![微信联系](https://img.shields.io/badge/微信-uaycar-brightgreen?style=flat-square&logo=wechat)](#)

**智能体工程平台(The agent engineering platform)**

</div>

---

> 这是 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 官方 README 的中文翻译文档。
> 完整源代码请访问原项目:https://github.com/langchain-ai/langchain

**代部署 / 定制服务 / 技术咨询 请添加微信:uaycar**

---

## 📖 项目简介

LangChain 是一个用于构建**智能体(Agent)和 LLM 应用的框架**。它帮助你把可互操作的组件与第三方集成串联起来,简化 AI 应用开发——同时随着底层技术演进,你的架构决策依然经得起时间考验。

> 💡 **提示**:刚开始上手?看看 **[Deep Agents](https://docs.langchain.com/oss/python/deepagents/)**——一个构建在 LangChain 之上的更高层封装,为智能体内置了规划、子智能体、文件系统使用等常见能力。

## 🚀 快速开始

安装 langchain:

```bash
uv add langchain
```

第一个示例:

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-5.5")
result = model.invoke("Hello, world!")
```

如果你需要更高级的定制或智能体编排,请查看 [LangGraph](https://github.com/langchain-ai/langgraph),我们用于构建可控智能体工作流的框架。

如需等价的 JS/TS 库,请查看 [LangChain.js](https://github.com/langchain-ai/langchainjs)。

> 💡 **提示**:关于 AI 智能体与 LLM 应用的开发、调试和部署,参见 [LangSmith](https://docs.langchain.com/langsmith/home)。

## 🌐 LangChain 生态

LangChain 框架可以独立使用,同时也能与 LangChain 家族的任何产品无缝集成,为开发者构建 LLM 应用提供全套工具:

- **[Deep Agents](https://docs.langchain.com/oss/python/deepagents/)** —— 构建能够规划任务、使用子智能体、借助文件系统完成复杂任务的智能体
- **[LangGraph](https://docs.langchain.com/oss/python/langgraph/overview)** —— 低层智能体编排框架,构建能可靠处理复杂任务的智能体
- **[Integrations](https://docs.langchain.com/oss/python/integrations/providers/overview)** —— 聊天与嵌入模型、工具与工具集等丰富集成
- **[LangSmith](https://www.langchain.com/langsmith)** —— 面向 LLM 应用的智能体评估、可观测性与调试
- **[LangSmith Deployment](https://docs.langchain.com/langsmith/deployments)** —— 专为长期运行、有状态工作流打造的平台,部署和扩展智能体

## ❓ 为什么选择 LangChain?

LangChain 通过为模型、嵌入、向量库等提供标准接口,帮助开发者构建由 LLM 驱动的应用:

- **实时数据增强** —— 依托 LangChain 庞大的集成库,轻松将 LLM 连接到多样化的数据源和内外部系统,覆盖模型提供方、工具、向量库、检索器等
- **模型互操作性** —— 工程团队在实验中随时更换模型,为应用找到最优选择;行业前沿演进时也能快速适配,LangChain 的抽象层让你持续前进、不失节奏
- **快速原型开发** —— 凭借模块化、组件化的架构,快速构建并迭代 LLM 应用;无需从零重建即可测试不同方案和工作流,加速开发周期
- **生产就绪特性** —— 通过 LangSmith 等集成内建监控、评估和调试支持,部署可靠的应用;使用久经考验的模式与最佳实践,放心扩展规模
- **活跃社区与生态** —— 利用丰富的集成、模板和社区贡献组件;通过活跃的开源社区持续改进,紧跟最新 AI 进展
- **灵活的抽象层次** —— 在适合你的抽象层级上工作:从快速上手的高层链路,到细粒度控制的底层组件;LangChain 随应用复杂度一起成长

## 📚 资源

- [文档](https://docs.langchain.com/oss/python/langchain/overview) —— 概念综述与指南
- [LangChain 生态总览](https://docs.langchain.com/oss/python/concepts/products) —— LangChain、LangGraph 与 Deep Agents 如何协同
- [API 参考](https://reference.langchain.com/python) —— 所有公开类、函数与类型的完整参考
- [ Discussions 论坛](https://forum.langchain.com/c/oss-product-help-lc-and-lg/langchain/14) —— 技术问题、想法与反馈的社区论坛
- [LangChain Academy](https://academy.langchain.com/) —— LangChain 团队出品,关于 LangChain 库与产品的免费系统课程
- [贡献指南](https://docs.langchain.com/oss/python/contributing/overview) —— 如何参与贡献并找到合适的入门 issue
- [行为准则](https://github.com/langchain-ai/langchain/?tab=coc-ov-file) —— 社区准则与规范

---

## 📞 联系方式

**代部署 / 定制服务 / 技术咨询 请添加微信:uaycar**

---

## ⚖️ 版权声明

本文档是 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 官方 README 的中文翻译版本,仅供学习交流使用。原项目所有代码版权归 langchain-ai 团队及相关作者所有,遵循其原始 [MIT 许可证](https://opensource.org/licenses/MIT)。翻译文档中的错误在所难免,如有疑问请以英文原文为准。

**如果觉得有用,请给原项目点个 Star!** ⭐
