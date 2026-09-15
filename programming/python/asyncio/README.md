# Python asyncio 学习笔记

围绕“协程为什么会暂停、事件循环如何再次唤醒它”整理 Python `asyncio` 的基础运行机制。

## 阅读顺序

- [事件循环、Task 与 Future 如何协作](./01-event-loop-task-future.md)：从 `async def f()` 到协程结束，串起创建、调度、等待、完成通知与取消。
- [实例：三个工具如何在并发上限 2 下完成执行](./02-bounded-tool-batch-walkthrough.md)：沿测试解释 Event、Task、wait_for、wait 和有序回填；[完整图文版](./assets/asyncio-tool-batch-walkthrough.html)包含总体机制图和两段时序图。

## 学习范围

当前以 Python 3.11、macOS 默认 `SelectorEventLoop` 的典型调度路径为例；图中的“可运行、执行中、等待中”是教学状态，不是 `Task` 的公开状态枚举。
