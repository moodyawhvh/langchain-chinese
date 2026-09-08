---
type: "Concept"
title: "Streaming: Token-by-Token Output"
description: "How streaming works across LLM components and chains, token-by-token delivery via AIMessageChunk, callback integration, and memory/latency tradeoffs."
tags: [streaming, token-streaming, llm-output, chat-models, callbacks, astream, real-time-feedback]
verified:
  - by: openwiki/0.5.0
    at: 2026-09-03T15:18:34.589Z
sources:
  - id: openwiki-source-c9313cf42f0120d86b20245f
    resource: repo://libs/core/langchain_core/callbacks/base.py
  - id: openwiki-source-c7a2c3ef4ec61c3e28011205
    resource: repo://libs/core/langchain_core/callbacks/streaming_stdout.py
  - id: openwiki-source-5f8bc32563177d89fbab9b2f
    resource: repo://libs/core/langchain_core/language_models/chat_model_stream.py
  - id: openwiki-source-c52037e7b642f7ac5a7642a8
    resource: repo://libs/core/langchain_core/language_models/chat_models.py
  - id: openwiki-source-77dc1fb726463969f9d53658
    resource: repo://libs/core/langchain_core/messages/ai.py
  - id: openwiki-source-a1981e868973f6fd7f71e12e
    resource: repo://libs/core/langchain_core/runnables/base.py
generated: { by: "openwiki/0.5.0", at: "2026-09-03T15:18:34.589Z" }
---

> 🌐 本文档由 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 翻译,英文原版见原项目。

## 总览

**流式(Streaming)**是 LangChain 逐 token 增量交付模型输出的机制,而不是等整段响应。它让 Web UI、控制台等用户界面获得实时反馈,是构建不因模型延迟而阻塞的响应式应用的基础。

应用不调用阻塞的 `invoke()` 等完整响应,而是调用 `stream()` 或 `astream()`,随模型产出逐段接收部分输出。每块是携带增量内容的 `AIMessageChunk`。回调经 `on_llm_new_token` 事件拦截这些块,使你能在不先收集完整响应的情况下观察、记录或响应每个 token。

流式贯穿链路 —— 提示词、模型、输出解析器等 runnable —— 在每一级保留增量交付。通过组合,只要链上所有组件支持流式,链就自动支持。本页讲清跨组件的流式机制、与非流式 invoke 的权衡,以及如何把流式集成进应用。

## 同步流式:stream()

**位置**:`repo://libs/core/langchain_core/language_models/chat_models.py#L727-L856`

`BaseChatModel.stream()` 是主要的同步流式入口。它随底层模型产出逐个 yield `AIMessageChunk`,内容为增量 —— 单个 token、JSON 片段或结构化块更新。

### 控制流

1. **检查流式是否实现**:`_should_stream()` 判断模型是否支持流式。不支持则 `stream()` 回退到 `invoke()`,只 yield 一个完整结果。

2. **初始化回调**:由传入的 `RunnableConfig` 配置 `CallbackManager`,绑定回调、标签和元数据。

3. **触发 on_chat_model_start**:回调生命周期从 `on_chat_model_start` 开始,标志 LLM 调用启动。

4. **迭代模型块**:对底层 `_stream()` 实现产出的每个 `ChatGenerationChunk`:
   - 块的消息 ID 缺失时设为唯一 run ID。
   - 计算并附加响应元数据(模型供应商、延迟等)。
   - **触发 on_llm_new_token**,传入块内容和完整块对象,回调可观察或缓冲每个 token。
   - 块消息转为 `AIMessageChunk` 并立即 yield。
   - 块被累积以便后续聚合。

5. **yield 最终 "last" 块**:模型结束后,若 output_version 为 v1(内容块格式),会 yield 一个 `chunk_position="last"` 的空块,通知解析器和消费者流已结束、tool_call_chunks 应定稿。

6. **回调生命周期收尾**:成功则 `on_llm_end` 携带合并了全部块的 `ChatGeneration` 触发;异常则 `on_llm_error` 携带部分累积触发。

### 回退行为

模型未实现流式时(经 `_should_stream(async_api=False)` 判断),`stream()` 委托给 `invoke()`,把单个结果转为 `AIMessageChunk` yield。这保证所有模型都有一致的流式接口,哪怕只有非流式 invoke。

### 限流

模型挂了限流器时,`stream()` 在开始前获取许可,阻塞到限流放行。

## 异步流式:astream()

**位置**:`repo://libs/core/langchain_core/language_models/chat_models.py#L858-L991`

`BaseChatModel.astream()` 是 `stream()` 的异步变体,镜像同步逻辑但使用 async/await 和 `AsyncCallbackManager`。

**关键差异**:
- 回调事件用 `await`(`await run_manager.on_llm_new_token(...)`、`await run_manager.on_llm_end(...)`)
- 经 `async for chunk in self._astream(...)` 迭代
- 经 `await self.rate_limiter.aacquire(blocking=True)` 获取限流许可

异步流式协议与同步一致:立即 yield 块、每 token 触发回调、收到 "last" 信号时定稿工具调用块。

## AIMessageChunk:增量内容

**位置**:`repo://libs/core/langchain_core/messages/ai.py#L418-L536`

`AIMessageChunk` 是流式期间 yield 的消息类型。与 `AIMessage` 不同,它表示对话消息的**部分增量更新**,并支持用 `+` 运算符合并。

### 结构

- **content**:字符串或内容块列表。流式期间每块只含该步骤的新 token 或增量。
- **tool_call_chunks**:`ToolCallChunk` 对象列表(流式中的不完整工具调用)。参数陆续到达时逐步更新。
- **chunk_position**:可选哨兵;为 `"last"` 表示流中最后一个块,触发工具调用与推理块的定稿。
- **response_metadata**:模型专属元数据(延迟、model_provider、用量计数等),由流式处理器附加。

### 合并与聚合

流式块经 `+` 运算符累积:合并内容、拼接 tool_call 参数、合并元数据。块合并完毕或收到 "last" 信号时,重建出 `tool_calls` 已定稿(不再是 chunks)的完整 `AIMessage`。

## 回调集成:on_llm_new_token

**位置**:`repo://libs/core/langchain_core/callbacks/base.py#L65-L88`

`on_llm_new_token` 回调在流式期间每个 token 或块触发,支持实时观察与日志。

### 签名

```python
def on_llm_new_token(
    self,
    token: str | list[str | dict[str, Any]],
    *,
    chunk: GenerationChunk | ChatGenerationChunk | None = None,
    run_id: UUID,
    parent_run_id: UUID | None = None,
    tags: list[str] | None = None,
    **kwargs: Any,
) -> Any:
```

- **token**:字符串 token 或内容块列表(output_version="v1" 时)。
- **chunk**:完整 `ChatGenerationChunk`,携带元数据、消息 ID、响应元数据和 tool_call_chunks。
- **run_id**:本次流式运行的唯一标识,用于追踪与关联。

### 示例:流式输出到 stdout

```python
from langchain_core.callbacks import StreamingStdOutCallbackHandler

callback = StreamingStdOutCallbackHandler()

# Callbacks are passed via RunnableConfig
for chunk in model.stream(
    messages,
    config=RunnableConfig(callbacks=[callback])
):
    pass  # callback prints each token to stdout
```

`StreamingStdOutCallbackHandler` 实现 `on_llm_new_token` 把 token 写入 `sys.stdout`,让流式输出实时可见。

## 贯穿链路的流式

流式流经由 runnable(提示词、模型、解析器)组成的链。流式协议在每一级经 `Runnable` 的 `stream()` 和 `transform()` 方法实现。

### 默认行为

**位置**:`repo://libs/core/langchain_core/runnables/base.py#L1194-L1235`

默认情况下,`Runnable.stream()` yield `invoke()` 的一个完整输出。支持流式的子类重写 `stream()` 或 `transform()` 来 yield 块。

### 经 RunnableSequence 流式

**位置**:`repo://libs/core/langchain_core/runnables/base.py#L3075-L3320`

`RunnableSequence`(`|` 运算符创建的链)在以下条件时自动支持流式:
1. **所有上游组件实现 transform**:`transform()` 方法把流式输入映射为流式输出。
2. **最后组件产出块**:输出解析器和模型实现 `transform()` 来 yield 部分结果。

任一组件未实现 `transform()` 时,流式只能在该组件完成后开始(阻塞点)。多个阻塞组件形成多个缓冲点;但只要最后组件支持流式,最终输出仍从那里流式。

### 流式示例:模型 → 解析器

```python
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

model = ChatOpenAI()
parser = StrOutputParser()
chain = model | parser

# stream yields parser outputs incrementally as tokens arrive
for chunk in chain.stream("What is 2+2?"):
    print(chunk, end="", flush=True)
```

`model.stream()` yield 块时,解析器的 `transform()`(或默认 `stream()`)消费每个块并 yield 其转换结果。文本解析器可直接 yield token;JSON 解析器在可解析时 yield 部分 JSON 对象。

## 经 stream_events 流式:ChatModelStream

**位置**:`repo://libs/core/langchain_core/language_models/chat_model_stream.py`

对需要细粒度事件的高级场景,`BaseChatModel.stream_events(version="v3")` 返回 `ChatModelStream` 对象,暴露**类型化投影属性**(`.text`、`.tool_calls`、`.usage`、`.reasoning`、`.output`),随事件到达逐步累积。

这与简单 token 流式不同,适合需要对推理、工具调用等协议事件做结构化逐事件观察的应用。`ChatModelStream` 还对每个协议事件(不只是 token)触发 `on_stream_event` 回调。

## 内存与延迟权衡:stream() vs invoke()

### invoke()

- **延迟**:等整个模型响应完成才返回。
- **内存**:无需中间存储;只持有最终消息。
- **响应性**:阻塞调用线程/协程直到完成。
- **场景**:批处理;需要一次性拿到完整响应。

### stream()

- **延迟**:首个 token 一到就产出;对用户即时响应。
- **内存**:若调用方收集块,需要缓冲累积。
- **响应性**:非阻塞;支持渐进展示。
- **场景**:Web UI、控制台应用、重视实时反馈的用户交互。

实践中,流式相比 invoke 并不增加显著延迟;模型产 token 的速率相同,区别在于 **token 何时交付给调用方**。交互式应用更适合流式交付 —— 用户看到输出实时出现,而不是盯着空白屏等完整响应。

## 集成模式

### 实时控制台输出

```python
from langchain_core.callbacks import StreamingStdOutCallbackHandler

callback = StreamingStdOutCallbackHandler()
for _ in model.stream(
    messages,
    config=RunnableConfig(callbacks=[callback])
):
    pass  # Tokens are printed as they arrive
```

### 累积流式输出

```python
result = ""
for chunk in model.stream(messages):
    result += chunk.content or ""
print(result)  # Final complete response
```

### 应用逻辑的自定义回调

```python
from langchain_core.callbacks import BaseCallbackHandler

class MyCallback(BaseCallbackHandler):
    def on_llm_new_token(self, token, **kwargs):
        # React to each token (e.g., update UI, log, rate-limit)
        self.buffer.append(token)

for _ in model.stream(
    messages,
    config=RunnableConfig(callbacks=[MyCallback()])
):
    pass
```

### Web 框架中的异步流式

```python
async def chat_endpoint(messages):
    async for chunk in model.astream(messages):
        # Yield to HTTP client as server-sent event
        yield f"data: {chunk.content}\n\n"
```

## 生命周期与错误处理

### 成功的流

1. 触发 `on_chat_model_start`
2. 每块触发 `on_llm_new_token`
3. `on_llm_end` 携带合并后的 `ChatGeneration` 触发

### 出错的流

1. 触发 `on_chat_model_start`
2. 出错前的每块触发 `on_llm_new_token`
3. `_stream()` 或回调中发生错误
4. `on_llm_error` 携带已聚合的部分块触发
5. 异常重新抛给调用方

### 清理

流退出(break、异常或正常结束)时,缓冲的块被合并,回调完成运行。异步流式还会对存在的异步生成器调用 `aclose()` 收尾。

## 扩展点

### 自定义流式实现

`BaseChatModel` 子类重写 `_stream()` 和/或 `_astream()` 实现模型专属流式:

```python
class MyModel(BaseChatModel):
    def _stream(
        self,
        messages: list[BaseMessage],
        stop: list[str] | None = None,
        **kwargs: Any,
    ) -> Iterator[ChatGenerationChunk]:
        # Yield ChatGenerationChunk for each token
        for token in model_api.stream(messages, stop=stop, **kwargs):
            yield ChatGenerationChunk(message=AIMessageChunk(content=token))
```

`stream()` 方法负责回调、合并和生命周期;子类只需实现核心流式循环。

### 自定义输出解析器 transform

输出解析器可重写 `transform()` 来流式产出部分结果:

```python
class MyParser(BaseGenerationOutputParser[T]):
    def transform(
        self,
        input: Iterator[str | BaseMessage],
        config: RunnableConfig | None = None,
        **kwargs: Any,
    ) -> Iterator[T]:
        buffer = ""
        for chunk in input:
            buffer += chunk.content or ""
            # Attempt partial parsing
            if partial := self.parse_result([Generation(text=buffer)], partial=True):
                yield partial
```

解析器因此能在 token 到达时逐步产出更完整的结果。

## 配置与运维

### 禁用流式

模型遵循 `stream=False` 参数或 `_should_stream()` 中的假值判断。直接调 `invoke()` 可绕过流式,即使模型支持。

### 配置回调

```python
config = RunnableConfig(
    callbacks=[StreamingStdOutCallbackHandler()],
    tags=["user-interaction"],
    metadata={"session_id": "..."},
)
for chunk in model.stream(messages, config=config):
    pass
```

回调、标签和元数据贯穿回调生命周期传播。

### 异步流式

异步上下文用 `astream()` 并 await 异步回调:

```python
async for chunk in model.astream(messages):
    # Process chunks asynchronously
    await handle_chunk(chunk)
```

## 结语

流式是构建响应式 LangChain 应用的核心。逐 token 产出并对每个 token 触发回调,流式在不牺牲性能的前提下带来实时用户反馈。协议在模型、链、解析器之间保持一致,让流式操作易于组合,并可在应用栈任意层级观察输出。
