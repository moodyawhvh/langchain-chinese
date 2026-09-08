---
type: "Architecture"
title: "Message Types and Content Representation"
description: "Document the message abstraction, standardized content blocks for multimodal LLM I/O, message hierarchy, and provider-specific block translators."
tags: [messages, content-blocks, chat-models, streaming, multimodal, provider-adapters]
verified:
  - by: openwiki/0.5.0
    at: 2026-09-03T15:18:34.589Z
sources:
  - id: openwiki-source-77dc1fb726463969f9d53658
    resource: repo://libs/core/langchain_core/messages/ai.py
  - id: openwiki-source-b32b84365d17276620c41ebc
    resource: repo://libs/core/langchain_core/messages/base.py
  - id: openwiki-source-2e77747f30fe980d17d5d1a2
    resource: repo://libs/core/langchain_core/messages/block_translators/__init__.py
  - id: openwiki-source-b0f0ae0889f60e428f2f1b96
    resource: repo://libs/core/langchain_core/messages/block_translators/anthropic.py
  - id: openwiki-source-ad04883edeb0ba80d9ebcb7e
    resource: repo://libs/core/langchain_core/messages/block_translators/langchain_v0.py
  - id: openwiki-source-ac2e0f8b0fb1cb3b223672b7
    resource: repo://libs/core/langchain_core/messages/block_translators/openai.py
  - id: openwiki-source-fc874ddb29b9c5840565397f
    resource: repo://libs/core/langchain_core/messages/content.py
  - id: openwiki-source-8bb392f5dbc1fe7faaf52430
    resource: repo://libs/core/langchain_core/messages/human.py
  - id: openwiki-source-dad8cfeb38a829e03e165986
    resource: repo://libs/core/langchain_core/messages/system.py
  - id: openwiki-source-9861ba5cf0c42c142cf732f9
    resource: repo://libs/core/langchain_core/messages/tool.py
  - id: openwiki-source-498a9586e021b126ab8a8b42
    resource: repo://libs/core/langchain_core/messages/utils.py
generated: { by: "openwiki/0.5.0", at: "2026-09-03T15:18:34.589Z" }
---

> 🌐 本文档由 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 翻译,英文原版见原项目。
>
> ⚠️ 原文超过 10000 字符,本页翻译核心章节;代码块保持原样,完整细节见英文原版。

## 总览

LangChain 的消息抽象为面向大语言模型的对话输入输出提供了统一的、与供应商无关的接口。核心是 **`BaseMessage`**,一个可序列化的内容容器,既可容纳纯文本字符串,也可容纳结构化的**内容块**列表 —— 表示文本、图片、音频、视频、工具调用、推理等的 TypedDict 对象。

关键创新是**内容块**:LangChain 把所有内容规范化为统一格式,而不是各家用各家的 schema(OpenAI 的 `image_url` 对比 Anthropic 的 `document` source 块)。应用因此可以可移植地处理多模态消息;适配器(块转换器)只在调用时才转换为供应商专属格式。

## BaseMessage 层级与核心字段

**位置**:`repo://libs/core/langchain_core/messages/base.py#L93-L180`

`BaseMessage` 是所有消息类型的抽象基类。关键字段:

- **`content`**:`str | list[str | dict[Any, Any]]`  
  持有纯文本,或字符串(视为文本块)与字典(内容块字典)的混合列表。

- **`type`**:`str`(schema 必需字段)  
  唯一标识消息种类(`"human"`、`"ai"`、`"system"`、`"tool"`、`"chat"`、`"function"` 及各 chunk 变体)。

- **`additional_kwargs`**:`dict[Any, Any]`  
  保留给尚未映射到标准字段的供应商专属数据(如 Ollama 或 DeepSeek 的 `reasoning_content`)。

- **`response_metadata`**:`dict[Any, Any]`  
  响应相关元数据:headers、token 数、模型名、供应商名、输出版本。

- **`name`**(可选):消息的人类可读标识;多数模型不使用。

- **`id`**(可选):唯一标识,通常由模型供应商分配。

### 核心消息类型

**`HumanMessage`**(`repo://libs/core/langchain_core/messages/human.py#L9-L61`)  
表示用户输入。用于提示、提问和用户的对话轮次。流式支持有 chunk 变体 `HumanMessageChunk`。

**`AIMessage`**(`repo://libs/core/langchain_core/messages/ai.py#L160-L305`)  
表示模型输出。含专用字段:
- **`tool_calls`**:`ToolCall` 字典列表(结构化工具调用请求)。
- **`invalid_tool_calls`**:解析失败的 `ToolCall` 字典(JSON 参数畸形等)。
- **`usage_metadata`**:标准化的 `UsageMetadata` 字典(`input_tokens`、`output_tokens`、`total_tokens`,以及可选的分类细目)。

流式时使用 `AIMessageChunk` 变体,持有 `tool_call_chunks` 而非完整 `tool_calls`。

**`SystemMessage`**(`repo://libs/core/langchain_core/messages/system.py#L9-L61`)  
设定模型行为;通常是对话第一条消息。同样有 `SystemMessageChunk`。

**`ToolMessage`**(`repo://libs/core/langchain_core/messages/tool.py#L26-L164`)  
表示工具调用结果。必需字段:
- **`tool_call_id`**:把本结果与请求它的 `AIMessage.tool_calls[].id` 关联。
- **`content`**:工具输出(字符串或内容块列表)。
- **`status`**:`"success"` 或 `"error"`。
- **`artifact`**(可选):不发送给模型的完整工具输出(如 `content` 只放摘要时的原始数据)。

**`ChatMessage` 与 `FunctionMessage`**  
遗留/专用消息类型。`ChatMessage` 是带 `role` 字段的通用消息;`FunctionMessage` 表示已弃用的函数调用格式。

### 消息块与流式

**位置**:`repo://libs/core/langchain_core/messages/base.py#L450-L500`(BaseMessageChunk)

流式期间,模型增量发出 `AIMessageChunk`。这些块设计为**可合并**:用 `+` 组合时,它们累积内容、按 index 合并工具调用块并聚合 token 用量。

**`AIMessageChunk`** 字段:
- **`tool_call_chunks`**:部分工具调用对象,`name` 与 `args`(JSON 字符串片段)可为空。
- **`chunk_position`**:最终块为 `"last"`,触发聚合逻辑(如把完成的工具调用块解析为完整 `tool_calls`)。

用 `+` 合并块会调用 `add_ai_message_chunks()`,它:
- 拼接字符串内容、合并列表内容块。
- 合并 `tool_call_chunks`,尊重其 `index` 字段。
- 跨块聚合 token 用量。
- 收到最终块(`chunk_position="last"`)时,把累积的工具调用块解析为完整 `ToolCall` 对象。

## 内容块:统一的多模态表示

**位置**:`repo://libs/core/langchain_core/messages/content.py#L1-L878`

内容块是表示不同消息内容类型的 **TypedDict 对象**,提供与供应商无关的抽象,由块转换器转为供应商格式。

### 标准块类型

**TextContentBlock**  
```python
{
    "type": "text",
    "text": str,
    "id": str (optional, auto-generated),
    "annotations": list[Annotation] (optional, citations/metadata),
    "index": int | str (optional, for streaming),
    "extras": dict (optional, provider-specific fields),
}
```
模型的纯文本输出。annotations 支持指向源文档的引用。

**ReasoningContentBlock**  
```python
{
    "type": "reasoning",
    "reasoning": str (optional),
    "id": str (optional),
    "index": int | str (optional),
    "extras": dict (optional),
}
```
来自 o1、o3 等模型的思维链或中间推理。常从 `<think>` 标签或 `additional_kwargs` 中的供应商字段提取。

**ToolCall**  
```python
{
    "type": "tool_call",
    "id": str | None,
    "name": str,
    "args": dict,
    "index": int | str (optional),
    "extras": dict (optional),
}
```
模型调用工具的请求。ID 在单条消息内必须唯一,以便与 `ToolMessage` 响应配对。

**ToolCallChunk**(流式变体)  
```python
{
    "type": "tool_call_chunk",
    "id": str | None,
    "name": str | None,
    "args": str | None,
    "index": int | str (optional),
    "extras": dict (optional),
}
```
部分工具调用(流式时发出)。字符串 `args` 累积 JSON。相同 `index` 的块在到达时合并。

**InvalidToolCall**  
```python
{
    "type": "invalid_tool_call",
    "id": str | None,
    "name": str | None,
    "args": str | None,
    "error": str | None,
    "index": int | str (optional),
    "extras": dict (optional),
}
```
解析失败的工具调用。error 字段记录异常消息。

### 多模态数据块

**ImageContentBlock**  
```python
{
    "type": "image",
    "url": str (optional),
    "base64": str (optional),
    "file_id": str (optional),
    "mime_type": str (optional, required for base64),
    "id": str (optional),
    "index": int | str (optional),
    "extras": dict (optional),
}
```
通过 URL、base64 编码或云端文件引用(如 OpenAI Files API)的图片数据。

**AudioContentBlock、VideoContentBlock**  
与 `ImageContentBlock` 结构类似,`type` 分别为 `"audio"` 和 `"video"`。

**FileContentBlock**  
```python
{
    "type": "file",
    "url": str (optional),
    "base64": str (optional),
    "file_id": str (optional),
    "mime_type": str (optional),
    "id": str (optional),
    "index": int | str (optional),
    "extras": dict (optional),
}
```
图片/音频/纯文本类型未覆盖的通用文件数据(PDF、Word 文档等)。

**PlainTextContentBlock**  
```python
{
    "type": "text-plain",
    "text": str (optional),
    "base64": str (optional),
    "url": str (optional),
    "file_id": str (optional),
    "mime_type": Literal["text/plain"],
    "title": str (optional),
    "context": str (optional),
    "id": str (optional),
    "index": int | str (optional),
    "extras": dict (optional),
}
```
纯文本文档,可选 title 和 context 帮助模型理解。

### 服务端工具调用

**ServerToolCall、ServerToolCallChunk、ServerToolResult**  
支持在服务端执行的工具(如代码执行、网页搜索)。模型发出这些块请求执行,无需本地处理代码。

### NonStandardContentBlock

```python
{
    "type": "non_standard",
    "value": dict,
    "id": str (optional),
    "index": int | str (optional),
}
```
容纳无法映射到标准块类型的供应商专属内容。求值 `content_blocks` 属性时,块转换器会尝试解析非标准块。

### 访问内容块

**位置**:`repo://libs/core/langchain_core/messages/base.py#L199-L260`

`content_blocks` 属性把消息内容规范化为类型化内容块字典列表:

```python
@property
def content_blocks(self) -> list[types.ContentBlock]:
```

**行为**:
1. `content` 为字符串时,包装为 `{"type": "text", "text": content}`。
2. 解析列表项:字符串变文本块;有已知 `type` 的字典原样保留;其余变 `{"type": "non_standard", "value": ...}`。
3. 对 `AIMessage`,检查 `response_metadata["model_provider"]`,若已注册则使用该供应商的转换器(如 OpenAI、Anthropic)。
4. 无转换器时回退到尽力解析。
5. 对 `AIMessage`,把 content 中没有的 `tool_calls` 追加为工具调用块。
6. 若存在,从 `additional_kwargs["reasoning_content"]` 提取推理内容。

## 块转换器:适配供应商格式

**位置**:`repo://libs/core/langchain_core/messages/block_translators/__init__.py` 及各供应商模块

块转换器在 LangChain 标准块与供应商专属格式之间转换。每个供应商模块注册转换函数;访问 `AIMessage.content_blocks` 且 `response_metadata["model_provider"]` 匹配时调用。

### 注册系统

**`register_translator`** 与 **`get_translator`**:
```python
def register_translator(
    provider: str,
    translate_content: Callable[[AIMessage], list[ContentBlock]],
    translate_content_chunk: Callable[[AIMessageChunk], list[ContentBlock]],
) -> None
```

转换器存于 `PROVIDER_TRANSLATORS`,模块加载时经 `_register_translators()` 自动初始化。

### 关键转换器

**OpenAI**(`repo://libs/core/langchain_core/messages/block_translators/openai.py`)  
处理 Chat Completions 格式:
- 把 OpenAI 的 `image_url` 块转为标准 `ImageContentBlock`。
- 把 `tool_calls`(来自函数调用)解析为 `ToolCall` 块。
- 支持 Responses API 的 `input_audio`、`input_file`、`input_image` 类型。
- `convert_to_openai_image_block()` 与 `convert_to_openai_data_block()` 是模型与集成使用的公共工具。

**Anthropic**(`repo://libs/core/langchain_core/messages/block_translators/anthropic.py`)  
处理 Anthropic 格式:
- 把 `document` 块(`source` 字段指定类型:`base64`、`url`、`file` 或 `text`)转为标准文件/纯文本块。
- 把各种 source 类型的 `image` 块转为 `ImageContentBlock`。
- 把 `cache_control` 等供应商字段填入 `extras`。

**Google GenAI 与 Bedrock Converse**  
对 Google 与 AWS 格式做类似转换。

**LangChain v0(向后兼容)**(`repo://libs/core/langchain_core/messages/block_translators/langchain_v0.py`)  
把基于 `source_type` 的遗留块(如 `{"type": "image", "source_type": "url", "url": "..."}`)解析为 v1 块。

### `content_blocks` 中的转换流程

访问 `AIMessage.content_blocks` 时:
1. 检查 `response_metadata["output_version"]` 是否为 `"v1"`(已规范化,短路返回)。
2. 若 `response_metadata["model_provider"]` 已设置,尝试供应商专属转换。
3. 回退到 `BaseMessage.content_blocks` 尽力解析。
4. 对 `AIMessage`,追加 content 中缺失的工具调用,并从 kwargs 提取推理内容。

## 消息操作工具

**位置**:`repo://libs/core/langchain_core/messages/utils.py#L1-L150` 及之后

utils 模块提供消息处理辅助:

**`get_buffer_string`**  
把消息序列转为单个字符串用于日志/调试。支持 `format="prefix"`(角色前缀)或 `format="xml"`(带正确转义的 XML 标签)。多模态内容块安全截断,base64 数据省略。

**`convert_to_messages` 与 `convert_to_openai_messages`**  
把各种输入格式(字典、字符串、`MessageLikeRepresentation` 联合)强转为类型化消息对象。

**`filter_messages`、`trim_messages`、`merge_message_runs`**  
过滤、截断、合并同类型的连续消息。

**`AnyMessage` 联合类型**  
```python
AnyMessage = Annotated[
    Annotated[AIMessage, Tag(tag="ai")]
    | Annotated[HumanMessage, Tag(tag="human")]
    | ... (all message and chunk types)
    Field(discriminator=Discriminator(_get_type)),
]
```
用于 Pydantic 反序列化的 tagged union。`type` 字段在反序列化时判定正确的消息类。

## 内容表示:字符串 vs 块列表

消息接受两种内容形式:

**字符串内容**:
```python
AIMessage(content="Hello, world!")
```
简单、向后兼容。内部视为单个文本块。

**块列表内容**:
```python
AIMessage(
    content=[
        {"type": "text", "text": "What is this?"},
        {"type": "image", "url": "https://example.com/img.png"},
    ]
)
```
或用类型化的 `content_blocks` kwarg:
```python
AIMessage(
    content_blocks=[
        create_text_block("What is this?"),
        create_image_block(url="https://example.com/img.png"),
    ]
)
```

`text` 属性提取所有文本块:
```python
msg = AIMessage(content=[
    {"type": "text", "text": "Hello"},
    {"type": "image", "url": "..."},
])
print(msg.text)  # "Hello"
```

## 与聊天模型集成

聊天模型用消息抽象规范化输入输出:

- **输入**:用户提供消息(字符串、字典或 `MessageLikeRepresentation`)。模型调用 `_normalize_messages()` 转为 `BaseMessage` 对象,并按需为目标供应商展开多模态内容。

- **输出**:模型返回 `AIMessage`:
  - `content`:模型文本响应(多模态时为块列表)。
  - `response_metadata`:填充 `model_provider`、`output_version`、token 数等。
  - `tool_calls`:从供应商格式解析为结构化 `ToolCall` 字典。
  - `usage_metadata`:标准化的 token 计数。

模型调用与流式生命周期详见 `/openwiki/chat-models.md`。

## 供应商专属扩展

内容块的 `extras` 字段在不破坏标准结构的前提下携带供应商元数据:

```python
{
    "type": "text",
    "text": "Response text",
    "extras": {
        "thought_signature": "EpoWCpc...",  # Google
        "cache_control": {"type": "ephemeral"},  # Anthropic
    },
}
```

这种做法在支持新兴供应商能力的同时保持类型安全。

## 向后兼容与版本

LangChain v1.0 引入 v1 内容块格式,取代 v0 的 `source_type` 风格。块转换器两者都处理:

- **v0 块**(如 `{"type": "image", "source_type": "url", "url": "..."}`)被识别并包装为非标准块,再经 `_convert_v0_multimodal_input_to_v1()` 解析。
- **供应商专属块**(如原始 API 响应中 OpenAI 的 `image_url`)由供应商转换器拆包。
- **输出版本跟踪**:`response_metadata["output_version"] = "v1"` 表示内容已规范化,可短路优化。

## 示例工作流

### 发送多模态消息

```python
from langchain_core.messages import HumanMessage, create_text_block, create_image_block

message = HumanMessage(
    content_blocks=[
        create_text_block("Describe this chart."),
        create_image_block(url="https://example.com/chart.png", mime_type="image/png"),
    ]
)

# Access text
print(message.text)  # "Describe this chart."

# Get normalized blocks
for block in message.content_blocks:
    print(block["type"])  # "text", "image"
```

### 处理模型的工具调用

```python
ai_msg = model.invoke([...])
# ai_msg.tool_calls = [
#     {"type": "tool_call", "id": "call_1", "name": "search", "args": {"query": "..."}},
# ]

for tool_call in ai_msg.tool_calls:
    result = invoke_tool(tool_call["name"], tool_call["args"])
    tool_response = ToolMessage(
        content=str(result),
        tool_call_id=tool_call["id"],
    )
```

### 流式与块聚合

```python
chunks = []
for chunk in model.stream(input_msg):
    chunks.append(chunk)
    print(f"Received: {chunk.content}")

# Aggregate all chunks
final = chunks[0]
for chunk in chunks[1:]:
    final = final + chunk

# final.tool_calls are now complete (parsed from tool_call_chunks)
```

### 使用块转换器

模型设置了 `response_metadata["model_provider"]` 时,块转换器透明调用:

```python
# OpenAI model
ai_msg = openai_model.invoke(msg)
# response_metadata contains model_provider="openai"

blocks = ai_msg.content_blocks
# If content is from OpenAI's API, translator converts image_url → ImageContentBlock
```

自定义供应商集成可注册自己的转换器:

```python
from langchain_core.messages.block_translators import register_translator

def my_translate_content(msg: AIMessage) -> list[ContentBlock]:
    # Custom logic
    pass

def my_translate_content_chunk(chunk: AIMessageChunk) -> list[ContentBlock]:
    # Custom logic
    pass

register_translator("my_provider", my_translate_content, my_translate_content_chunk)
```
