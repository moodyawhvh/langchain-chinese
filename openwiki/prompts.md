---
type: "Concept"
title: "Prompt Templates and Few-Shot Learning"
description: "Prompt templates define message sequences and variable substitution patterns for chat models. Few-shot learning selects examples dynamically to teach models by example."
tags: [prompt, template, few-shot, example-selection, variable-substitution, structured-output]
verified:
  - by: openwiki/0.5.0
    at: 2026-09-03T15:18:34.589Z
sources:
  - id: openwiki-source-1f4e0a5b877db4f050f2a34c
    resource: repo://libs/core/langchain_core/example_selectors/base.py
  - id: openwiki-source-d533a177a8d9a5dd46f561d9
    resource: repo://libs/core/langchain_core/example_selectors/length_based.py
  - id: openwiki-source-5e027af8cc764d2750129cf1
    resource: repo://libs/core/langchain_core/example_selectors/semantic_similarity.py
  - id: openwiki-source-03d7415879ed05a392edd62d
    resource: repo://libs/core/langchain_core/prompts/base.py
  - id: openwiki-source-15fdd645c1ee76ae559799c1
    resource: repo://libs/core/langchain_core/prompts/chat.py
  - id: openwiki-source-bc32774051e0e8a931a6fecd
    resource: repo://libs/core/langchain_core/prompts/few_shot.py
  - id: openwiki-source-5549894302ea4dfd5b8f4278
    resource: repo://libs/core/langchain_core/prompts/prompt.py
  - id: openwiki-source-cf81d0ba0a387a7cd9b5dfb8
    resource: repo://libs/core/langchain_core/prompts/string.py
  - id: openwiki-source-204b5e61a019044332bd2dd4
    resource: repo://libs/core/langchain_core/prompts/structured.py
generated: { by: "openwiki/0.5.0", at: "2026-09-03T15:18:34.589Z" }
---

> 🌐 本文档由 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 翻译,英文原版见原项目。
>
> ⚠️ 原文超过 10000 字符,本页翻译核心章节;代码块保持原样,完整细节见英文原版。

## 总览

LangChain 的**提示词模板系统**提供了灵活、可组合的方式来为语言模型构造消息。提示词接受输入变量,把它们格式化为消息序列,并可选地解析结构化输出。系统区分**字符串模板**(原始文本)与**聊天模板**(类型化消息序列)。**Few-shot 提示词模板**增加了动态挑选并注入示例的能力,以示范方式教模型。

## 基础概念

### 提示词类型

LangChain 提供两大类提示词:

#### PromptTemplate(StringPromptTemplate)

`PromptTemplate` 包装带变量占位符的单个字符串模板,格式化引擎三选一:

- **f-string**(默认):Python f-string 语法。快,`{...}` 内支持任意表达式,`{{`/`}}` 转义。
- **mustache**:Mustache 语法,`{{variable}}`。对用户可控模板更安全。
- **jinja2**:完整 Jinja2 模板。支持条件、循环和过滤器,但若模板来自不可信来源有安全风险;LangChain 默认用 `SandboxedEnvironment` 做纵深防御。

**关键属性:**
- `template`:模板字符串。
- `input_variables`:格式化时必须提供的变量名列表。
- `partial_variables`:预填变量;减少格式化所需输入。
- `template_format`:使用哪个引擎(`f-string`、`mustache` 或 `jinja2`)。

```python
from langchain_core.prompts import PromptTemplate

# Simple f-string prompt
prompt = PromptTemplate.from_template("Tell me about {topic}")
output = prompt.format(topic="machine learning")

# Jinja2 with conditionals
prompt = PromptTemplate(
    template="{% if detailed %}Detailed:{% endif %} {content}",
    template_format="jinja2",
    input_variables=["content"],
    partial_variables={"detailed": True}
)

# Partial variables reduce required inputs
prompt = PromptTemplate(
    template="User: {name}, Topic: {topic}",
    input_variables=["topic"],
    partial_variables={"name": "Alice"}
)
result = prompt.format(topic="AI")  # name is already set
```

#### ChatPromptTemplate

`ChatPromptTemplate` 把消息提示词模板串成对话结构。每条消息有角色(system、human、ai、tool 等)和内容。这与 GPT-4、Claude 等聊天模型基于消息的 API 对齐。

**构造模式:**

```python
from langchain_core.prompts import ChatPromptTemplate

# Tuple shorthand
template = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("human", "Hello, {name}"),
    ("ai", "Hi {name}! How can I help?"),
    ("human", "{user_input}"),
])

# Or direct list construction
template = ChatPromptTemplate([
    ("system", "You are a helpful assistant."),
    ("human", "Hello, {name}"),
])
```

简写语法支持的消息类型:
- `"system"` → `SystemMessagePromptTemplate`
- `"human"` → `HumanMessagePromptTemplate`
- `"ai"` → `AIMessagePromptTemplate`
- `"user"` → `"human"` 别名
- `"assistant"` → `"ai"` 别名
- `"tool"` / `"function"` → `ToolMessagePromptTemplate` / `FunctionMessagePromptTemplate`
- `"placeholder"` → `MessagesPlaceholder`,用于动态消息列表

**关键方法:**
- `format_messages(**kwargs)`:返回 `BaseMessage` 对象列表。
- `invoke(dict)`:Runnable 接口,返回含格式化消息的 `ChatPromptValue`。
- `format(**kwargs)`:把消息列表转成单个字符串(调试或非聊天 API 有用)。

#### MessagesPlaceholder

`MessagesPlaceholder` 在提示词的特定位置注入一份预先格式化的消息列表,是维护对话历史的关键。

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

template = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder("chat_history", optional=True),  # optional=True allows empty list
    ("human", "{question}"),
])

# Pass conversation history
result = template.invoke({
    "chat_history": [
        ("human", "What's 2+2?"),
        ("ai", "4"),
    ],
    "question": "And 3+3?",
})
# Messages: [system, human, ai, human]
```

`optional=True` 允许省略该占位符;未提供时以空列表代替。`n_messages` 参数限制纳入多少条最近消息(对 token 预算有用)。

### 模板变量替换

提示词自动从模板语法检测变量名,并在格式化时要求提供。

```python
from langchain_core.prompts import PromptTemplate

prompt = PromptTemplate.from_template("Q: {question}\nA: {answer}")
print(prompt.input_variables)  # ['answer', 'question']

# Variables are validated at runtime
try:
    prompt.format(question="What?")  # Missing 'answer'
except KeyError as e:
    print(f"Error: {e}")
```

**偏应用(partial)**预填部分变量,缩小必填集合:

```python
prompt = PromptTemplate.from_template("User: {name}, Question: {question}")
partial_prompt = prompt.partial(name="Bob")  # Bind 'name'
output = partial_prompt.format(question="How are you?")
# Only 'question' is required now
```

当提示词**只有一个**输入变量时,模板可直接接受非字典参数:

```python
template = ChatPromptTemplate.from_messages([
    ("system", "You are a bot."),
    ("human", "{input}"),
])
result = template.invoke("Hello!")  # Auto-wraps as {"input": "Hello!"}
```

## Few-Shot 提示词模板

Few-shot 学习在用户真实提问之前提供输入-输出示例来教模型。LangChain 提供两种模式:字符串提示词版和聊天版。

### FewShotPromptTemplate

`FewShotPromptTemplate` 把示例格式化进单个字符串提示词。

**结构:**
```
[前缀]

[格式化示例 1]

[格式化示例 2]

...

[后缀]
```

**组成:**
- `prefix`:示例前文本(可选)。
- `example_prompt`:规定每个示例如何格式化的 `PromptTemplate`。
- `examples` 或 `example_selector`:示例来源(固定列表或动态选择器)。
- `suffix`:示例后文本。通常包含实际任务和新输入的占位符。
- `example_separator`:连接前缀、示例和后缀的字符串(默认 `"\n\n"`)。

```python
from langchain_core.prompts import PromptTemplate, FewShotPromptTemplate

examples = [
    {"input": "happy", "output": "sad"},
    {"input": "tall", "output": "short"},
]

example_prompt = PromptTemplate(
    template="Input: {input}\nOutput: {output}",
    input_variables=["input", "output"],
)

prompt = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    suffix="Input: {input}\nOutput:",
    input_variables=["input"],
)

output = prompt.format(input="big")
# Output:
# Input: happy
# Output: sad
#
# Input: tall
# Output: short
#
# Input: big
# Output:
```

### FewShotChatMessagePromptTemplate

`FewShotChatMessagePromptTemplate` 把示例作为消息对嵌入聊天序列。

```python
from langchain_core.prompts import (
    ChatPromptTemplate,
    FewShotChatMessagePromptTemplate,
)

examples = [
    {"input": "2+2", "output": "4"},
    {"input": "2+3", "output": "5"},
]

example_prompt = ChatPromptTemplate.from_messages([
    ("human", "What is {input}?"),
    ("ai", "{output}"),
])

few_shot = FewShotChatMessagePromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
)

template = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful math tutor."),
    few_shot,
    ("human", "What is {input}?"),
])

result = template.invoke({"input": "4+4"})
# Messages: [system, human(2+2?), ai(4), human(2+3?), ai(5), human(4+4?)]
```

## 示例选择器

**示例选择器**不用固定列表,而是按输入动态挑选相关示例,优化提示词长度与相关性。

### BaseExampleSelector 接口

所有选择器实现:

```python
class BaseExampleSelector:
    def add_example(self, example: dict[str, str]) -> Any:
        """Add a new example to the store."""
        
    def select_examples(self, input_variables: dict[str, str]) -> list[dict[str, Any]]:
        """Select which examples to use based on inputs."""
```

### SemanticSimilarityExampleSelector

把示例和输入嵌入向量空间,检索最相似的 `k` 个示例。需要 `VectorStore` 和 embeddings 模型。

```python
from langchain_core.example_selectors import SemanticSimilarityExampleSelector
from langchain_core.embeddings import OpenAIEmbeddings
from langchain_core.vectorstores import Chroma
from langchain_core.prompts import PromptTemplate, FewShotPromptTemplate

examples = [
    {"input": "happy", "output": "sad"},
    {"input": "tall", "output": "short"},
    {"input": "energetic", "output": "lethargic"},
    {"input": "sunny", "output": "gloomy"},
]

# Create vector store from example texts
to_vectorize = [" ".join(example.values()) for example in examples]
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_texts(to_vectorize, embeddings, metadatas=examples)

selector = SemanticSimilarityExampleSelector(
    vectorstore=vectorstore,
    k=2,  # Always return 2 examples
)

example_prompt = PromptTemplate(
    template="Input: {input}\nOutput: {output}",
    input_variables=["input", "output"],
)

prompt = FewShotPromptTemplate(
    example_selector=selector,
    example_prompt=example_prompt,
    suffix="Input: {input}\nOutput:",
    input_variables=["input"],
)

# When formatting, the selector retrieves the 2 most similar examples to "bright"
output = prompt.format(input="bright")
```

**关键参数:**
- `vectorstore`:存放已嵌入示例的 VectorStore。
- `k`:返回的示例数(默认 4)。
- `input_keys`:可选过滤,只用特定键做相似度搜索(如只用 "input" 字段,不用 "output")。
- `example_keys`:可选过滤,只保留返回示例的某些键。
- `vectorstore_kwargs`:传给向量库 `similarity_search` 的额外参数。

### LengthBasedExampleSelector

贪心地选择示例直到达到最大 token/词数,防止提示词超长。token 预算紧张时有用。

```python
from langchain_core.example_selectors import LengthBasedExampleSelector
from langchain_core.prompts import PromptTemplate, FewShotPromptTemplate

examples = [
    {"input": "happy", "output": "sad"},
    {"input": "tall", "output": "short"},
    {"input": "energetic", "output": "lethargic"},
]

example_prompt = PromptTemplate(
    template="Input: {input}\nOutput: {output}",
    input_variables=["input", "output"],
)

selector = LengthBasedExampleSelector(
    examples=examples,
    example_prompt=example_prompt,
    max_length=50,  # Limit prompt to ~50 words
    get_text_length=lambda x: len(x.split()),  # Custom length function
)

prompt = FewShotPromptTemplate(
    example_selector=selector,
    example_prompt=example_prompt,
    suffix="Input: {input}\nOutput:",
    input_variables=["input"],
)

# Selector returns only as many examples as fit within max_length
output = prompt.format(input="fast")
```

**关键参数:**
- `examples`:全部可用示例。
- `max_length`:最大提示词长度(token 或词,由 `get_text_length` 决定)。
- `get_text_length`:测量提示词长度的函数;默认按正则词数。

**行为:** 按顺序迭代示例;下一个示例会超过 `max_length` 时停止添加。贪心而非最优,但快速且可预测。

## 结构化输出提示词

`StructuredPrompt`(beta)把 `ChatPromptTemplate` 与 Pydantic schema 结合,让模型产出符合指定 schema 的 JSON。

```python
from pydantic import BaseModel
from langchain_core.prompts import StructuredPrompt

class QuestionAnswer(BaseModel):
    question: str
    answer: str

template = StructuredPrompt.from_messages_and_schema(
    messages=[
        ("system", "You are a helpful assistant."),
        ("human", "{input}"),
    ],
    schema=QuestionAnswer,
)

# When invoked with a model supporting structured output,
# the model is instructed to return JSON matching QuestionAnswer
result = template.invoke({"input": "What is LangChain?"})
```

适合要求输出一致、可解析的任务(如事实抽取、数据分类)。

## Runnable 接口与链接

所有提示词继承 `RunnableSerializable`,与 LangChain 链构建系统兼容。

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

template = ChatPromptTemplate.from_messages([
    ("system", "You are a poet."),
    ("human", "Write a poem about {topic}"),
])

model = ChatOpenAI(model="gpt-4o")
parser = StrOutputParser()

# Chain: prompt → model → parser
chain = template | model | parser

result = chain.invoke({"topic": "the internet"})
print(result)
```

**关键方法:**
- `invoke(dict) -> PromptValue`:同步格式化。
- `ainvoke(dict) -> PromptValue`:异步格式化。
- `stream(dict)`:流式模式(提示词很少用,下游更常见)。
- `batch(list[dict])`:批量格式化多个输入。

## 提示词组合

提示词用 `+` 运算符组合,合并消息与变量。

```python
from langchain_core.prompts import ChatPromptTemplate

system = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant. Your name is {bot_name}."),
])

conversation = ChatPromptTemplate.from_messages([
    ("human", "{user_input}"),
])

combined = system + conversation
# Equivalent to:
# ChatPromptTemplate([
#     ("system", "You are a helpful assistant. Your name is {bot_name}."),
#     ("human", "{user_input}"),
# ])

result = combined.invoke({"bot_name": "Alice", "user_input": "Hello!"})
```

**规则:**
- 组合 `ChatPromptTemplate` 时,消息拼接。
- 两个模板的输入变量合并。
- partial 变量合并;键冲突报错。
- 模板格式必须兼容(都用 f-string、都用 mustache 等)。

## 从文件加载提示词

**注意:** 经旧的 `save()` / `load_prompt_from_config()` API 做提示词序列化与加载已弃用,推荐使用 `langchain_core.load` 的 `dumpd()` / `loads()`。

```python
from langchain_core.load import loads
import json

with open("prompt.json") as f:
    prompt_dict = json.load(f)

prompt = loads(prompt_dict)
# Returns a deserialized PromptTemplate or ChatPromptTemplate
```

提示词可用 `langchain_core.load` 的 `dumpd()` 序列化为 JSON,便于版本管理与共享:

```python
from langchain_core.load import dumpd
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are helpful."),
    ("human", "{input}"),
])

prompt_dict = dumpd(prompt)
# Contains nested structure compatible with loads()
```

## 与 Agent Factory 集成

提示词是 **Agent Factory**(`create_agent`)的核心输入,为智能体推理和工具使用提供对话上下文。

```python
from langchain.agents import create_agent
from langchain_core.prompts import ChatPromptTemplate

system_template = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful weather assistant."),
])

# Or use a simple string
agent = create_agent(
    model="openai:gpt-4o",
    tools=[weather_tool],
    system_prompt="You are a helpful weather assistant."
)
```

智能体工厂内部把提示词与模型、工具绑定一起编译,经状态机管理消息流。中间件可通过 `wrap_model_call` 钩子在模型调用前拦截并修改提示词,支持提示词优化或安全过滤等用例。

## 安全注意事项

### 模板注入

从用户输入构造提示词时,用**partial 变量**或**输入变量**,不要字符串拼接:

```python
# UNSAFE: Vulnerable to prompt injection
user_input = input("Enter text: ")
template = f"User said: {user_input}"  # Don't do this

# SAFE: Use variable substitution
from langchain_core.prompts import PromptTemplate
prompt = PromptTemplate.from_template("User said: {user_input}")
output = prompt.format(user_input=user_input)
```

### Jinja2 沙箱

使用 Jinja2 模板时,LangChain 默认应用 `SandboxedEnvironment`,阻止访问双下划线属性(`__class__`、`__globals__` 等)。但:

- **不要接受不可信来源的 Jinja2 模板。** 沙箱是尽力而为,并非万无一失。
- 普通方法调用与属性访问仍然允许(如 `obj.method()`)。
- 必须用 Jinja2 时,对不可信输入优先 `f-string` 或 `mustache`。

```python
# Safe: f-string template from user, validated at construction time
from langchain_core.prompts import PromptTemplate
prompt = PromptTemplate(
    template="Hello {name}",  # User-provided, but simple variable syntax
    input_variables=["name"],
)
```

## 生命周期与状态

提示词模板在函数意义上**不可变**:调用 `format()` 或 `invoke()` 不修改模板。`partial()` 等方法返回新实例。

```python
original = PromptTemplate.from_template("Say {text}")
partial = original.partial(text="hello")  # Returns a NEW PromptTemplate

# original is unchanged
print(original.input_variables)  # ['text']
print(partial.input_variables)   # []
```

不可变性使管线中的安全组合与缓存成为可能。

## 可观测与追踪

所有提示词支持 LangChain 标准追踪与可观测钩子:

```python
template = ChatPromptTemplate.from_messages([
    ("system", "You are helpful."),
    ("human", "{input}"),
])

# Add metadata for tracing
template_with_metadata = template.with_config({
    "metadata": {"version": "1.0"},
    "tags": ["important"],
})

result = template_with_metadata.invoke({"input": "hello"})
# The invoke is traced with the given metadata
```

元数据与标签传播到 LangSmith 等可观测后端,支持调试与性能分析。
