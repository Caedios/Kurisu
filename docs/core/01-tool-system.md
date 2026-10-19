# 工具系统 设计规范 
- **定位**：框架核心层工具原语与执行调度引擎 ( `src/my_agent_core/tools/` , `registry.py` )
- **核心类**：`Tool`, `@tool`, `ToolRegistry`, `ToolResult`, `ToolCall`
- **主要实现**：`tools/core.py`, `registry.py`, `tools/builtin/`

## 一、架构设计与定位

> 下面读完再来看这一段

## 二、核心类与机制
### 1. 结构化 `ToolCall` 实体（一等公民）
```python
class ToolCall(BaseModel):
    """统一结构化工具调用对象（由模型边界层输出）。"""
    id: str
    name: str
    args: dict[str, Any] = Field(default_factory=dict)
    error: str | None = None
```

- **消除 4x JSON Ping-Pong**：在传统实现中，模型层输出 JSON 字符串 ➔ 核心层解析为字典进行 Hook 检查 ➔ 核心层重新序列化为字符串传给注册表 ➔ 注册表再次反序列化传给函数。
- **现代化重塑**：模型边界层（ `my-agent-llm` ）直接交付已经完成反序列化的结构化 `ToolCall` （Anthropic 原生直接使用 `block.input` 字典），核心微内核 `loop.py` 与 `ToolRegistry` 全程直接消费原生字典，零多余序列化开销。

> 矛盾的根源：传统实现方式通过 `函数 <-> 参数` 交流，函数之间传输的都是参数
>   **作者的解决方式**：设置一等公民 `ToolCall` 实体，函数之间传输的都是对象
> ```
> ① 模型边界层（my-agent-llm）收到模型响应
>         │
>         │  在这里完成唯一的反序列化
>         ▼
> ② 构造 ToolCall 对象
>         ToolCall(id="call_1", name="search", args={"query": "Harness"})
>         │
>         ▼  直接传递对象，args 已经是字典
> ③ 微内核 loop.py 拿到 ToolCall，直接读 tc.args 做 Hook 检查
>         │
>         ▼  还是同一个字典，没有转换
> ④ ToolRegistry 拿到 ToolCall，直接 tc.args 传给工具函数
>         │
>         ▼
>    tool_function(**tc.args)
> ```

### 2. `@tool` 装饰器与 Pydantic 动态建模
```python
@tool(name="calculator", description="计算算术表达式", is_parallel_safe=True, timeout=10.0)
def calculate(expr: str, precision: int = 2) -> float:
    ...
```
- 底层使用 `pydantic.create_model` 从函数形参、类型注解与默认值动态合成参数验证模型；

> **实现方式：** 作者使用 `@tool` 去修饰 `calculate` 函数，实际上在底层实现如下（未考证）
> ```python
> CalculateParams = pydantic.create_model(
> 	"CalculateParams",
> 	expr = (str, Field(..., description = "需要计算的算术表达式")),
> 	precision = (int, Field(2, description = "保留的小数位数"))
> )
> 
> calculate = StructuredTool(
> 	name = "calculator",
> 	description = "计算算数表达式",
> 	args_schema = CalculateParams,
> 	func = lambda expr, precision = 2: ...,
> 	metadata = {"is_parallel_safe": True, "timeout": 10.0}
> )
> ```

- **保留参数过滤（ `_FRAMEWORK_RESERVED_PARAMS` ）**：自动从生成的 JSON Schema 中过滤掉框架注入保留参数（ `on_update` , `signal` ），防止大模型幻觉生成这些内部控制参数；

> **实现方式：** 将特定参数名 `on_update` 过滤，按照 `def calculate(expr: str, precision: int = 2) -> float: ...` 的例子来说判断 `param_name = "expr"` 是否在过滤参数中
> ```python
> _FRAMEWORK_RESERVED_PARAMS: frozenset[str] = frozenset({"on_update", "signal", "tool_call_id"})
> for param_name, param in inspect.signature(func).parameters.items():  
> 	if param_name in _FRAMEWORK_RESERVED_PARAMS:  
> 		continue
> ```

- 自动生成符合 OpenAI / Anthropic 规范的 Function Calling JSON Schema 字典；

> **实现方式：** 在 `pydanic.create_model()` 创建对象的时候可以自动返回
> ```python
> params_model = pydanic.create_model()
> schema = params_model.model_json_schema()
> ```

- 运行时在调用真实函数前通过 `params_model.model_validate` 进行强类型校验与宽松类型转换（如 `"37"` 自动转换为 `37` ）。

> **实现方式：** 
> ```python
> params = params_model.model_validate({"expr": "1+2", "precision": "37"})
> ```









