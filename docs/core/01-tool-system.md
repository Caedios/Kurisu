# 工具系统 设计规范 
- **定位**：框架核心层工具原语与执行调度引擎 ( `src/my_agent_core/tools/` , `registry.py` )
- **核心类**：`Tool`, `@tool`, `ToolRegistry`, `ToolResult`, `ToolCall`
- **主要实现**：`tools/core.py`, `registry.py`, `tools/builtin/`

## 一、架构设计与定位

工具系统是 Agent 与外部物理环境交互的手和脚。 `my-agent-core` 的工具系统深度对标 **Tau** 与 **Pi** 的微内核设计，强调四大核心设计不变式：

1. **自动提取契约**：基于 Pydantic v2 从 Python 函数签名动态提取标准化 Function Calling Schema；
2. **结构化 ToolCall 第一公民**：在模型边界层完成参数反序列化，微内核与注册表原生消费字典（ `args: dict` ），**彻底消灭 4 重 JSON 编解码死循环（4x JSON Ping-Pong）**；
3. **Never-Throw 强稳定性保证**：任何参数校验失败或业务执行异常，统一包装为结构化错误引导 LLM 自我修正，绝不让异常崩溃蔓延至主循环；
4. **并发批执行与因果时序保护**：支持声明式 `is_parallel_safe` 并发加速，并引入“一票否决”严格因果时序降级与保序回填机制。

```text
       普通 Python 函数 / 远程 MCP 工具
                     │
                     ▼
         @tool 包装 或 Tool(raw_schema=...)
                     │
                     ▼
             Tool 实体实例
      (to_openai_schema / execute)
                     │
                     ▼
               ToolRegistry
  (register / execute_tool / execute_batch / get_schemas)
```

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

> **实现方式：** 将特定参数名 `on_update` 过滤，按照 `def calculate(expr: str, precision: int = 2, on_update=None,) -> float: ...` 的例子来说判断 `param_name = "expr"` 和 `param_name = "on_update"` 是否在过滤参数中
> ```python
> _FRAMEWORK_RESERVED_PARAMS: frozenset[str] = frozenset({"on_update", "signal", "tool_call_id"})
> for param_name, param in inspect.signature(func).parameters.items():  
> 	if param_name in _FRAMEWORK_RESERVED_PARAMS:  
> 		continue
> ```

- 自动生成符合 OpenAI / Anthropic 规范的 Function Calling JSON Schema 字典；

> **实现方式：** 在 `pydantic.create_model()` 创建对象的时候可以自动返回
> ```python
> params_model = pydantic.create_model(
> 	"CalculateParams",
> 	expr=(str, Field(..., description="需要计算的算术表达式")),
>     precision=(int, Field(2, description="保留的小数位数")),
> )
> schema = params_model.model_json_schema()
> ```

- 运行时在调用真实函数前通过 `params_model.model_validate` 进行强类型校验与宽松类型转换（如 `"37"` 自动转换为 `37` ）。

> **实现方式：** 
> ```python
> params = params_model.model_validate({"expr": "1+2", "precision": "37"})
> ```


### 3. `Tool` 实体对象与提示词自声明 (Tool-as-a-Contributor)

- ** `prompt_snippet` (一句话动作概要)**：声明简短动作说明（如 `"Read file contents"` ），供系统提示词 `<tools>` 块使用。若缺省，框架自动从 `description` 中提取第一句作为紧凑概要，杜绝长文本注意力稀释与 Token 浪费；
- ** `prompt_guidelines` (工具级操作准则)**：声明该工具在调用时的排他性铁律与避坑指南（如 `edit` 声明最小唯一匹配、 `write` 声明禁止随意覆写），由 `format_rules_section` 统一聚合进 `<rules>` 块；

> **举例：** 假如有如下 `prompt = "在 config.py 里面加一行配置 {"path": "config.py", "content": "DEBUG = True\n"}"` ，其符合 schema，结果直接执行 `write` 操作所有配置都被覆盖，假如有 `prompt_guidelines = {禁止随意覆写}` ，这种情况能避免
> 实际上 `prompt_guidelines` 和 `schema` 一个是自然语言约束，一个是机器校验

- ** `raw_schema` 支持**：支持直接接收外部或远程传入的 JSON Schema 字典（如 MCP 远程工具），无需定义本地 Python 函数签名；

> **举例：** 收到服务器端的 `MCP_TOOLS` 中 `remote_search` 的 JSON Schema 后，本地通过 `tool_from_mcp` 组建 tool
> ```python
> import asyncio  
> from dataclasses import dataclass
> from typing import Callable, Optional
>
> @dataclass
> class Tool:
>     name: str
>     description: str
>     func: Callable
>     raw_schema: Optional[dict] = None   # 外部 JSON Schema（MCP 场景）
>
> MCP_TOOLS = [
>     {
>         "name": "remote_search",
>         "description": "远程搜索工具",
>         "inputSchema": {
>             "type": "object",
>             "properties": {"query": {"type": "string"}},
>             "required": ["query"],
>         },
>     }
> ]
>
> async def remote_call(query: str) -> str:
>     """执行侧：模拟 RPC/HTTP 调用 MCP 服务器。"""
>     print(f"  [RPC] 远程调用 remote_search(query={query!r})")
>     await asyncio.sleep(0.05)           # 模拟网络延迟
>     return f"远程搜索结果: {query} 的相关内容..."
>
> def tool_from_mcp(mcp_tool: dict) -> Tool:
>     """把 MCP 工具定义直接转成框架 Tool，不定义本地签名。"""
>     return Tool(
>         name=mcp_tool["name"],
>         description=mcp_tool["description"],
>         raw_schema=mcp_tool["inputSchema"],
>         func=remote_call,
>     )
>
> tool = tool_from_mcp(MCP_TOOLS[0])
> ```

- ** `timeout` 超时防护**：配置工具执行超时上限（秒），底层通过 `asyncio.wait_for` 拦截慢操作，超时自动转化为 `ToolResult(ok=False, error="Tool execution timed out after X seconds")` ；
- ** `is_parallel_safe` 标记**：声明式只读并发安全标记。当大模型单轮返回多个并发工具调用时， `ToolRegistry.execute_batch` 利用 `asyncio.gather` 并行执行，将多工具串行调用的 O (N) 耗时降为 O (1)。

### 4. `ToolResult` 、Never-Throw 保证与执行期输出定型截断

```python
DEFAULT_TOOL_MAX_BYTES = 50 * 1024  # 51,200 bytes (50KB)
DEFAULT_TOOL_MAX_LINES = 2000

def truncate_tool_output(
    content: str,
    max_bytes: int = DEFAULT_TOOL_MAX_BYTES,
    max_lines: int = DEFAULT_TOOL_MAX_LINES,
) -> str:
    """对齐 Pi 原厂 truncate.ts：在工具执行期对超长输出进行不可变截断（50KB 或 2000行）。

    保证工具输出进入会话消息历史时即已定型，永不随后续轮次发生就地篡改。
    """
    ...

@dataclass
class ToolResult:
    """工具执行结果：成功/失败 + 数据或错误消息 + 熔断提前退出标记 + 结构化元数据。"""
    ok: bool
    data: Any = None
    error: str | None = None
    terminate: bool = False
    meta: dict[str, Any] = field(default_factory=dict)

    def serialize(self) -> str:
        """转成写入 messages 的字符串。失败时返回错误文本，并在执行期完成定型截断。"""
        raw = str(self.data) if self.ok else (self.error or "Unknown error")
        return truncate_tool_output(raw)
```

- **Never-Throw 异常强隔离**：任何参数校验失败（`ValidationError`）、超时（`TimeoutError`）或业务执行异常，统一在 `Tool.execute()` 内部被捕获，转换为 `ToolResult(ok=False, error="...")`，由模型进行下一轮自我纠错；

> 服务于架构不变式 **Never-Throw Guarantee**

- **对齐 Pi `truncate.ts` 的执行期定型截断 (commit 8f01e4c)**：
    - **截断规约**：单条输出上限为 **50KB (51,200 bytes) 或 2000 行**。超限时先按行截断（保留前 2000 行），再按字节截断并在最近的换行符处切齐，末尾附带标准 notice：`\n[Output truncated: showing X of Y lines (A KB of B KB)]`；
    - **前缀缓存不变式（Prefix Cache Invariant）**：传统框架常在多轮滚动后去回溯修改历史工具消息（如滚动替换为占位符），但这会彻底破坏大模型的 KV-Cache 前缀。`my-pi-agent` 坚持**在工具执行期完成不可变定型截断**，输出一旦写入消息历史即成为永久确定性事实，确保全生命周期前缀一致性。

> **举例：** 
> **滚动修改：** 给定 `[System 8000] [User 100] [ToolOutput 3000] [Assistant 500]` ，当上下文空间不足，框架会将历史消息修改为 `[System 8000] [User 100] [ommitted 20] [Assistant 500]` ，导致 KV-Cache 命中率下降

## 三、 `ToolRegistry` 注册表调度与因果时序保护
### 1. 多态分发： `execute_tool`

`ToolRegistry.execute_tool` 支持灵活的入参形式，抹平外部协议差异：

1. **原生结构化传参**：`execute_tool(name="add", args={"a": 1})`
2. **元组传参**：`execute_tool(("add", {"a": 1}))`
3. **实体传参**：`execute_tool(ToolCall(id=..., name="add", args={"a": 1}))`
4. **Wire 字典传参**：`execute_tool({"name": "add", "args": {...}})` 或 `{"function": {"name": "...", "arguments": "..."}}`

如果 `args` 已为 Python 字典，直接跳过 `json.loads`；若为字符串则执行安全解析与异常转译。

>  **作者实现方式：** 简略版，一般就是 `tool_call = Tool()` 和 `name = "name", args = {}` 这种组合
>  ```python
>  async def execute_tool(
> 	 tool_call: Any = None,
> 	 name: str | None = None,
> 	 args: dict | str | None = None,
> 	 ...
>  ):
> 	 if name is None:
> 		 if isInstance(tool_call, tuple): if(): ...
> 		 elif isInstance(tool_call, dict): if(): ...
> 		 elif isInstance(tool_call, str): if(): ...
> 	if isInstance(args, str):
> 	elif isInstance(args, dict):
>  ```

### 2. 批执行调度与“一票否决”因果时序保护 ( `execute_batch` )

- **因果时序一致性（Causal Consistency）**： 当大模型在单轮中同时输出多个工具调用（例如 `ToolCall 1 (write: 修改文件)` 与 `ToolCall 2 (read: 确认文件内容)`），绝不能盲目把只读工具提前并发执行，否则会导致 `read` 先于 `write` 读到旧数据（产生严重的因果倒置错误）。
- **一票否决降级机制（Unanimous Parallel / Sequential Fallback）**：
    - **全员只读并发**：当且仅当批次中的**每一个**工具均为 `is_parallel_safe=True` 时，才放行 `asyncio.gather` 并行执行；
    - **含写严格串行**：只要批次中包含**任何一个**写操作（或未知工具），整批工具立即放弃并发，**严格按照大模型输出的原始先后顺序串行执行**，确保因果顺序绝对正确。

> **悲观策略：** 只要有一个 `write` 操作就串行（如果考虑任务之间的依赖关系会怎么样？）
> ```python
> has_sequential = any(   (t := self._tools.get(_get_name(tc))) is None or not t.is_parallel_safe for tc in tool_calls   )
> if has_sequential: 
> 	for tc in enumerate(tool_calls): 
> 		results.append(...)
> 	return results
> tasks = [asyncio.create_task() for i, tc in enumerate(tool_calls)]
> return list(await asyncio.gather(*tasks))
> ```

- **保序回填（Preserved Order Invariant）**： 执行完毕后，返回的 `ToolResult` 列表严格与入参 `tool_calls` 的索引位置完全对齐，确保生成的消息历史与模型调用上下文 1:1 保序匹配。

> `asyncio.gather (*tasks)` 天然就会返回一个聚合 `Future` ，再 `await` 之后就会返回 ` list(Future) ` 列表
> ```python
> async def task(name: str):  
> 	await asyncio.sleep(0.5)  
> 	print(f"task {name} done")
> async def main():  
> 	tasks = [asyncio.create_task(task(item)) for item in ["a", "b", "c", "d"]]   # 这里不能加await asyncio.create_task(), 否则就是串行
> 	results = await asyncio.gather(*tasks)  
> 	print(results)  
>  
> asyncio.run(main())
> ```

## 四、深度辨析：宏观串并行 vs 微观同异步（及异步队列的本质）

在工具系统的工程实现中，必须彻底厘清**“宏观批调度的串并行”**与**“微观函数的同异步”**这两个正交维度：

|维度|关心的核心问题|决定者|核心实现手段|
|:--|:--|:--|:--|
|**维度 1：宏观批处理 (并发 vs 串行)**|这批工具是一起开工，还是排队执行？| `ToolRegistry.execute_batch` (基于 `is_parallel_safe` 属性)|并发： `asyncio.gather(...)` ；串行： `for` 循环依次 `await` |
|**维度 2：单工具执行 (主线程 vs 子线程)**|这个工具自身会阻塞事件循环吗？| `Tool.execute` (基于 `inspect.iscoroutinefunction` )|异步（ `async def` ）：主线程协程直接跑；同步（普通 `def` ）：扔进系统线程池 `asyncio.to_thread` |

> **宏观与微观：** 
> **宏观**应该是说的主线程中的协程如何工作的，以下方代码为例，实际上执行时间是 2s
> ```python
> import asyncio
>
> async def task(name: str, time: int):
>     await asyncio.sleep(time)
>     print(f"{name} done")
>     return f"{name}-result"
> async def main():
>     tasks = [asyncio.create_task(task(item[0], item[1])) for item in [("a", 2), ("b", 1)]]
>     results = await asyncio.gather(*tasks)
>     return results
>
> res = asyncio.run(main())
> print(res)
> ```
> **微观**应该是说主线程中再调用多个子线程，我们看 `threading.Thread(target=fun)` 和 `asyncio.gather(asyncio.to_thread(fun))` 这两种做法，不过 `threading.Thread(target=fun)` 的方式无法获得返回值，并且 `join()` 会阻塞等待；而 `asyncio.to_thread(fun)` 是将函数交给线程池，可以获得返回值。此处可以去看 `03-agent-loop.md` 中的描述，具体如何实现的
> 
> **以下是 `threading.Thread(target=fun)` **
> ```python
> import threading  
> import time  
>   
> def task_(name: str, timeout: int):  
> 	time.sleep(timeout)  
> 	print(f"{name} done")  
> 	return f"{name}-result"  
> 
> threads = [threading.Thread(target=task_, kwargs={"name": name, "timeout": 2}) for name in ["a", "b", "c"]]  
> for thread in threads: thread.start()  
> for thread in threads: thread.join()
> ```
> **以下是 `asyncio.gather(asyncio.to_thread(fun))` **
> ```python
> import asyncio  
> import threading  
> import time  
 >  
> def task_(name: str, timeout: int):  
> 	time.sleep(timeout)  
> 	print(f"{name} done")  
> 	return f"{name}-result"  
>   
> async def main():  
> 	result = await asyncio.gather(  
> 		asyncio.to_thread(task_, "a", 2),  
> 		asyncio.to_thread(task_, "b", 2),  
> 		asyncio.to_thread(task_, "c", 2),  
> 	)  
> 	print(result)  
>   
> asyncio.run(main())
> ```

### 1. 异步队列里流动的到底是什么？

 **关键认知**：  
异步队列 `queue` 里面装的**不是工具执行的最终返回值（`ToolResult`）**！最终结果是在批处理 Task 结束时由 `batch_out = await runner` 一次性拉取的列表。  
**队列里流动的，纯粹是工具执行中途发射出来的“实时过程流式事件”（ `ToolExecutionUpdate` ）**。

> **解释：** 作者将工具执行分为两个通道：A 通道用于向 UI 传输中间结果即 `ToolExecutionUpdate` ，B 通道则是传输最终工具的执行结果放在消息 `Messages` 里面，如下流程图所示
> ```
> 工具执行
>     │
>     ├── 通道 A：过程事件（实时、可能很多条）
>     │      通过 queue → 前台 while 循环 → yield 给前端
>     │      内容：ToolExecutionUpdate，比如 {"pct": 50, "output": "..."}
>     │
>     └── 通道 B：最终结果（一次性、只有一份）
>            通过 runner 的返回值 → await runner → batch_out
>            内容：ToolResult 列表，比如 [ToolResult(ok=True, data="..."), ...]
> ```

### 2. 并行与串行在流水线中的具体运行行为
- **并行模式（全只读安全工具）**：
    - 宏观上 `execute_batch` 使用 `asyncio.gather` 同时打出所有工具；
    - 异步协程工具在主事件循环运行，同步阻塞工具被 `asyncio.to_thread` 发配到各个工作子线程运行；
    - 各工具中途产生进度时，主线程直接推入 `queue` ，子线程通过 `loop.call_soon_threadsafe` 跨线程安全预约推入 `queue` ；
    - 前台生成器实时 `yield` 谁最新产生的事件，呈现交织滚动的流式效果。

> 实际代码如下，简略版
> ```python
> 	tasks = [  
> 		asyncio.create_task(self.execute_tool(tc, signal=signal, on_update=_get_update_cb(i, tc)))  
> 		for i, tc in enumerate(tool_calls)  
> 		]
> 	return list(await asyncio.gather(*tasks))
> ```

- **串行模式（含任一写操作/非安全工具）**：
    - 宏观上 `execute_batch` 退化为顺序 `for` 循环；
    - 工具 1 执行中产生进度流入 `queue` → 前台实时打印 → 工具 1 完成；
    - 工具 2 执行中产生进度流入同一个 `queue` → 前台接着实时打印 → 工具 2 完成；
    - 整批完成后由 `finally` 注入 `_SENTINEL` 哨兵，队列消费无缝复用，零冗余代码。

> 实际代码如下，简略版
> ```python
> if has_sequential:
> 	for i, tc in enumerate(tool_calls):
> 		results.append(await self.execute_tool(tc, signal=signal, on_update=_get_update_cb(i, tc)))
> ```

## 五、实战指南：什么样的工具写成 async def？什么样的工具写成普通 def？

核心判断准则：**工具执行时主要是在“等外部网络/事件”，还是在“消耗本地 CPU/阻塞型操作系统资源”？**

1. **必须/推荐写成【异步工具】（ `async def` ）**：
    - **子代理委派（Subagent / Task）**：例如 `task` 、 `delegate_subagent` 。等待子 Agent 的整个 LLM 交互与工具循环，必须通过 `await` 挂起；
    - **网络请求与外部服务（Web Search / Crawl / MCP）**：基于 `httpx.AsyncClient` 或 `aiohttp` ，等待远程握手与数据传输；
    - **浏览器自动化（Playwright）**：等待 DOM 渲染、页面导航，天然全是非阻塞异步 Promise。
2. **通常写成【同步工具】（普通 `def` ）**：
    - **本地小文件读写与外科手术式编辑（ `read` / `write` / `edit` ）**：本地 SSD 耗时通常小于 1 毫秒，用普通的 `pathlib` 或 `open()` 简洁高效，无额外调度开销；
    - **CPU 密集运算与正则匹配（ `grep` / `calculator` / `ast-grep` ）**：纯计算无空闲等待，无需 `async` ；
    - **封装阻塞式系统库的工具（ `bash` ）**：调用原生的 `subprocess.run(...)` 。

> ** 判断：**  `等` -> `async def` ； `算` -> `def`

3. **框架的自适应无感桥接（Zero Mental Overhead）**： `Tool.execute` 内部根据 `self.is_async` 自动路由： `async def` 直接协程 `await` ，普通 `def` 自动包裹 `asyncio.to_thread` 。开发者想怎么写就怎么写，既不卡死 Agent 主事件循环，又拥有极致执行性能。

> 实际代码如下，简略版
> ```python
> if self.is_async:
> 	result = await async_func_call()
> else:
> 	result = await asyncio.to_thread(func_call)
> ```

