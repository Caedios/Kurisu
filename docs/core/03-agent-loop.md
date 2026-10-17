# Agent 设计规范 
- 定位
- 核心组件
- 关键 API
## 一、


## 二、微内核子生成器分治机制
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
> 	 - 引入 Jitter 防抖动——若所有客户端都固定 `2000ms` 后重试，会造成重试风暴、引发服务端雪崩；若服务端通过 `Retry-After` 声明了可再次访问的时间，则优先服从服务端响应。

- **取消与异常 Never-Throw**： 若检测到 `CancellationToken.is_cancelled()` ，优雅中断并标记 `stop_reason="cancelled"` ；模型报错时封装为错误消息，保证上层轮次能安全闭环。

>  **解决问题：** 当模型取消或异常该如何处理使得程序不崩溃？
>  将一切错误封装为 Message，服务于架构不变式 **Never-Throw Guarantee**

### 2 专职工具执行车间： `_execute_tools_turn`

1. **阶段 1：输出截断防御检查 ( `_fail_tool_calls_from_truncated_message` )**： 当检测到模型输出触达 Token 上限被截断（ `stop_reason == "length"` ）时，断然拒绝执行任何工具，防止流式 salvage 拼出残缺参数导致代码写崩或命令腰斩。自动合成警告错误并回传模型引导重新完整发起调用。

> **防坑：** 模型输出被 Token 上限截断，此时工具参数**残缺**。
> **场景：** 假如此时执行 `write_file(path="main.py", content="...很长的内容...")` ，salvage 拼接出残缺调用会导致严重后果：
>
> - `write_file` 写入半截文件，原文件损坏；
> - `bash` 执行了未闭合的参数，删库跑路？
>
> **怎么解决：** 检测到 `stop_reason == "length"` 后，拒绝执行工具。
> **为什么是阶段 1：** 这是工具执行前就能检查的事项；若拖到下一阶段 `ToolExecutionStart` ，为时已晚。

2. **阶段 2：Preflight 广播 ( `ToolExecutionStart` )**： 在审批与执行前，**率先按 source order 广播 `ToolExecutionStart` **，使 UI 能够毫秒级渲染工具准备运行状态。

> **防坑：** 当用户已经调用工具时，前端 UI 等待好久才会显示正在调用工具
> **怎么解决：** 按照模型声明的工具执行顺序 source order 广播 `ToolExecutionStart` ，告诉前端直接渲染
> **为什么是阶段 2：** 从用户体验上，用户肯定希望一调用工具就能得到正反馈，符合人性

3. **阶段 3：串并行决策网关 ( `ToolRegistry.execute_batch` )**： 悲观读写分流：全只读安全工具（ `is_parallel_safe=True` ）启用 `asyncio.gather` 全并发加速；只要包含任一写入/串行工具，整批退化为保序串行执行，防止因果时序倒置。



4. **阶段 4：前置审查审批与改参 ( `before_tool_call` )**： 通过 `_coerce_tool_call` 归一化入参，调用 `before_tool_call` 审批，支持安全阻断（ `block` ）与参数就地热修改（ `updated_args` ）。



5. **阶段 5：并发批执行与流式进度回传 ( `ToolExecutionUpdate` )**： 采用 `asyncio.Queue` 与 `loop.call_soon_threadsafe` 跨线程安全桥接，支持长耗时工具（如 Bash 编译或子代理）在运行态向外广播累积快照（Cumulative Snapshot）。生命周期锁存（ `accepting_updates` ）确保工具返回后丢弃迟到回调。



6. **阶段 6：后置改写与单工具终态广播 ( `after_tool_call` & `ToolExecutionEnd` )**： 调用 `after_tool_call` 支持结果脱敏与改写（ `updated_result` ），广播包含 `terminate` 状态的 `ToolExecutionEnd` 。



7. **阶段 7：转录本保序归档与批量优雅熔断 ( `MessageStart/End` & `should_terminate` )**： 无论并发执行完成顺序如何，回传大模型的 `role="tool"` 消息严格按 Assistant 原始 Source Order 恢复排布。若批次中任一工具（ `any()` 语义）或 Hook 返回 `terminate=True` ，立即终结 ReAct 循环，保全 `final_text` 并正常交付结果。