# asyncio 实例：三个工具如何在并发上限 2 下完成执行

[返回专题目录](./README.md) · [先读总体机制](./01-event-loop-task-future.md) · [打开完整图文版](./assets/asyncio-tool-batch-walkthrough.html)

## 1. 这个测试要证明什么？

### 完整测试函数

下面代码与当前 `tests/test_runner.py` 中的测试函数一致。`ScriptedFakeModel`、`ToolRequest`、`ToolSpec` 等定义或导入位于该测试文件前部。

```python
def test_run_agent_bounds_parallel_tool_batch_and_preserves_result_order() -> None:
    """滚动池不得超过 Run 上限，回填顺序仍应等于模型请求顺序。"""

    async def scenario() -> None:
        requests = tuple(
            ToolRequest(
                call_id=f"call-{index}",
                name="parallel_tool",
                arguments={"index": index},
            )
            for index in range(1, 4)
        )
        model = ScriptedFakeModel(
            responses=(
                ModelResponse(text=None, raw={}, tool_requests=requests),
                ModelResponse(text="完成。", raw={}),
            )
        )
        release = asyncio.Event()
        first_two_started = asyncio.Event()
        started: list[str] = []
        active = 0
        peak_active = 0

        async def executor(request: ToolRequest) -> object:
            """阻塞前两个调用，以便观察滚动池的真实峰值。"""
            nonlocal active, peak_active
            started.append(request.call_id)
            active += 1
            peak_active = max(peak_active, active)
            if len(started) == 2:
                first_two_started.set()
            await release.wait()
            active -= 1
            return {"call_id": request.call_id}

        registry = ToolRegistry()
        registry.register(
            ToolSpec(
                name="parallel_tool",
                schema={"name": "parallel_tool"},
                executor=executor,
                execution_mode=ToolExecutionMode.ASYNC,
                concurrency_mode=ToolConcurrencyMode.PARALLEL,
            )
        )

        running = asyncio.create_task(
            run_agent(
                model=model,
                user_text="并行执行三个工具",
                registry=registry,
                max_parallel_tool_calls=2,
            )
        )
        await asyncio.wait_for(first_two_started.wait(), timeout=1)
        assert started == ["call-1", "call-2"]
        assert peak_active == 2

        release.set()
        result = await running

        assert result.final_text == "完成。"
        assert started == ["call-1", "call-2", "call-3"]
        assert peak_active == 2
        assert model.requests[1][1:] == (
            AssistantTurn(text=None, tool_requests=requests),
            ToolResult(
                call_id="call-1",
                name="parallel_tool",
                content='{"call_id":"call-1"}',
            ),
            ToolResult(
                call_id="call-2",
                name="parallel_tool",
                content='{"call_id":"call-2"}',
            ),
            ToolResult(
                call_id="call-3",
                name="parallel_tool",
                content='{"call_id":"call-3"}',
            ),
        )

    asyncio.run(scenario())


```

案例来自 `test_run_agent_bounds_parallel_tool_batch_and_preserves_result_order()`。模型第一次请求三个工具调用 `call-1、call-2、call-3`，Runner 的工具并发上限为 2。工具全部完成后，Runner 按请求顺序回填结果，模型第二次返回“完成。”。

本例运行在同一个事件循环线程中。工具是原生异步 Fake Executor，等待 `asyncio.Event`，没有真实网络请求，也没有工作线程。“并发”指多个工具的生命周期重叠；事件循环逐个执行回调，同一时刻只有一段 Python 代码在这个线程上运行。

| 对象 | 谁操作 | 作用 |
| --- | --- | --- |
| `first_two_started` | 第二个启动的工具 `.set()`；测试等待 | 通知测试：前两个工具已启动 |
| `release` | 测试 `.set()`；工具等待 | 暂时扣住工具，使测试能检查中间状态 |

两个 Event 都是状态对象，不是线程或 Task。它们各自保存布尔标志和等待者集合；未置位时，每次 `wait()` 创建一个独立的等待 Future。

## 2. 与总体机制图的对应关系

<!-- OVERVIEW -->

总图的主链是：**创建协程 → Task 推进 → 等待 Future → Future 完成 → 唤醒回调入队 → Task 恢复**。本例用可控的 `Event.set()` 完成等待条件：工具等待测试放行，测试等待工具发出启动通知。

| 总图概念 | 本例对应物 |
| --- | --- |
| 启动入口 | 同步测试最后的 `asyncio.run(scenario())` |
| Task 推进协程 | 测试 T、Runner R、通知辅助 N、工具 W1/W2/W3 |
| Future | Event 的等待者，以及 `wait_for()`、`wait()` 的内部 waiter |
| 完成动作 | Event 置位、工具 Task 返回、Runner Task 返回 |
| 就绪队列 `_ready` | 首次推进、唤醒、完成回调的排队位置 |
| 定时器堆 `_scheduled` | `wait_for(..., timeout=1)` 的保护定时器 |

### 有几个 Task？不是有几个 async def 就有几个

正常路径中，测试及应用代码累计涉及六个 Task，不是六个同时运行的线程，也不是始终同时存活；不包含 `asyncio.run()` 收尾时可能创建的内部清理 Task。

| Task | 创建者 | 负责什么 |
| --- | --- | --- |
| T：测试 Task | `asyncio.run(scenario())` | 装配、等待通知、中途断言、放行、最终断言 |
| R：Runner Task | `create_task(run_agent(...))` | 两次模型调用、批次调度、回填 |
| N：通知辅助 Task | Python 3.11 的 `wait_for()` 内部 `ensure_future()` | 执行 `first_two_started.wait()` |
| W1、W2、W3：工具 Task | Runner 的 `create_task(_execute_registered_tool(...))` | 一次工具执行及结果序列化 |

R 通过直接 `await` 进入 `run_agent → _run_agent_loop → _execute_tool_batch → _execute_parallel_group`，仍然是同一个 R。每个 W 内部直接等待 `_execute_tool()` 和 `executor()`，也不创建额外执行器 Task。

`ScriptedFakeModel.generate()` 虽然声明为 `async def`，函数体没有 `await`，会立即记录上下文并返回。因此 **await 表达式不一定引发任务切换**。

## 3. 关键代码、机制与动作

### 3.1 创建协程，安排 Task，启动循环

```python
async def scenario():
    ...

asyncio.run(scenario())
```

`scenario()` 先创建协程对象；`asyncio.run()` 随后创建事件循环、入口 T 并运行循环。定义或调用协程函数都不会单独启动事件循环。

```python
running = asyncio.create_task(run_agent(...))
await asyncio.wait_for(first_two_started.wait(), timeout=1)
```

`run_agent(...)` 创建协程对象；默认调度方式下，`create_task()` 安排 R 的首次推进回调，返回 Task 对象。T 继续执行，直到进入 `wait_for()` 真正挂起，事件循环才有机会推进其他就绪任务。

### 3.2 工具怎样报告启动，又怎样暂停？

下面是测试执行器原文：

```python
async def executor(request: ToolRequest) -> object:
    """阻塞前两个调用，以便观察滚动池的真实峰值。"""
    nonlocal active, peak_active
    started.append(request.call_id)
    active += 1
    peak_active = max(peak_active, active)
    if len(started) == 2:
        first_two_started.set()
    await release.wait()
    active -= 1
    return {"call_id": request.call_id}
```

工具先更新计数，再等待放行。到 `await release.wait()` 前没有挂起点，其他 Task 不会插进这些计数更新；本例因此不需要锁。这不适用于跨线程共享变量，也不保证跨 await 的多步读写安全。

W1、W2 各自调用尚未置位的 `release.wait()`，分别创建不同的 Future F1、F2 并挂起。“阻塞工具”在这里指协程等待，事件循环线程仍能调度其他任务。

### 3.3 发出通知，不等于测试立即恢复

Python 3.11 的通知链包含这些具体动作：

1. T 进入 `wait_for()`：创建内部 waiter FT、安排 1 秒定时回调，把 `first_two_started.wait()` 包装成 N；T 等待 FT。
2. N 执行 Event 的 `wait()`：未置位时创建 FN，等待 FN。
3. W2 调用 `first_two_started.set()`：置位、完成 FN，N 的唤醒回调入队；W2 继续走到自己的 `release.wait()` 才挂起。
4. 事件循环推进 N；N 返回 `True`，其完成回调再完成 FT。
5. FT 完成安排 T 的唤醒；循环推进 T，`wait_for()` 返回并取消保护定时器，T 开始中途断言。

若 N 开始执行前 Event 已置位，N 的 `wait()` 可以直接返回，通知不会丢失。上述内部 waiter 描述以 CPython 3.11 为准。

1 秒只保护“等待前两个工具启动”，不是整个 Run 的预算。本例 `run_agent()` 的 Run Timeout 与模型 Timeout 都默认为 `None`；`asyncio.timeout(None)` 不安排到期取消。

### 3.4 放行也不会当场切换到工具

```python
assert started == ["call-1", "call-2"]
assert peak_active == 2
release.set()
result = await running
```

`release.set()` 置位、完成 F1/F2、安排工具唤醒回调。T 继续执行，直到 `await running` 等待未结束的 R，才让出控制权。

测试没有 `release.clear()`。W3 稍后调用 `release.wait()` 时，事件已置位，直接返回，不创建等待 Future，也不会在这里挂起。Event 是保持状态的开关，不是一次只放行一个等待者的 Semaphore。

## 4. 完整时序图

<!-- SEQUENCE_A -->

### A. 启动与等待

1. 同步测试调用 `asyncio.run(scenario())`，循环推进 T。
2. T 装配模型与工具，创建 R，然后等待启动通知。
3. R 第一次调用 Fake Model，立即取得三个请求；创建 W1、W2，池占用达到 2。
4. R 进入 `await asyncio.wait(in_flight, return_when=FIRST_COMPLETED)`，等待内部 Future FR。
5. 循环推进 W1：增加计数后等待 F1；推进 W2：通知测试，再等待 F2。
6. 通知通过 N 和 FT 传回 T；T 验证只启动两个工具，峰值是 2。

此刻 W3 尚未创建为 Task，只是请求列表里尚未被 `next_index` 取到的项目。不存在一个已经启动、正在等待 Semaphore 的 W3。

<!-- SEQUENCE_B -->

### B. 放行与收尾

1. T 调用 `release.set()`，安排 W1/W2 唤醒，然后等待 R。
2. W1/W2 恢复，减少 `active`、返回字典；各自完成序列化并返回 ToolResult，工具 Task 才结束。
3. 工具 Task 的完成回调完成 `asyncio.wait()` 的 FR，安排 R 唤醒。
4. R 恢复，取得 `done`，调用 `collect()` 读取结果并移出 `in_flight`，随后补充 W3。
5. W3 运行，直接通过已置位的 Event，完成并通知 R。
6. R 收齐结果、按请求索引排列；追加 AssistantTurn 与三个 ToolResult，第二次模型调用返回“完成。”；R 返回 RunResult。
7. R 的完成安排 T 唤醒；T 读取结果、完成断言并返回。`asyncio.run()` 清理异步生成器与默认执行器等资源，关闭循环，同步测试结束。

图展示正常路径的代表性调度。`FIRST_COMPLETED` 表示至少一个完成即可恢复，**不保证 `done` 只有一个 Task**。本例统一放行后，R 恢复前 W1/W2 可能都已结束，返回集合可以同时包含它们；集合遍历顺序不作保证。

## 5. 并发上限与结果顺序由谁保证？

### 5.1 上限来自创建 Task 前的检查

源码节选如下，省略权限解析、失败分支和外层循环；`request`、`tool` 由省略部分取得：

```python
while (
    not failures
    and next_index < len(tool_requests)
    and len(in_flight) < max_parallel_tool_calls
):
    task = asyncio.create_task(
        _execute_registered_tool(registry, request, tool=tool)
    )
    in_flight[task] = next_index
    next_index += 1

done, _ = await asyncio.wait(
    in_flight, return_when=asyncio.FIRST_COMPLETED,
)
collect(done)
```

Runner 创建两个 Task 后停止补充。工具完成不直接修改 `in_flight`；R 恢复、`collect()` 移除后，下一轮才能使用释放的容量。`asyncio.wait()` 只等待任务，不知道上限是 2。

`asyncio.wait()` 对工具 Task 注册完成回调并创建 FR，回调在满足条件时完成 FR。输入已经是 Task，不会再为每个工具创建一层 Task。

### 5.2 三种数量不要混淆

| 变量 | 记录的事实 |
| --- | --- |
| `started` | 曾经进入执行器的调用身份，不随结束删除 |
| `active` / `peak_active` | 执行器进入后、返回前的当前活跃数 / 历史峰值 |
| `len(in_flight)` | 已创建但 Runner 尚未收集的工具 Task 数 |

例如两个工具已返回、R 还未恢复时，`active` 可以是 0，`len(in_flight)` 仍然是 2。池占用与当前执行器活跃数不同。

### 5.3 收集顺序不决定回填顺序

下面摘出收集与最终排列的关键表达式，省略异常处理与外层返回元组：

```python
request_index = in_flight.pop(task)
completed[request_index] = task.result()

# 全部结束后按请求索引 0、1、2 排列。
[completed[index] for index in range(start_index, next_index)]
```

`task.result()` 同步读取已完成结果或抛出原任务异常，不会等待；对未完成 Task 调用会抛 `InvalidStateError`。这里的 `done` 保证任务已结束。

即使索引 1 先被收集，最终也会按 0、1、2 输出。`call_id` 保证结果与请求的身份对应，索引保证上下文中的排列顺序。

## 6. asyncio API 速查：动作与挂起

| API / 表达式 | 动作 | 是否挂起当前 Task |
| --- | --- | --- |
| `asyncio.run(scenario())` | 创建并运行循环 | 同步入口等待整个过程结束 |
| `asyncio.create_task(coro)` | 创建 Task，默认安排首次推进回调 | 调用本身不挂起 |
| `await coroutine` | 在当前 Task 中推进协程 | 取决于是否真正等待未完成对象 |
| `asyncio.Event()` | 创建未置位 Event 与等待者集合 | 否 |
| `await event.wait()` | 未置位时创建并等待 Future；已置位直接返回 | 未置位时会 |
| `event.set()` | 置位、完成等待 Future、安排唤醒回调 | 否 |
| `await asyncio.wait_for(coro, 1)` | 辅助 Task、内部 waiter、保护定时器 | 本例会 |
| `await asyncio.wait(tasks, FIRST_COMPLETED)` | 等至少一个 Task 完成，返回 done / pending | 本例会 |
| `task.result()` | 读完成结果或抛异常 | 否 |
| `await running` | 等待 Runner 的 RunResult | 本例 R 未结束，会 |
| `asyncio.timeout(None)` | 建立无到期时间的上下文 | 不因进入本身挂起 |

源码中存在、但本例成功路径未走到的 API：`to_thread()`（工具为 ASYNC）、`sleep()`（没有重试）、`wait(..., ALL_COMPLETED)`（没有工具失败）、`task.cancel()` 与 `gather(..., return_exceptions=True)`（没有异常清理）。应另用失败或取消测试解释这些分支。`wait_for()` 正常返回时取消的是定时器 Handle，不是成功的 R 或工具 Task。

## 7. 测试证据、局限与源码依据

### 已验证与尚未强制制造的时序

断言验证了：放行前只有两个工具启动，活跃峰值为 2，三个请求最终全部执行；第二次模型调用的上下文是一条完整 AssistantTurn，加按 1/2/3 排列的 ToolResult；最终文本为“完成。”。

但本测试没有强制 W2 先于 W1 完成，也没有保持 W1 未完成、仅释放 W2 来观察 W3 补位。“按索引重排”“腾出一个位置即可补位”可以从实现解释，但这两种特定时序没有在本测试中被单独强制复现。

1 秒只保护启动通知；`await running` 没有另设测试级超时。未来如果出现完成通知丢失，这个等待仍可能挂住。此处仅记录边界，不改动测试代码。

### 本次核验

- 日期：2026-09-15；环境：macOS、CPython 3.11.15，默认事件循环与 Task 调度。
- 命令：`.venv/bin/python -m pytest tests/test_runner.py::test_run_agent_bounds_parallel_tool_batch_and_preserves_result_order -q -p no:cacheprovider`。
- 实际结果：`1 passed in 0.05s`。图中回调链依据源码推演，没有将其声称为逐回调 Trace。
- 仓库：`Crevettezdx/mini-agent-runtime`；分支：`agent/stage-5-error-concurrency-control`；基准 HEAD：`653835d4a2fdbcbdaf3f71cab43871c8045a0308`。本文依据包含未提交修改的工作树，单独 checkout 该 HEAD 不足以复现。
- 位置：`tests/test_runner.py:1401`（测试）、`:56`（Fake Model）；`src/mini_agent/runner.py:63`（执行边界）、`:163`（滚动池）、`:245`（批次）、`:341`（循环）。
- SHA-256：`tests/test_runner.py` = `3f14865756952d290a9f2399dcb538fc10cfcf3383dcb5649044dea04cccbca7`；`src/mini_agent/runner.py` = `e333f10be94227ba53738cfac6f60f0fdd5f83dd0cf8868ad6d5810255c6c2b1`。
- Python 对照：[3.11 Task 文档](https://docs.python.org/3.11/library/asyncio-task.html)、[同步原语](https://docs.python.org/3.11/library/asyncio-sync.html)、[CPython v3.11.15 tasks.py](https://github.com/python/cpython/blob/v3.11.15/Lib/asyncio/tasks.py)、[locks.py](https://github.com/python/cpython/blob/v3.11.15/Lib/asyncio/locks.py)。`wait_for()` 的内部 waiter 描述限定于 3.11。

## 8. 理解自检与下一项

1. W2 执行 `first_two_started.set()` 后，为什么 T 不会立刻插进 W2 的函数体？
2. W3 明明执行了 `await release.wait()`，为什么这一次不一定挂起？
3. 工具已经完成，为什么 `in_flight` 还可能占着位置？最终结果顺序由谁保证？

下一项最小建议：只先放行 W2、暂时扣住 W1，验证 W3 及时补位以及乱序完成后的有序回填。
