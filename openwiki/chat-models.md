---
type: "Architecture"
title: "Chat Model Interface and Lifecycle"
description: "Document BaseChatModel protocol, input/output handling, streaming, and integration points with callbacks and model profiling."
tags: [chat-models, llm-integration, streaming, structured-output, model-capabilities]
verified:
  - by: openwiki/0.5.0
    at: 2026-09-03T15:18:34.589Z
sources:
  - id: openwiki-source-132f3183693cd9cf79d029a5
    resource: repo://libs/core/langchain_core/language_models/base.py
  - id: openwiki-source-5f8bc32563177d89fbab9b2f
    resource: repo://libs/core/langchain_core/language_models/chat_model_stream.py
  - id: openwiki-source-c52037e7b642f7ac5a7642a8
    resource: repo://libs/core/langchain_core/language_models/chat_models.py
  - id: openwiki-source-a0aef6917b7e1f4a06e6db95
    resource: repo://libs/core/langchain_core/language_models/model_profile.py
generated: { by: "openwiki/0.5.0", at: "2026-09-03T15:18:34.589Z" }
---

> 🌐 本文档由 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 翻译,英文原版见原项目。
>
> ⚠️ 原文超过 10000 字符,本页翻译核心章节;代码块、行号引用保持原样,完整细节见英文原版。

## 总览

**聊天模型系统**是把大语言模型接入 LangChain 应用的核心接口。`BaseChatModel` 是所有聊天模型实现继承的抽象协议,定义了同步/异步 invoke 与流式行为、回调集成、限流、结构化输出绑定,以及通过模型画像进行能力发现的契约。

聊天模型把对话消息历史转换为 AI 响应,同时支持简单生成(`invoke`)与流式输出(`stream`)。框架统一了同步/异步模式,透明处理缓存,依据配置和挂载的回调路由到流式或非流式后端,并通过方法重写提供自定义行为的扩展点。

## 核心接口:BaseChatModel

**位置**:`repo://libs/core/langchain_core/language_models/chat_models.py#L284-L2400`

`BaseChatModel` 继承自 `BaseLanguageModel[AIMessage]`,是一个接受 `LanguageModelInput`、产出 `AIMessage` 的 `Runnable`。它为继承而设计:实现必须重写 `_generate`(必需),可选重写 `_llm_type`、`_stream`、`_agenerate`。

### 输入与输出类型

**LanguageModelInput**(`repo://libs/core/langchain_core/language_models/base.py#L140`)是一个联合类型:

```python
LanguageModelInput = PromptValue | str | Sequence[MessageLikeRepresentation]
```

- **字符串**:转换为 `StringPromptValue`(简单用户消息)
- **消息列表**:转换为 `ChatPromptValue`(完整对话历史)
- **PromptValue**:本身已是结构化提示(直接透传)

`_convert_input` 方法把所有输入形式规范化为 `PromptValue` 供下游处理。

**输出**:所有 invoke/stream 方法返回 `AIMessage`(流式为 `AIMessageChunk`)。聊天结果包装在 `ChatGeneration` 对象(含消息与生成元数据)中,聚合成 `ChatResult`。

### 同步方法

**`invoke`**(`repo://libs/core/langchain_core/language_models/chat_models.py#L474-L499`)是主要的同步入口:

```python
def invoke(
    self,
    input: LanguageModelInput,
    config: RunnableConfig | None = None,
    *,
    stop: list[str] | None = None,
    **kwargs: Any,
) -> AIMessage
```

- 把输入转为 `PromptValue`,再转为消息
- 调用 `generate_prompt`(内部调用 `_generate_with_cache`)
- 提取并返回第一个生成的消息
- 从 config 传播 `run_id`、callbacks、tags 与 metadata

**`stream`**(`repo://libs/core/langchain_core/language_models/chat_models.py#L726-L856`)随数据到达逐个产出 `AIMessageChunk`:

```python
def stream(
    self,
    input: LanguageModelInput,
    config: RunnableConfig | None = None,
    *,
    stop: list[str] | None = None,
    **kwargs: Any,
) -> Iterator[AIMessageChunk]
```

- 通过 `_should_stream()` 判断是否启用且实现了流式
- 流式被禁用或未实现时回退到 `invoke`
- 对支持流式的模型,直接调用 `_stream()` 并逐块产出
- 输出包在回调生命周期里:`on_chat_model_start`、`on_llm_new_token`(每块)、`on_llm_end` 或 `on_llm_error`
- 配置了限流器则应用限流
- 规范化消息并处理流式特有的输出格式(如 `output_version="v1"`)
- 流式结束时产出 `chunk_position="last"` 的最终空块

### 异步方法

**`ainvoke`**(`repo://libs/core/langchain_core/language_models/chat_models.py#L501-L523`)是异步变体:

```python
async def ainvoke(
    self,
    input: LanguageModelInput,
    config: RunnableConfig | None = None,
    *,
    stop: list[str] | None = None,
    **kwargs: Any,
) -> AIMessage
```

- 等待 `agenerate_prompt`
- 其余行为与 `invoke` 一致

**`astream`**(`repo://libs/core/langchain_core/language_models/chat_models.py#L857-L990`)是异步流式变体:

```python
async def astream(
    self,
    input: LanguageModelInput,
    config: RunnableConfig | None = None,
    *,
    stop: list[str] | None = None,
    **kwargs: Any,
) -> AsyncIterator[AIMessageChunk]
```

- 检查 `_should_stream(async_api=True)` 以路由到 `_astream` 或回退
- 其余与 `stream` 一致,但用异步回调分发

## 流式架构

### 流式决策逻辑

**`_should_stream()`**(`repo://libs/core/langchain_core/language_models/chat_models.py#L549-L585`)决定是否走流式代码路径:

```python
def _should_stream(
    self,
    *,
    async_api: bool,
    run_manager: CallbackManagerForLLMRun | AsyncCallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> bool
```

满足以下条件返回 `True`:
1. 流式未被禁用(`_streaming_disabled()` 返回 `False`)
2. 对应变体(同步/异步)实现了流式方法
3. 以下任一成立:
   - 显式传入 `stream=True` kwarg
   - 实例级 `streaming=True` 属性
   - 挂载了 v1 风格的 `_StreamingCallbackHandler`

以下情况返回 `False`(回退非流式):
- `disable_streaming=True`(硬禁用)
- `disable_streaming="tool_calling"` 且传入了工具
- 显式 `stream=False`
- 流式未实现且异步回退到同步

### 流式实现方法

**`_stream()`**(`repo://libs/core/langchain_core/language_models/chat_models.py#L2255-L2273`)是同步流式钩子(可选重写):

```python
def _stream(
    self,
    messages: list[BaseMessage],
    stop: list[str] | None = None,
    run_manager: CallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> Iterator[ChatGenerationChunk]
```

- 子类重写以实现原生流式
- 默认抛 `NotImplementedError`(回退到 `_generate`)
- 接收 run_manager 以逐 token 触发回调

**`_astream()`**(`repo://libs/core/langchain_core/language_models/chat_models.py#L2275-L2311`)是异步流式钩子(可选重写):

```python
async def _astream(
    self,
    messages: list[BaseMessage],
    stop: list[str] | None = None,
    run_manager: AsyncCallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> AsyncIterator[ChatGenerationChunk]
```

- 默认实现在 executor 中运行 `_stream()` 并产出结果
- 子类可重写以实现原生异步流式

### ChatModelStream 与 AsyncChatModelStream

**位置**:`repo://libs/core/langchain_core/language_models/chat_model_stream.py`

对 v3 事件协议(`stream_events(version="v3")`),模型返回 `ChatModelStream`(同步)或 `AsyncChatModelStream`(异步),为增量内容暴露**类型化投影**:

- **`.text`**:累积文本内容块
- **`.reasoning`**:累积推理/思维链内容
- **`.tool_calls`**:累积解析后的工具调用块
- **`.usage`**:累积 token 用量
- **`.output`**:最终组装的 `AIMessage`

每个投影都可迭代获取增量,或等待最终值。内部由累加器跟踪协议事件并合并为结构化输出。

## 生成与缓存

### 核心生成方法

**`_generate()`**(`repo://libs/core/langchain_core/language_models/chat_models.py#L2208-L2226`)是所有子类**必须实现的抽象方法**:

```python
@abstractmethod
def _generate(
    self,
    messages: list[BaseMessage],
    stop: list[str] | None = None,
    run_manager: CallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> ChatResult
```

- 调用底层模型 API
- 返回含 `ChatGeneration` 列表的 `ChatResult`
- 内部处理错误或向上传播
- 接收规范化后的消息与用于回调的 run manager

**`_agenerate()`**(`repo://libs/core/langchain_core/language_models/chat_models.py#L2228-L2253`)是可选的异步重写:

```python
async def _agenerate(
    self,
    messages: list[BaseMessage],
    stop: list[str] | None = None,
    run_manager: AsyncCallbackManagerForLLMRun | None = None,
    **kwargs: Any,
) -> ChatResult
```

- 默认实现在 executor 中运行 `_generate`
- 子类重写以支持原生异步 API

### 缓存生成

**`_generate_with_cache()`** 与 **`_agenerate_with_cache()`** 在核心方法外包装了:

1. **提示词缓存**:检查 `self.cache` 或全局 `get_llm_cache()` 是否已有该输入的缓存结果
2. **缓存命中**:返回缓存的生成;若挂载 v2 处理器,则以 v2 事件形式重放
3. **缓存未命中**:路由到流式或非流式路径
4. **协议路由**:分发到 v2 事件(`_should_use_protocol_streaming`)或 v1 回调路径(`_should_stream`)

### 批量方法

**`generate()`** 与 **`agenerate()`** 接受消息列表的列表,利用内部缓存/流式批量处理提示词:

```python
def generate(
    self,
    messages: list[list[BaseMessage]],
    stop: list[str] | None = None,
    callbacks: Callbacks = None,
    **kwargs: Any,
) -> LLMResult
```

返回 `LLMResult`,生成结果按输入提示词分组,并合并 llm_output。

## 回调生命周期

聊天模型与回调系统集成,在执行全程发出结构化事件:

### LLM 运行生命周期

1. **`on_chat_model_start`**(或回退 `on_llm_start`):
   - `invoke`、`stream` 或 `generate` 开始时触发
   - 接收序列化的模型配置、格式化后的输入消息、调用参数和批量大小
   - 返回绑定到本次操作的 run manager
    
2. **`on_llm_new_token`**(仅流式):
   - 每个流式 token/块触发一次
   - 接收 token 字符串与 `ChatGenerationChunk` 元数据
   - 支持实时输出捕获

3. **`on_llm_end`**:
   - 生成成功完成时触发
   - 接收含全部生成与元数据的最终 `LLMResult`

4. **`on_llm_error`**:
   - 生成抛异常时触发
   - 接收异常和部分 `LLMResult`(如有)
   - `_generate_response_from_error()` 从 HTTP 错误中提取响应元数据

5. **`on_stream_event`**(v2/v3 协议):
   - 流式期间每个内容块协议事件触发
   - 支持细粒度事件观察,用于高级追踪

### 回调配置

回调通过 `RunnableConfig` 配置:

```python
config = {
    "callbacks": [my_handler],  # Callbacks for this run
    "tags": ["agent", "tools"],  # Labels for filtering
    "metadata": {"user_id": "123"},  # Context data
    "run_name": "my_run",  # Human-readable run name
    "run_id": uuid.uuid4(),  # Explicit run ID (optional)
}
result = model.invoke(input, config=config)
```

可继承的元数据和 LangSmith 参数通过 `_get_invocation_params()` 与 `_get_ls_params()` 提取。

## 结构化输出与工具绑定

### with_structured_output()

**位置**:`repo://libs/core/langchain_core/language_models/chat_models.py#L2385-L2565`

`with_structured_output()` 包装聊天模型,把输出约束到指定 schema:

```python
def with_structured_output(
    self,
    schema: dict[str, Any] | type,
    *,
    include_raw: bool = False,
    **kwargs: Any,
) -> Runnable[LanguageModelInput, dict[str, Any] | BaseModel]
```

**工作原理**:

1. 委托给 `bind_tools([schema], tool_choice="any", ...)`
2. 把结果接入输出解析器:
   - schema 是 Pydantic 类:`PydanticToolsParser` → Pydantic 实例
   - schema 是 dict:`JsonOutputKeyToolsParser` → dict
3. `include_raw=True`:输出包装为 `{"raw": AIMessage, "parsed": ..., "parsing_error": ...}`
4. 解析失败且 `include_raw=False`:抛异常

**前提**:要求模型实现 `bind_tools()`(并非所有模型都支持)。

### bind_tools()

**位置**:`repo://libs/core/langchain_core/language_models/chat_models.py#L2366-L2383`

```python
def bind_tools(
    self,
    tools: Sequence[dict[str, Any] | type | Callable[..., Any] | BaseTool],
    *,
    tool_choice: str | None = None,
    **kwargs: Any,
) -> Runnable[LanguageModelInput, AIMessage]
```

- 抽象方法;支持工具调用的子类必须实现
- 把工具列表绑定到模型
- 返回一个在 API 请求中携带工具定义的绑定 runnable
- `tool_choice="any"` 强制模型至少调用一个工具

## 模型画像与能力

**位置**:`repo://libs/core/langchain_core/language_models/model_profile.py`

`BaseChatModel` 的 `profile` 字段保存模型能力元数据:

```python
class ModelProfile(TypedDict, total=False):
    # Metadata
    name: str  # Human-readable model name
    status: str  # 'active', 'deprecated', etc.
    release_date: str  # ISO 8601
    last_updated: str  # ISO 8601
    open_weights: bool  # Weights publicly available?
    
    # Input constraints
    max_input_tokens: int  # Context window size
    text_inputs: bool
    image_inputs: bool
    image_url_inputs: bool
    pdf_inputs: bool
    audio_inputs: bool
    video_inputs: bool
    image_tool_message: bool  # Images in ToolMessage?
    pdf_tool_message: bool  # PDFs in ToolMessage?
    
    # Output constraints
    max_output_tokens: int
    text_outputs: bool
    image_outputs: bool
    audio_outputs: bool
    video_outputs: bool
    
    # Capabilities
    tool_calling: bool  # Supports function calling?
    tool_choice: bool  # Supports tool_choice parameter?
    tool_call_streaming: bool  # Returns structured tool_call_chunks when streaming?
    structured_output: bool  # Native structured output support?
    reasoning_output: bool  # Reasoning/chain-of-thought?
    reasoning_effort_levels: list[str]  # ['low', 'medium', 'high']
    reasoning_effort_default: str
    temperature: bool  # Supports temperature parameter?
    attachment: bool  # Supports file attachments?
```

**自动加载**:画像通过 `_resolve_model_profile()`(子类重写)解析并缓存在 `profile` 字段。无法识别的键会经 `_warn_unknown_profile_keys()` 触发警告。

### Partner 模式集成

Partner 包(如 `langchain-openai`)重写 `_resolve_model_profile()`,从自己的画像数据加载模型专属元数据。基类校验器 `_set_model_profile`(Pydantic mode="after")在未显式设置时自动填充该字段。

## 配置与状态

### 核心字段

```python
class BaseChatModel(BaseLanguageModel[AIMessage], ABC):
    rate_limiter: BaseRateLimiter | None = Field(default=None, exclude=True)
    
    disable_streaming: bool | Literal["tool_calling"] = False
    # False: use streaming if available
    # True: always use non-streaming (invoke)
    # "tool_calling": use non-streaming only when tools are passed
    
    output_version: str | None = None
    # 'v0': provider-specific format (lazy-parse via content_blocks)
    # 'v1': standardized format (merged into content)
    
    profile: ModelProfile | None = Field(default=None, exclude=True)
    # Capability metadata (auto-loaded if not provided)
    
    cache: BaseCache | None = None  # Inherited from BaseLanguageModel
    callbacks: list[BaseCallbackHandler] | None = None
    verbose: bool = False
    tags: list[str] | None = None
    metadata: dict[str, Any] | None = None
```

- `disable_streaming`:`False` 表示可用时走流式;`True` 表示总是非流式(invoke);`"tool_calling"` 表示仅在传入工具时用非流式
- `output_version`:`'v0'` 为供应商特有格式(经 content_blocks 惰性解析);`'v1'` 为标准化格式(合并进 content)
- `profile`:能力元数据(未提供时自动加载)

### 必需属性

- **`_llm_type`**(property,抽象):模型类型唯一标识(如 `"openai"`、`"anthropic"`)
- **`_identifying_params`**(property,可选):用于追踪的模型配置字典(如 `{"model": "gpt-4", "temperature": 0.7}`)

## 实现要求

子类必须实现:

| 方法/属性 | 说明 | 必需 | 备注 |
|---|---|---|---|
| `_generate()` | 核心生成逻辑 | ✓ | 调用供应商 API,返回 `ChatResult` |
| `_llm_type` | 模型类型标识 | ✓ | 如 `"openai"`、`"anthropic"` |
| `_identifying_params` | 追踪用配置字典 | ✗ | 被 `_get_llm_string()` 与序列化使用 |
| `_stream()` | 同步流式 | ✗ | 可选;未实现时 stream 回退到 invoke |
| `_agenerate()` | 原生异步生成 | ✗ | 可选;默认在 executor 中运行 `_generate` |
| `_astream()` | 原生异步流式 | ✗ | 可选;默认在 executor 中运行 `_stream` |
| `bind_tools()` | 结构化输出的工具绑定 | ✗ | 仅当需要 `with_structured_output()` 时必需 |

## 模型初始化

**位置**:`repo://libs/langchain_v1/langchain/chat_models/base.py`(v1 兼容)及 langchain_core 各 partner 包

模型实例化方式:

1. **直接实例化**:`ChatOpenAI(model="gpt-4", temperature=0)`
2. **工厂函数 `init_chat_model()`**:自动识别供应商并动态导入类
3. **Partner 包导出**:每个供应商(如 `langchain-openai`)导出具体模型类

`init_chat_model()` 接受模型名字符串(如 `"gpt-4"`、`"claude-3-sonnet"`)和可选 `model_provider`,无需显式导入即可实例化正确的类。

## 示例:自定义聊天模型

```python
from langchain_core.language_models.chat_models import BaseChatModel
from langchain_core.messages import BaseMessage, AIMessage
from langchain_core.outputs import ChatResult, ChatGeneration
from langchain_core.callbacks import CallbackManagerForLLMRun

class MyCustomChatModel(BaseChatModel):
    """Custom chat model for demonstration."""
    
    model_name: str = "my-model"
    temperature: float = 0.7
    
    def _generate(
        self,
        messages: list[BaseMessage],
        stop: list[str] | None = None,
        run_manager: CallbackManagerForLLMRun | None = None,
        **kwargs: Any,
    ) -> ChatResult:
        """Generate a response from the messages."""
        # Call your model API here
        response_text = f"Echo: {messages[-1].content}"
        
        message = AIMessage(content=response_text)
        generation = ChatGeneration(message=message)
        return ChatResult(generations=[generation])
    
    def _stream(
        self,
        messages: list[BaseMessage],
        stop: list[str] | None = None,
        run_manager: CallbackManagerForLLMRun | None = None,
        **kwargs: Any,
    ) -> Iterator[ChatGenerationChunk]:
        """Stream tokens from the model."""
        text = f"Echo: {messages[-1].content}"
        for char in text:
            chunk = ChatGenerationChunk(
                message=AIMessageChunk(content=char)
            )
            yield chunk
    
    @property
    def _llm_type(self) -> str:
        """Return the model type identifier."""
        return "my-custom-model"
    
    @property
    def _identifying_params(self) -> dict[str, Any]:
        """Return identifying parameters for tracing."""
        return {
            "model_name": self.model_name,
            "temperature": self.temperature,
        }
```

## 高级模式

### 带回调的流式

```python
from langchain_core.callbacks import StreamingStdOutCallbackHandler

handler = StreamingStdOutCallbackHandler()
config = {"callbacks": [handler]}

# Streams token-by-token to stdout
for chunk in model.stream("Tell me a joke", config=config):
    pass  # Handler prints as chunks arrive
```

### 带校验的结构化输出

```python
from pydantic import BaseModel

class Answer(BaseModel):
    text: str
    confidence: float

structured_model = model.with_structured_output(Answer)
result = structured_model.invoke("What is 2+2?")  # -> Answer(text="4", confidence=0.99)
```

### 缓存与限流

```python
from langchain_core.caches import InMemoryCache
from langchain_core.rate_limiters import InMemoryRateLimiter

model = ChatOpenAI(
    model="gpt-4",
    cache=InMemoryCache(),  # Cache results
    rate_limiter=InMemoryRateLimiter(requests_per_second=10)  # Limit requests
)

# Subsequent identical calls hit the cache
result1 = model.invoke("Hello")
result2 = model.invoke("Hello")  # Cached, no API call
```

### 条件流式

```python
model_with_fallback = ChatOpenAI().with_fallbacks([ChatAnthropic()])

# Use streaming only when a handler requests it
config = {"callbacks": [MyStreamingHandler()]}
model_with_fallback.invoke("Prompt", config=config)
```

## 关键不变量与保证

1. **输入规范化**:所有输入形式(字符串、消息列表、PromptValue)在调用 `_generate`/`_stream` 前都规范化为消息。

2. **消息 ID**:每个流式消息块和最终消息都有唯一 ID(派生自 run_id)用于追踪。

3. **回调顺序**:回调按序触发:`on_chat_model_start` → `on_llm_new_token`(每块)→ `on_llm_end` 或 `on_llm_error`。

4. **流式回退**:流式未实现或被禁用时,`stream` 无缝回退到 `invoke`,把结果作为单个块产出。

5. **缓存透明**:缓存命中完全透明 —— 触发的生命周期回调与未命中相同。

6. **异步/同步等价**:异步方法镜像同步行为;默认异步实现在 executor 中运行同步方法。

7. **响应元数据**:每个生成把元数据(token、finish_reason 等)累积在 `message.response_metadata`。

8. **错误处理**:生成期间的异常触发 `on_llm_error` 并向调用方传播;如可用,会从 HTTP 响应中提取错误元数据。
