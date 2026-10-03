# DeepSeek Harness 思想学习笔记

以论文 [A Programming Paradigm for Spatiotemporal Composability](assets/dsh-paper.pdf) 为主线，用 DSH 源码辅助理解。先认识设计理由与运行过程，按需要再深入形式化证明和实现细节。

第 0 单元给出整篇论文的背景与论证路线，后续五个单元依次展开。第 0–5 单元均已成稿，核心学习路线已完成；进一步的算法与证明可按各篇的“按需深入”继续学习。

| 单元 | 内容 | 状态 |
|---|---|---|
| [0 · 论文总览](00-overview.md) | 背景、痛点、作者方案及其价值 | 已成稿 |
| [1 · 动态组合需要同时管理依赖与影响](01-dynamic-composition.md) | effect、coeffect 与两种可组合性 | 已成稿 |
| [2 · 时间可组合性](02-temporal-composability.md) | 建立能力时记下撤销动作；`ctx.effect` 与工具注册 | 已成稿 |
| [3 · 空间可组合性](03-spatial-composability.md) | 依赖变化与插件启停；隔离、拦截及按需深入指引 | 已成稿 |
| [4 · 用生命周期协调依赖与清理](04-lifecycle-coordination.md) | context、Fiber、原文图 1–2 与有序退出 | 已成稿 |
| [5 · 让运行中的 DSH 随需求调整组成](05-composing-dsh.md) | 配置装配、实例管理、运行总图及适用边界 | 已成稿 |

每个单元先讲解和讨论，再确认成稿；完成标准是能用自己的话解释核心问题，并指出一个 DSH 对应例子。图和文字保持相同粒度，每次只读一小段有助于理解的源码。

论文的正式应用案例是 Koishi；自演化 agent harness 是后续验证方向。笔记中的 DSH 源码与自绘图属于辅助说明。

笔记所引论文页码为正文印刷页码；本地 PDF 的页序与其一致。源码引用固定于 [deepseek-harness 的 `639ed01` 版本](https://github.com/deepseek-ai/deepseek-harness/tree/639ed015397290b3745d163aafe02ffee4aa3f84)，避免后续更新造成行号漂移。
