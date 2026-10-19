# Agent 设计规范 
- **定位**：Agent 执行中枢、ReAct 微内核状态机与转录本自愈引擎 ( `src/my_agent_core/agent.py` , `loop.py` , `tool_history.py` )
- **核心组件**：
    - `run_agent_loop`：无状态纯函数异步生成器微内核（约 110 行优雅状态机）
    - `_assistant_turn` & `_execute_tools_turn`：专职子生成器车间
    - `_provider_context` & `repair_tool_history`：上下文前置清洗与转录本自愈引擎
    - `Agent`：轻量有状态外壳（`AgentHarness`），对外暴露 `prompt_stream` 一等公民事件流与 `subscribe` 观察者接口
- **关键 API**：`agent.prompt_stream(user_input)`, `agent.run(user_input)`, `agent.subscribe(listener)`, `agent.steer(msg)`, `agent.follow_up(msg)`, `agent.abort()`

## 一、架构设计

```text
用户调用 agent.prompt_stream("...")
        │
        ▼
┌─ Agent（外壳）──────────────────────────┐
│  组装好组件，启动微内核                 │
└────────────────┬────────────────────────┘
                 │ 消费事件流
                 ▼
┌─ run_agent_loop（微内核）───────────────┐
│  while True:                            │
│    while 需要继续:                      │
│      yield TurnStart                    │
│      async for ev in _assistant_turn    │──▶ 车间①：调模型，流式吐事件
│      async for ev in _execute_tools_turn│──▶ 车间②：跑工具，吐事件
│      yield TurnEnd                      │
│  yield AgentEnd                         │
└─────────────────────────────────────────┘
                 │
                 ▼
        事件流返回给调用方
   （TUI 逐字渲染 / run() 提取 final_text）
```

## 二、微内核
### 1  专职模型推理车间： `_assistant_turn`
- **流式增量与状态定型**： 首个 Chunk 到达时发射 `MessageStart(assistant)` ；在流式生成中逐字发射 `MessageUpdate` 驱动终端打字机；流式结束时直接接收模型边界层交付的完整 `Response` 实体，发射 `MessageEnd` 定型。

>  **解决问题：** 如何把散装的 Chunk 聚合成一个完整的 `Response` 实体？
>  - 反直觉 1： 为什么不在模型调用的时候开始发送 Chunk？
> 	 - 模型调用本身可能失败（ `401` Key 无效或过期； `403` 越权 / 模型不可用），因此不会产生任何内容
>  - 反直觉 2：为什么需要逐 Chunk 发送信息？
> 	 - 如果 Response 的信息太长，导致一次发送大段信息，体验不好，主流模型都提供流式输出（SSE）
>  - 反直觉 3：为什么不直接把收到的 Chunk 拼接为 `Response` （省点内存？），反而还是使用完整的 `Response` 实体？
> 	 - 完整 Response 实体中具有 `content` 、 `token_usage` 字段，拼接可能导致错误（未考证：聚合错误）
> 
>  MessageEnd 定型服务于架构不变式 **Prefix Cache Invariant** 

```text
> 时间轴 ──────────────────────────────────────────────▶
>
> 模型调用开始
>     │
>     │  （等待首个 chunk，可能重试，见下半部分）
>     │
>     ▼
> 首个 chunk ──▶ MessageStart(assistant)                                            ← 告诉前端：模型开始说话了，创建一条 assistant 消息
>     │
>     ├── chunk 2 ──▶ MessageUpdate                                               ← 流式期间每收到一个片段就发一次
>     ├── chunk 3 ──▶ MessageUpdate                                                   前端据此逐字渲染（打字机效果）
>     ├── ...
>     │
> 流式结束 ──▶ 模型边界层交付完整 Response
>     │
>     ▼
> MessageEnd(full_response)   ← 定型，写入历史，不可变
```

- **智能退避重试体系 ( `AutoRetryPolicy` )**： 对标 Pi 原厂自动重试规范，在微内核层拦截大模型调用异常：
    - **瞬时可恢复错误（自动重试）**：429（频率超限）、500/502/503/504（服务端偶发故障）、`httpx.ConnectError` / `ConnectTimeout` / `ReadTimeout` / `RemoteProtocolError`（网络传输断流与连接重置）；
    - **致命不可恢复错误（立即熔断，绝不重试）**：400（参数畸形）、401（Key 无效或过期）、403（越权/模型不可用）、404、以及账号欠费配额超限（`insufficient_quota` / `out of budget`）；
    - **退避算法**：基于 `base_delay_ms`（默认 2000ms）的指数退避（$2^{attempt-1}$），引入 Jitter 防雪崩抖动，并优先尊重服务端回传的 `Retry-After` / `retry-after-ms` 响应头；
    - **可取消退避**：重试睡眠期间支持 `CancellationToken` 毫秒级中断（按 `Esc` 立即停止重试并安全退出）。


>  **解决问题：** 当模型调用异常时，该重试还是熔断？重试几次？每次等多久？
>  - 判断 1： 如何判断异常的方式是重试还是熔断
> 	 - 根据错误码分类：瞬时可恢复错误进入重试；致命不可恢复错误立即熔断。
>  - 判断 2：如何执行重试算法？
> 	 - 引入 Jitter 防抖动——若所有客户端都固定 `2000ms` 后重试，会造成重试风暴、引发服务端雪崩；若服务端通过 `Retry-After` 声明了可再次访问的时间，则优先服从服务端响应

- **取消与异常 Never-Throw**： 若检测到 `CancellationToken.is_cancelled()` ，优雅中断并标记 `stop_reason="cancelled"` ；模型报错时封装为错误消息，保证上层轮次能安全闭环。

>  **解决问题：** 当模型取消或异常该如何处理使得程序不崩溃？
>  
>  - 将一切错误封装为 Message，服务于架构不变式 **Never-Throw Guarantee**

### 2 专职工具执行车间： `_execute_tools_turn`

1. **阶段 1：输出截断防御检查 ( `_fail_tool_calls_from_truncated_message` )**： 当检测到模型输出触达 Token 上限被截断（ `stop_reason == "length"` ）时，断然拒绝执行任何工具，防止流式 salvage 拼出残缺参数导致代码写崩或命令腰斩。自动合成警告错误并回传模型引导重新完整发起调用

> **防坑：** 模型输出被 Token 上限截断，此时工具参数**残缺**
> 
> **场景：** 假如此时执行 `write_file(path="main.py", content="...很长的内容...")` ，salvage 拼接出残缺调用会导致严重后果：
>
> - `write_file` 写入半截文件，原文件损坏；
> - `bash` 执行了未闭合的参数，删库跑路？
>
> **怎么解决：** 检测到 `stop_reason == "length"` 后，拒绝执行工具
> 
> **为什么是阶段 1：** 这是工具执行前就能检查的事项；若拖到下一阶段 `ToolExecutionStart` ，为时已晚

2. **阶段 2：Preflight 广播 ( `ToolExecutionStart` )**： 在审批与执行前，**率先按 source order 广播 `ToolExecutionStart` **，使 UI 能够毫秒级渲染工具准备运行状态

> **防坑：** 当用户已经调用工具时，前端 UI 等待好久才会显示正在调用工具
> 
> **怎么解决：** 按照模型声明的工具执行顺序 source order 广播 `ToolExecutionStart` ，告诉前端直接渲染
> 
> **为什么是阶段 2：** 从用户体验上，用户肯定希望一调用工具就能得到正反馈，符合人性

3. **阶段 3：串并行决策网关 ( `ToolRegistry.execute_batch` )**： 悲观读写分流：全只读安全工具（ `is_parallel_safe=True` ）启用 `asyncio.gather` 全并发加速；只要包含任一写入/串行工具，整批退化为保序串行执行，防止因果时序倒置。

> **防坑：** 防止因果倒置
> 
> **场景：** 假如此时队列中 `write_file()` 和 `bash()` ，很可能先 `bash()`  了一个不存在的文件，因为没有 `write_file()`：
>
> **怎么解决：** 采用悲观读写方式，如果都是读操作（ `read_file()` 、 `find()`  ）就并发执行，有一个写操作（ `write_file()` ）就串行
> 
> **为什么是阶段 3：** 决定执行顺序应该在工具执行之前，但是阶段 3 和阶段 4 目前不清楚为什么会这么分配

4. **阶段 4：前置审查审批与改参 ( `before_tool_call` )**： 通过 `_coerce_tool_call` 归一化入参，调用 `before_tool_call` 审批，支持安全阻断（ `block` ）与参数就地热修改（ `updated_args` ）。

> **防坑：** 模型生成的参数不合规，或者需要人工审批
> 
> **场景：** 假如 Agent  生成的参数是 JSON 格式的，而工具需要 dict ；假如 `path="/etc"` 或者 `command="rm -r -f"` 越权访问
>
> **怎么解决：** 在工具执行前通过 `_coerce_tool_call` 修改不合规参数，调用 `before_tool_call` 进行安全阻断或者参数修改
> 
> **为什么是阶段 4：** 修改参数应该在工具执行之前，但是阶段 3 和阶段 4 目前不清楚为什么会这么分配

```text
如果在阶段 4 模型拒绝了一部分工具执行，那阶段 3 的一部分工具就没有用了
猜测难道设计思路是审批改参是单工具粒度而不是批量的，这样好像能解释通。确实，因为阶段 3 是批量然后阶段 7 是批量处理，中间都是单工具粒度的操作，这样没问题
```

5. **阶段 5：并发批执行与流式进度回传 ( `ToolExecutionUpdate` )**： 采用 `asyncio.Queue` 与 `loop.call_soon_threadsafe` 跨线程安全桥接，支持长耗时工具（如 Bash 编译或子代理）在运行态向外广播累积快照（Cumulative Snapshot）。生命周期锁存（ `accepting_updates` ）确保工具返回后丢弃迟到回调。

> **防坑：** 执行长时间的任务时，用户不清楚后台究竟发生了什么
> 
> **场景：** 假如 Agent  执行了一个长时间的  `bash()` 任务，前端一片空白
>
> **怎么解决：** 前后台解耦的单向传送带，具体见后面
> - 前端：只负责一个异步队列 `asyncio.Queue` ，一直 `await` 如果有 `yield` 就马上返回给外部界面
> - 桥梁：负责将多线程执行结果通过  `loop.call_soon_threadsafe` 发送给 `asyncio.Queue`
> - 后端：只负责多线程 `asyncio.gather` 执行工具，通过  `on_update` 回调产生进度
> 
> 什么是累积快照：举例来说，假如发送队列 `[0, 1, 2, 3, 4]` ，增量快照是 `[0]` ， `[1]` ， `[2]` 逐个发送，累积快照是 `[0]` ， `[0, 1]` 逐个发送
> 
> 什么是声明周期锁存：在当前生命周期内，后续还有快照发来就接受，但是一结束当前生命周期（ `accepting_update` 标志设为 `False` ），后续的快照必须拒绝
> 
> **为什么是阶段 5：** 这是真正在执行工具，确实应该处于 `before_tool_call` 和  `after_tool_call` 之间

6. **阶段 6：后置改写与单工具终态广播 ( `after_tool_call` & `ToolExecutionEnd` )**： 调用 `after_tool_call` 支持结果脱敏与改写（ `updated_result` ），广播包含 `terminate` 状态的 `ToolExecutionEnd` 。

> **防坑：** 执行任务结束后，工具执行结果不合规，或者前台还认为任务在执行中
>
> **怎么解决：** `after_tool_call` 作为 `hook` 对返回的结果重新 `updated_result` ；同时修改 `terminate=True` 告诉前台任务执行结束，可以熔断
> 
> **为什么是阶段 6：** 这是在执行之后，确实应该是  `after_tool_call` ，由于设计问题，阶段 6 是单粒度的工具执行，所以要在整批粒度的阶段 7  处理前面

7. **阶段 7：转录本保序归档与批量优雅熔断 ( `MessageStart/End` & `should_terminate` )**： 无论并发执行完成顺序如何，回传大模型的 `role="tool"` 消息严格按 Assistant 原始 Source Order 恢复排布。若批次中任一工具（ `any()` 语义）或 Hook 返回 `terminate=True` ，立即终结 ReAct 循环，保全 `final_text` 并正常交付结果。

> **防坑：** 并发工具完成顺序 $\ne$ 模型声明顺序， `tool_id` 不一致会导致 API 400 错误
> 
> **场景：** 模型声明 `[tool_a, tool_b, tool_c]` ，结果完成顺序是 `[tool_b, tool_c, tool_a]` ，模型发现第一个返回的工具结果  `result_b` 的 `tool_id` 与 `tool_a` 的  `tool_id` 不一致，引发 API 400 错误
>
> **怎么解决：** 根据 Source Order 将 `role="tool"` 的结果按照顺序放入 Message 
> 
> **批量熔断：**
> - 为什么用 `any()` 而不是 `all()` ：假如模型同时调用了 ` [write_file, is_completed] ` ，结果 ` is_completed ` 返回已完成，所以一个 ` terminate=True ` 就应该结束；
> - 为什么要保留 ` final_text ` ：不清楚这个场景，难道是可能会有 `final_text="completed"` 这种信息显示地返回前台告诉完成了
> 
> **为什么是阶段 7：** 多个工具执行结束，应该要对多个工具的执行结果统一交付给前台

### 3 阶段  5 的技术难点和解决方法
在实现第 5 阶段（并发执行与实时流式进度回传）时，微内核面临了三个看似不可调和的技术冲突：
```
【冲突 1】异步生成器 vs 并发批量执行
  • 需求 A：多个工具必须并发跑（比如同时读 3 个文件），底层用的是 asyncio.gather(...)；
  • 需求 B：_execute_tools_turn 是一个生成器，必须一有进度就立刻 yield 出去。
  • 痛点：在 asyncio.gather 的深处是不能直接向外层生成器 yield 的！

【冲突 2】主事件循环 vs 工作线程池
  • 需求 A：很多工具是同步的（如普通的 Python 函数、调用操作系统的 bash），必须扔进线程池
           asyncio.to_thread 跑，不能卡死主线程；
  • 痛点：Python 的异步队列 asyncio.Queue 是严格单线程的，工作线程直接碰队列就会崩溃闪退！

【冲突 3】实时流式读取 vs 任务结束防死锁
  • 需求 A：前台必须一直监听队列，只要有日志就拿出来；
  • 痛点：前台怎么知道后台什么时候“全部跑完了”？如果盲目等，后台跑完后前台就会永久卡死（死锁）。
```

1. **冲突 1 解释： `gather` 要协程， `yield` 只能生成器**
目前想实现的功能伪代码如下：
```python
import asyncio
async def run_tool(name: str):
    print(name)
    asyncio.sleep(1)
    yield print("完成进度 50%")
    asyncio.sleep(1)
    print(name)
async def main():
    await asyncio.gather(run_tool("a"), run_tool("b"))
asyncio.run(main())
```
   
   矛盾的根源：** `asyncio.gather()` 只接受协程**，而 ** `yield` 让函数变成异步生成器**，两者不兼容。
   **作者的解决方式**：把「并发执行」和「流式输出」拆到两层——
   
   ```python
   async def run_tool(name: str, queue: asyncio.Queue):
       print(name + " start")
       for progress in [25, 50, 75]:
           await asyncio.sleep(1)
           await queue.put({"tool": name, "progress": progress})
       return f"{name} done"
       
   async def _execute_tools_turn():
       queue = asyncio.Queue()
       async def worker():
           result = await asyncio.gather(
               run_tool("tool_a", queue),
               run_tool("tool_b", queue),
               run_tool("tool_c", queue),
           )
           await queue.put({"complete": True, "result": result})
       task = asyncio.create_task(worker())
       while True:
           item = await queue.get()
           if item.get("complete"):
               await task
               yield item["result"]
               break
           yield item
           
   async def main():
       async for update in _execute_tools_turn():
           print(f"front receive {update}")
           
           
   asyncio.run(main())
   ```

2. **冲突 2 解释： `asyncio.Queue` 是单线程，工具池要求多线程**
目前想实现的功能伪代码如下（当前代码没有报错，不清楚什么原因）：
```python
async def main():
    queue = asyncio.Queue()
    errors = []

    def worker():
        try:
            queue.put({"from": threading.current_thread()})
        except Exception as e:
            errors.append(e)
            print("catch exception", e)

    threads = [threading.Thread(target=worker) for i in range(10)]
    for t in threads: t.start()
    for t in threads: t.join()

    print(errors)
    print("queue.qsize() =", queue.qsize())

asyncio.run(main())
```
矛盾的根源： `asyncio.Queue` 是单线程的不能线程阻塞，需要再本线程内拿到 `Future` 凭证才会执行下一步
- 举例说明 `Queue` 和 `asyncio.Queue` 的区别
	- 多员工场景，当前单位有多位员工， `Queue` 中存储员工要干的工作，在做某项工作时缺少批准，就休息（阻塞），等批准再干
	- 单员工场景，当前单位只有一位员工， `asyncio.Queue` 中存储员工要干的工作，在做某项工作时缺少该项目负责人签名，就要了 `Future` 凭证，等项目负责人通知再接着完成，等待过程中不能休息，需要接着做其他工作

 **作者的解决方式**：工具都在线程层，编排和消费在协程层，中间用 loop 事件（ `call_soon_threadsafe`  和 `Future` 沟通）
 - 线程层的 `sync_tool` 完成工作将上报的 `quque.put_nowait` 和 `item` 登记（ `call_soon_threadsafe` ）在协程层的 `run_tool` 里面
 ```python
import asyncio
 import threading
 def sync_tool(name: str, update):
     for pct in [25, 50, 75]:
         update({"name": name, "pct": pct, "thread": threading.current_thread().name})
     return name + " done"
 async def run_tool(name, queue, loop):
     def update(item):
         loop.call_soon_threadsafe(queue.put_nowait, item)
     return await asyncio.to_thread(sync_tool, name, update)
 async def _execute_tools_turn():
     queue = asyncio.Queue()
     loop = asyncio.get_running_loop()
     async def worker():
         result = await asyncio.gather(
             run_tool("tool_a", queue, loop),
             run_tool("tool_b", queue, loop),
         )
         await queue.put({"done": True, "result": result})
     asyncio.create_task(worker())
     while True:
         item = await queue.get()
         if item.get("done"):
             yield item["result"]
             break
         yield item
 async def main():
     async for item in _execute_tools_turn():
         print("前端收到：", item)
 asyncio.run(main())
 ```

3. **冲突 3 解释： 防止协程一直等待队列任务** 
目前想实现的伪代码如下：
```python
async def _execute_tools_turn():
	task = async.create_task(run_tools(queue))
	while True:
		item = await queue.get()
		yield item
```

   矛盾的根源：**前台一直需要一直监听 `queue` ，如果监听不到就会一直卡在 `await queue.get()` **。
   **作者的解决方式**：设置哨兵 **sentinel**
```python
async def _execute_tools_turn():
    queue = asyncio.Queue()
    loop = asyncio.get_running_loop()
    async def worker():
        result = await asyncio.gather(
            run_tool("tool_a", queue, loop),
            run_tool("tool_b", queue, loop),
        )
        await queue.put({"done": True, "result": result})
    asyncio.create_task(worker())
    while True:
        item = await queue.get()
        if item.get("done"):                 #设置哨兵 sentinel, 如果item里面发现有完成, 就跳出循环
            yield item["result"]
            break
        yield item
```

## 三、上下文清洗与转录本自愈
大模型提供商（如 OpenAI、Anthropic）对上下文格式有着极其严苛的校验规则：

- **禁止悬空断头调用**：Assistant 发起了 `tool_calls` ，后面必须紧随对应 ID 的 `tool` 消息，否则直接报 API 400；
- **禁止孤儿结果**：没有对应 `tool_calls` 的 `role="tool"` 消息会被拒绝；
- **禁止空失败轮次**：以 `stop_reason="error"` 结尾且 `content=""` 的中断消息会导致上下文语法错误。

1. `_provider_context` 前置清洗
在每次向大模型发送消息列表前，微内核自动执行 `_provider_context(messages)` ：
	1. 剔除无正文且以异常中断结尾的终端 assistant 失败轮次；
	2. 串联 `repair_tool_history` 进行转录本拓扑自愈。

> 流程是 `message` -> `_provider_context()` -> x -> `repair_tool_history()` -> repaired `message`
> 删除的典型消息 `{"role": "assistant", "content": "", "stop_reason": "error"}`

2. `repair_tool_history` 三阶段确定性状态机

```text
原始乱序/断头消息
       │
       ▼ Phase 1 (建立索引): 收集所有 Assistant 节点的 tool_calls 声明
       ▼ Phase 2 (孤儿与重复清洗): 从后往前扫描，丢弃无主结果与重复结果
       ▼ Phase 3 (重排与断头补齐): 严格按 Assistant 声明顺序重排 Tool 消息，
                                  对缺失结果的断头调用自动合成
                                  role="tool", content="Tool call interrupted by user"
       │
       ▼
格式 100% 严谨合法的消息历史 (免疫任何 LLM API 400 校验死锁)
```

> 

## 四、Agent Harness
`Agent` 类蜕变为轻量外壳，负责生命周期资源的组装与管理：

1. 一等公民事件流： `prompt_stream`

调用方可以直接以异步迭代器的方式消费微内核发射的每一个纯只读事件：

```python
agent = Agent(llm=llm, tools=[...])

async for event in agent.prompt_stream("帮我重构这个模块"):
    if isinstance(event, MessageUpdate):
        print(event.chunk.content, end="", flush=True)
    elif isinstance(event, ToolExecutionStart):
        print(f"\n[Tool Start] {event.tool_name} with {event.args}")
    elif isinstance(event, AgentEnd):
        print(f"\n[Finished] Total iterations: {event.iterations}")
```

> `MessageUpdate` 对应 `_assistant_turn` 中实现流式输出 Chunk
> `ToolExecutionStart` 对应 `_execute_tools_turn` 中执行工具 Tool

2. 观察者模式： `subscribe` 与 `_notify` 广播管道

对标 Pi 的 `AgentSession.subscribe` 与 Tau 的 `AgentHarness.subscribe`，`Agent` 实现了轻量只读观察者管道：

```python
# 注册监听器（返回注销句柄）
unsubscribe = agent.subscribe(lambda event: print(f"Audit log: {type(event).__name__}"))

# 业务注销（零内存泄露隐患）
unsubscribe()
```

> 类比理解为数据库连接 `conn = sqlite3.connect("db.sqlite")` ，然后通过句柄 `conn` 注销 `conn.close()`

核心实现不变式（ `Agent._notify` ）

1. **快照遍历（Snapshot Iteration）**：使用 `for sub in list(self._subscribers)` 遍历快照，防止回调函数在执行中反注册导致的遍历变异竞态；
> **防坑：** 实际上快照 snapshot 指的是 `list()` ，因为如果直接调 `self._subscribers` ，在遍历过程中列表 `self._subscribers` 发生改变，运行时可能出现 `RuntimeError: list changed size during iteration` 错误

2. **异常隔离（Never-Throw Guarantee）**：使用 `with contextlib.suppress(Exception):` 隔离监听器异常，旁路订阅者的任何错误绝对不会中断主循环；
> **防坑：** 如果监听器内部出现异常 （写入 `0/1` ），那么 `_notify` -> Agent 整段跨掉，而 `suppress(Exception)` 就是为了拒绝监听器的异常，这段设计也符合整个项目中的架构不变式**Never-Throw Guarantee**

3. **同异步自适应（Adaptive Dispatch）**： `if inspect.isawaitable(res): await res` ，无论监听器写成同步 `def` 还是异步 `async def` 均能无缝等待；
> **防坑：** 有的监听器被写成 `def sync_fun()` ，有的被写成 `async def async_fun()` ，同时有同步函数和异步函数，统一 `await` 或统一不 `await` 会报错， `await sync_fun()` 抛出 `object NoneType can't be used in 'await' expression` ， `async_fun()` 抛出 `RuntimeWarning: Enable tracemalloc to get the object allocation traceback` ，具体我们可以看以下例子：

```python
import asyncio
import inspect

def sync_fun(event):
    print(f"[sync] received {event}")
async def async_fun(event):
    await asyncio.sleep(0.5)
    print(f"[async] received {event}")

async def f_all_await(subs=[sync_fun, async_fun], event="?"):
    for sub in subs:
        await sub(event)
async def f_no_await(subs=[sync_fun, async_fun], event="?"):
    for sub in subs:
        sub(event)
async def f_adaptive_await(subs=[sync_fun, async_fun], event="?"):
    for sub in subs:
        res = sub(event)
        if inspect.isawaitable(res):
            await res

async def main():
    try: await f_all_await()
    except Exception as e: print(e)
    await asyncio.sleep(0.5)

    print("=" * 30)
    await f_no_await()
    await asyncio.sleep(0.5)

    print("=" * 30)
    await f_adaptive_await()

asyncio.run(main())
```

4. **注销令牌（Teardown Token）**： `subscribe` 返回闭包句柄，通过对象引用精确定位移除，消除索引耦合；
> **防坑：** 如果返回索引，假如 3 号订阅者注销，下一个订阅者是 3 号还是 4 号呢？不过这个解释真的有力吗？这个防坑可能从句柄的设计意义上来说，还是说当前项目只涉及了索引吗？

5. **全触点覆盖**：在 `UserInputHook` 阻断退出、 `AgentStartHook` 阻断退出、 `prompt_stream` 主事件循环、以及上下文压缩写回 4 大触点统一触发广播。
> **防坑：** 如果只在 `prompt_stream` 主事件循环中广播，那么在其他阶段没有办法退出了，得保证监听器在任意阶段都可以收到结束信号


3. 便利门面： `run`
经典阻塞式调用，内部纯粹消费 `prompt_stream` 并在 `AgentEnd` 时提取 `final_text` 返回：

```python
async def run(self, user_input: str) -> str | None:
    final_text = None
    async for event in self.prompt_stream(user_input):
        if isinstance(event, AgentEnd):
            final_text = event.final_text
    return final_text
```

> 这里不清楚为什么单独提出 `run` ，难道不是在 `一等公民事件流` 中已经介绍过了吗？

4. 协作式中断： `abort`
通过 `agent.abort()` 触发内部 `CancellationToken.cancel()` ，流式推理与工具执行车间将在下一个检查点安全终止，并完成断头结果自动合成与原子落盘，**绝不向 Session 写入半截脏数据**。

> 设计目的：使得系统能干净、安全地停下来，不泄露资源或者操作执行一半
> **中断流程：** `agent.abort()` 调用 `CancellationToken.cancel()` 协作式地取消进程，实际上是通知各个环节释放结束信号
> - 专职模型推理车间 `_assistant_turn` ：
> 	- **智能退避重试体系 ( `AutoRetryPolicy` )** 中设计了 `CancellationToken.is_cancelled()` 检查
> 	- 应该各个环节都涉及了，比如调用模型后检查、Chunk 到达后检查，文档没有描述应该在代码部分
> - 专职工具执行车间 `_execute_tools_turn` ：
> 	- **断头结果自动合成** 中假如模型正在执行工具 `[tool_a, tool_b, tool_c]` ，当前执行情况如下所示 `tool_a - completed` ， `tool_b - working` ， `tool_c - pending` 。如果直接中断应该为 `tool_b` 和 `tool_c` 补全工具结果，因为要符合消息合法性，添加诸如 `Tool_Message(content = "Tool is cancelled by user", tool_id = "tool_b")` 形式的信息。
> 	- **原子落盘** 中假如模型正在执行工具 `write_file` 而且执行了一半，要么全写要么全不写，其实现方式是 `Path(f"{tmp}.txt").write_text("...")` -> `fsync(f"{tmp}.txt")` ->  `os.replace(f"{tmp}.txt", f"{target}.txt")`
> 	
> 以下是 `abort` 的完整流程

```text
用户按 Esc
    │
    ▼
agent.abort()
    │
    ▼
CancellationToken.cancel()   ← 设置取消标志
    │
    ▼
当前操作走到下一个检查点
    │
    ├─ 流式推理：停止接收 chunk，发射 MessageEnd（标记 cancelled）
    │
    └─ 工具执行：停止后续工具，补齐断头结果
    │
    ▼
收尾三件事：
    1. 合成缺失的 ToolMessage
    2. 原子落盘会话
    3. 校验历史合法性
    │
    ▼
安全退出，历史可继续使用
```