---
type: "Reference"
title: "Form 1: No arguments (name from function)"
openwiki_generated: true
verified:
  - by: openwiki/0.5.0
    at: 2026-09-03T15:18:34.589Z
sources:
  - id: openwiki-source-9861ba5cf0c42c142cf732f9
    resource: repo://libs/core/langchain_core/messages/tool.py
  - id: openwiki-source-4ff475d7b00540f962384251
    resource: repo://libs/core/langchain_core/tools/base.py
  - id: openwiki-source-9c422fcb5ac12738f17d1cd1
    resource: repo://libs/core/langchain_core/tools/convert.py
  - id: openwiki-source-1ab4436ccb637ddf41e35732
    resource: repo://libs/core/langchain_core/tools/render.py
  - id: openwiki-source-80e84f93417c922f44011393
    resource: repo://libs/core/langchain_core/tools/simple.py
  - id: openwiki-source-b816e651a5890bde13cf8013
    resource: repo://libs/core/langchain_core/tools/structured.py
generated: { by: "openwiki/0.5.0", at: "2026-09-03T15:18:34.589Z" }
---

> 🌐 本文档由 [langchain-ai/langchain](https://github.com/langchain-ai/langchain) 翻译,英文原版见原项目。
>
> ⚠️ 原文超过 10000 字符,本页翻译核心章节;代码块保持原样,完整细节见英文原版。


## 总览

LangChain 的工具系统通过把 Python 函数和 Runnable 转换为带 schema 感知的组件,让智能体和语言模型能够执行结构化动作。工具是智能体工作流的核心执行机制,提供自动参数校验、错误处理以及与回调系统的集成。

工具生态由三层组成:

1. **BaseTool**:定义工具协议与执行语义的核心抽象接口
2. **工具类型**:面向不同输入模式的具体实现(StructuredTool、Tool)
3. **工具创建**:从函数和 runnable 生成工具的装饰器与工厂(@tool、convert_runnable_to_tool)

## BaseTool 协议与核心职责

BaseTool 是继承 RunnableSerializable 的抽象基类,定义所有工具的契约。每个工具都携带三个必备描述符以及执行控制配置。

**必需属性:**
- `name: str` — 清晰表达用途的唯一标识;智能体和模型靠它选择工具
- `description: str` — 说明何时以及为何使用该工具的人类可读文本;引导模型决策
- `args_schema: TypeBaseModel | dict | None` — 指定合法输入参数的 Pydantic 模型或 JSON schema 字典

**执行控制:**
- `return_direct: bool` — 为 True 时,智能体在工具执行后立即停止循环(终态动作)
- `response_format: "content" | "content_and_artifact"` — 若为 "content_and_artifact",工具必须返回二元组 (content, artifact),实现结构化输出并附带工件
- `handle_tool_error: bool | str | Callable` — ToolException 处理策略:False(重新抛出)、True(用异常消息)、字符串(固定消息)或可调用对象(自定义处理)
- `handle_validation_error: bool | str | Callable` — 输入解析期间 pydantic ValidationError 的处理策略

**回调与元数据:**
- `callbacks: Callbacks` — 生命周期回调(on_tool_start、on_tool_end、on_tool_error),用于追踪和监控
- `tags: list[str]` — 附加到所有调用的可选语义标签,用于过滤和统计
- `metadata: dict` — 传给回调的应用自定义元数据
- `verbose: bool` — 是否记录工具执行进度

**供应商集成:**
- `extras: dict[str, Any]` — 供应商专属配置(如 Anthropic 的 cache_control、defer_loading),在工具渲染时传给聊天模型

## 输入 Schema 生成与校验

工具输入校验建立在由函数签名生成的 Pydantic 模型上。schema 生成管线同时支持自动推断与显式指定。

**Schema 来源(按优先级):**
1. 提供给工具装饰器或工厂的显式 `args_schema` 参数
2. 若 `args_schema` 已是 dict,则直接作为 JSON schema
3. 通过 `create_schema_from_function()` 从函数签名推断

**推断过程:**

当 `infer_schema=True`(默认)时,工具检查函数签名生成 Pydantic 模型:

- 通过 `get_type_hints()` 提取类型标注,支持 `Annotated` 类型
- 解析函数 docstring(当 `parse_docstring=True`),按 Google 风格提取参数描述
- 描述合并顺序:Annotated 元数据 → docstring Args 小节 → 无
- 注入参数(标注 `InjectedToolArg`、`InjectedToolCallId` 或 `ToolRuntime` 的参数)自动从发给模型的 schema 中排除,并在运行时重新注入
- 保留参数名(`run_manager`、`callbacks`、`config`)会从面向用户的 schema 中过滤掉

**记忆化:** `tool_call_schema` 属性为每个工具实例构建并缓存一个子集模型类(排除注入参数)。该 schema 类的 `model_json_schema()` 方法被修补为缓存生成的 dict,避免每轮智能体循环都做昂贵重建。

**输入解析与校验:**

执行期间,工具输入由 `_parse_input()` 解析:
- 字符串输入在 schema 只有一个字段时映射到该参数
- 字典输入经 Pydantic 校验,Annotated 描述提供字段文档
- 注入参数通过签名检查识别,并从工具元数据或调用上下文重新注入(如 `tool_call_id`、`ToolRuntime`)
- 校验错误被捕获并按 `handle_validation_error` 配置处理

**注解驱动的描述:**

参数描述可来自 Annotated 字段元数据:

```python
from typing import Annotated
from pydantic import Field
from langchain_core.tools import tool

@tool
def my_function(
    query: Annotated[str, Field(description="The search query")],
    limit: Annotated[int, "Maximum number of results"] = 10
) -> str:
    return f"search: {query}"
```

`Field(description=...)` 和直接字符串标注都支持,并合并进生成的 schema。

## ToolCall 与 ToolMessage:请求-响应协议

工具通过 ToolCall 对象调用,并以 ToolMessage 对象响应,在智能体循环中实现结构化通信。

**ToolCall(来自 messages/tool.py):**

ToolCall 是表示模型执行工具请求的 TypedDict:

```python
{
    "name": "search_tool",        # Tool name to invoke
    "args": {"query": "python"},  # Validated arguments as dict
    "id": "call_abc123",          # Unique ID for pairing with response
    "type": "tool_call"           # Discriminator
}
```

多个 ToolCall 可通过 `AIMessageChunk` 流式传输与合并;流式产出 `ToolCallChunk` 对象,逐步拼出参数 JSON 字符串。

**ToolMessage(来自 messages/tool.py):**

工具用它把结果传回模型:

```python
ToolMessage(
    content="Result of the tool execution",
    tool_call_id="call_abc123",  # Must match ToolCall.id
    name="search_tool",           # Tool name (optional)
    artifact={"raw": "data"},     # Unshown to model (optional)
    status="success"              # "success" or "error"
)
```

- `artifact`:当只给模型发摘要时,存放完整工具输出
- `status`:允许工具不抛异常就上报错误(如 `handle_tool_error=True` 时)
- content 支持富格式:纯文本或消息内容块列表(图片、JSON、搜索结果、文档等)

**ToolOutputMixin:** 一个空 mixin 类,用于标识工具可以直接返回、不必强转为字符串的自定义对象。工具可返回 ToolOutputMixin 实例或其列表,绕过自动的 ToolMessage 包装。

## 执行:run() 与 arun() 方法

同步与异步执行遵循同一生命周期:

1. **配置:** 合并来自工具配置、调用参数和 runnable 配置的回调
2. **解析:** 通过 `_parse_input()` 与 `_to_args_and_kwargs()` 把工具输入(str/dict/ToolCall)转为函数 args/kwargs
3. **注入:** 若函数签名声明了运行时值(run_manager、callbacks、RunnableConfig),则注入
4. **执行:** 在回调上下文中调用 `_run()` 或 `_arun()`,经上下文变量传递配置
5. **格式化:** 若带 `tool_call_id` 调用,把输出转为 ToolMessage,保留 status 与 artifact
6. **错误处理:** 捕获 ToolException 与 ValidationError,应用处理策略(重抛、返回消息或调用自定义处理)

**回调生命周期:**

- `on_tool_start()`:执行前触发,输入已过滤(移除注入参数),附工具元数据和 trace ID
- `on_tool_end()`:成功后触发,附格式化输出
- `on_tool_error()`:异常时触发,附异常与 trace ID

**配置传播:**

传给 invoke/ainvoke 的 RunnableConfig 会被补上子回调并注入工具执行上下文,使嵌套工具以及经 `ToolRuntime` 参数访问 state/store 成为可能。

## 工具类型:Tool 与 StructuredTool

LangChain 提供两个输入处理语义不同的具体实现。

**Tool(simple.py):**
- 单输入工具,接受字符串或可强转为字符串的 dict
- 无需显式 args schema;默认 `{"tool_input": {"type": "string"}}`
- 用于简单函数包装和遗留兼容
- 校验 schema 解析后恰好只传一个参数

**StructuredTool(structured.py):**
- 多参数工具,按显式 schema 驱动解析
- 每个函数参数成为独立的 schema 字段(注入参数除外)
- 同时支持 `func`(同步)与 `coroutine`(异步)
- 未定义 coroutine 时,同步调用回退到 executor
- 多命名参数的智能体工具首选模式

两者都继承 BaseTool,重写 `_run()` 和 `_arun()`,委托给被包装的函数,同时保留配置与回调。

## 工具创建:@tool 装饰器与工厂

`@tool` 装饰器是把函数转为工具的主要用户 API,多个重载支持多种用法。

**装饰器形式:**

```python
# Form 1: No arguments (name from function)
@tool
def search(query: str) -> str:
    """Search the API."""
    return f"Results for {query}"

# Form 2: With parameters
@tool(description="Custom description", return_direct=True)
def calculate(expression: str) -> str:
    return str(eval(expression))

# Form 3: With explicit name
@tool("my_search")
def search(query: str) -> str:
    return query

# Form 4: With Runnable
tool_obj = tool("math_tool", my_runnable, description="...")
```

**关键行为:**

- 默认名称为 `function.__name__`,除非显式覆盖
- 描述优先级:显式参数 → 函数 docstring → args_schema 描述
- `parse_docstring=True` 提取 Google 风格 Args 小节作为参数描述(并校验文档参数与签名一致)
- `infer_schema=True`(默认)从类型标注自动生成 schema
- `infer_schema=False` 要求显式描述,并创建 Tool(字符串输入)而非 StructuredTool
- `response_format="content_and_artifact"` 要求函数返回 `(content, artifact)` 元组

**Runnable 转换:**

装饰 Runnable 时,工具会自动:
- 包装同步/异步 invoke 方法以注入回调
- 用 Runnable 的 input_schema 作为工具的 args_schema
- 未提供描述时从输入 schema 生成
- 多参数 runnable 委托给 StructuredTool.from_function(),字符串 schema 则用 Tool

**异步支持:**

装饰器检测 coroutine,创建同时设置 `func` 与 `coroutine` 的 StructuredTool,实现真异步执行。混合同步/异步模式经 executor 回退可用。

## 面向模型的 Schema 渲染

工具经 `render.py` 中的工具函数渲染给语言模型:

- `render_text_description(tools: list[BaseTool]) -> str` — 返回 `"name - description\n..."` 格式,用于提示词
- `render_text_description_and_args(tools: list[BaseTool]) -> str` — 含参数:`"name - description, args: {...}"`

模型以供应商专属格式(OpenAI function_calling、Anthropic tool_use 等)接收工具 schema,由 `function_calling.py` 工具函数把 tool_call_schema 转换为带 JSON schema 参数的 FunctionDescription 字典。

`tool_call_schema` 属性保证模型永远看不到注入参数或保留参数名,保护工具实现细节。

## 高级模式:注入参数与 ToolRuntime

工具可以接收不受模型控制的运行时值,即注入参数。

**InjectedToolArg:** 标记应在运行时注入的参数:

```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolArg

@tool
def my_tool(
    user_query: str,
    context_var: Annotated[str, InjectedToolArg]
) -> str:
    # context_var is injected; user only provides user_query
    return f"{user_query} in {context_var}"
```

**InjectedToolCallId:** 专门注入 tool_call_id 的标记:

```python
@tool
def track_call(query: str, call_id: InjectedToolCallId) -> str:
    # call_id automatically populated with tool_call_id from invocation
    return f"Call {call_id}: {query}"
```

**ToolRuntime:** 直接注入的参数类型,提供对 state、context 和 store 的访问:

```python
from langchain_core.tools import tool, ToolRuntime

@tool
def stateful_tool(query: str, runtime: ToolRuntime) -> str:
    # Access application state, context, and LangGraph store
    state = runtime.state
    context = runtime.context
    store = runtime.store
    return f"State: {state}, Context: {context}"
```

注入参数:
- 不会出现在发给模型的 tool_call_schema 中
- 由 `_get_injected_args_keys_from_signature()` 经签名检查识别
- 在 `_parse_input()` 期间从工具元数据或调用上下文重新注入
- 经 `_filter_injected_args()` 从回调输入中过滤

## 错误处理策略

工具支持灵活的错误处理,让智能体循环能优雅恢复。

**ToolException:** 受控工具错误的自定义异常:

```python
from langchain_core.tools import tool, ToolException

@tool
def validate_input(value: str) -> str:
    if not value:
        raise ToolException("Value cannot be empty")
    return f"Valid: {value}"
```

**校验错误:** Pydantic 校验失败被捕获并按 `handle_validation_error` 处理:

```python
@tool(handle_validation_error="Invalid input format")
def my_tool(count: int) -> str:
    return f"Count: {count}"

# If user passes non-integer, returns "Invalid input format" instead of raising
```

**工具错误:** ToolException 按 `handle_tool_error` 处理:

```python
@tool(handle_tool_error=True)  # Use exception message
def risky_operation() -> str:
    raise ToolException("Operation failed")
    
# Returns ToolMessage with status="error", content="Operation failed"
```

自定义处理器接收异常,返回字符串或消息内容块列表:

```python
def my_error_handler(e: ToolException) -> str:
    logger.error(f"Tool failed: {e}")
    return "Operation failed. Please try again later."

@tool(handle_tool_error=my_error_handler)
def operation() -> str:
    raise ToolException("Internal error")
```

带 `tool_call_id` 调用时,已处理的错误返回 `status="error"` 的 ToolMessage,让智能体可以观察并响应失败而不中断循环。

## BaseToolkit:组织相关工具

复杂系统中,工具经 `BaseToolkit` 组织为工具箱:

```python
from langchain_core.tools import BaseToolkit, tool

class MathToolkit(BaseToolkit):
    """Toolkit for mathematical operations."""
    
    @property
    def description(self) -> str:
        return "Tools for arithmetic and algebra"
    
    def get_tools(self) -> list[BaseTool]:
        @tool
        def add(a: int, b: int) -> int:
            return a + b
        
        @tool
        def multiply(a: int, b: int) -> int:
            return a * b
        
        return [add, multiply]

toolkit = MathToolkit()
tools = toolkit.get_tools()  # Retrieve all related tools
```

工具箱支持:
- 相关功能的逻辑分组
- 条件化的工具可用性(按运行时状态返回子集)
- 动态工具生成
- 与智能体初始化管线集成

## Runnable 转工具

Runnable 可通过 `tool()` 装饰器或 `convert_runnable_to_tool()` 函数转为工具:

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.tools import convert_runnable_to_tool

my_runnable = RunnablePassthrough()

# Via convert function
tool_obj = convert_runnable_to_tool(
    my_runnable,
    name="passthrough",
    description="Passes input through unchanged"
)

# Via decorator
tool_obj = tool("passthrough", my_runnable)
```

转换过程:
- 从 Runnable.get_input_jsonschema() 提取 input_schema
- 校验 schema 为 object 类型(多参数工具必需)
- 包装 invoke/ainvoke,把回调注入 config
- 用包装后的函数委托给 StructuredTool.from_function()
- 字符串输入的 runnable 回退到 Tool

## 生命周期与不变量

**工具实例生命周期:**

1. **构造:** 若 name/description/args_schema 经 `__setattr__` 或 `model_copy()` 变化,schema 记忆化缓存被清除
2. **首次访问 schema:** tool_call_schema 构建子集模型,并修补类以缓存 JSON schema
3. **执行:** 配置回调、解析输入、识别注入参数、调用函数、格式化输出
4. **Pickle:** 清除 schema 缓存(动态类无法按引用 pickle);下次访问时重建

**Schema 缓存不变量:**

- name/description/args_schema 不变时,记忆化的子集模型类绝不重建
- 对子集类调用 Pydantic model_json_schema(),后续调用返回缓存 dict
- 缓存失效通过私有 _TOOL_CALL_SCHEMA_FIELDS 检查显式触发
- 在高频智能体循环下保持性能

**执行不变量:**

- 回调始终按序触发:on_tool_start → (on_tool_error | on_tool_end)
- 执行期间设置配置上下文,嵌套工具可访问 state/store
- 只有提供 tool_call_id 时才包装为 ToolMessage
- 注入参数对模型和回调输入均不可见
- 仅当 handle_tool_error 把异常转为消息时才置 status="error"

## 扩展点

**继承 BaseTool:**

自定义工具实现重写:
- `_run(self, *args, **kwargs) -> Any` — 同步执行逻辑
- `_arun(self, *args, **kwargs) -> Any` — 异步执行逻辑(默认经 executor 委托给 _run)
- `get_input_schema()` — 覆盖 schema 来源(默认用 args_schema 或从 _run 签名创建)

**自定义错误处理器:**

作为可调用对象传给 `handle_tool_error` 和 `handle_validation_error`:

```python
def custom_validation_handler(e: ValidationError) -> str:
    # Extract user-friendly message from Pydantic error
    return ", ".join(f"{err['loc'][0]}: {err['msg']}" for err in e.errors())

@tool(handle_validation_error=custom_validation_handler)
def my_tool(count: int) -> str:
    return str(count)
```

**回调管理器:**

工具注入 CallbackManager/AsyncCallbackManager 以支持:
- 自定义事件处理器(日志、指标、追踪)
- 带回调传播的嵌套工具执行
- 用于可观测性的 on_tool_start/on_tool_end 钩子

工具在 `_run()` 签名中暴露 run_manager,允许直接调用回调。

## 配置与运维注意事项

**保留参数名:**

名为 `config`、`run_manager` 或 `callbacks` 的参数会从工具 schema 中过滤,因为它们与 LangChain 的运行时注入冲突。要访问运行时状态请改用 `ToolRuntime` 注解。

**供应商 extras:**

`extras` 字典可传供应商专属配置:

```python
@tool(extras={"cache_control": {"type": "ephemeral"}})
def cached_operation(query: str) -> str:
    return query
```

聊天模型在渲染工具时检查 extras 并应用供应商专属行为。

**Docstring 解析:**

`parse_docstring=True` 时,解析 Google 风格 docstring 提取参数描述:

```python
@tool(parse_docstring=True)
def process(name: str, count: int) -> str:
    """Process items by name.
    
    Args:
        name: The item name
        count: Number of items to process
    """
    return f"{name}: {count}"
```

若 `error_on_invalid_docstring=True`,无效 docstring(缺 Args 小节、参数不在签名中、格式错误)会抛 ValueError。

**Verbose 输出:**

设 `verbose=True` 记录工具执行。配合回调管理器实现全面可观测。

## 总结:各模式适用场景

- **@tool 装饰器:** 函数转工具的首选;配合类型标注自动推断 schema
- **StructuredTool.from_function():** 装饰器语法不便或需要编程式创建工具时的直接工厂
- **Tool(简单):** 遗留兼容或单字符串输入工具
- **BaseToolkit:** 组织相关工具或动态生成工具
- **convert_runnable_to_tool():** 把既有 Runnable 包装为调用一致的实践工具
- **注入参数:** 共享运行时上下文(state、store、call ID)且对模型不可见
- **自定义错误处理器:** 把 Pydantic 或工具错误转换为对智能体友好的消息
