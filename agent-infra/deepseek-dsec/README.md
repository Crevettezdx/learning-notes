# DSec 论文学习笔记

论文：[DeepSeek Elastic Compute (DSec)](https://arxiv.org/pdf/2609.22978v1)

按“需求与负载 → 平台架构 → 核心机制 → RL 协同与安全”的顺序学习，实验依据随对应机制就地解读。每一部分先由我阅读论文，给出中文译解与重点提炼；你看过后我们再讨论。得到你的完成确认后，再生成对应笔记并在此处加入链接。

| 序号 | 逻辑单元 | 状态 |
| --- | --- | --- |
| 01 | [为什么需要 DSec；四种后端如何选择](01-why-dsec-needs-four-backends.md) | 已完成 |
| 02 | [沙箱架构与生命周期](02-sandbox-architecture-and-lifecycle.md) | 已完成 |
| 03 | [生产负载如何推导出三类平台难题](03-workload-to-platform-challenges.md) | 已完成 |
| 04 | [图 9 绿色区域：可组合环境层](04-composable-environment-layers.md) | 已完成 |
| 05 | [镜像按需加载](05-on-demand-image-loading.md) | 已完成 |
| 06 | [microVM 内存共享与回收](06-microvm-memory-sharing-and-reclamation.md) | 已完成 |
| 07 | [CPU 超分配与服务质量](07-cpu-overcommit-and-qos.md) | 已完成 |
| 08 | [rollout 状态留在 DSec，训练抢占后继续执行](08-rollout-preemption-and-recovery.md) | 已完成 |
| 09 | [Agent 自建环境需要清理残留并约束访问](09-agent-built-environments-and-access-control.md) | 已完成 |
| 附录 | [EROFS 与 ext4：只读镜像与可写文件系统](appendix-erofs-and-ext4.md) | 已完成 |

## 实验证据入口

| 机制 | 论文实验 | 对应笔记 |
| --- | --- | --- |
| 可组合环境层 | 图 11：EROFS 挂载与 Tar 解包 | [04｜环境组合](04-composable-environment-layers.md) |
| 镜像按需加载 | 图 10：突发启动与磁盘写入 | [05｜按需加载](05-on-demand-image-loading.md) |
| microVM 内存优化 | 图 12：内存占用与 CPU 代价 | [06｜内存共享与回收](06-microvm-memory-sharing-and-reclamation.md) |
| CPU 服务质量 | 图 13：背景负载下的 Agent 延迟 | [07｜CPU 调度](07-cpu-overcommit-and-qos.md) |

原论文截图放在 `assets/source/`，新增原理图放在 `assets/explainer/`。笔记中的每张图标注图号或主题、论文页码及其要说明的问题。

## 笔记写作约定

- 每篇回答一个完整的核心问题，按“背景与痛点 → 为什么这样设计 → 原理过程 → 结果”串成可独立阅读的叙事，并链接前后篇。
- 后续笔记的篇名、章节标题和图表标题用简短陈述句直接写出关键结论，不用疑问句作标题。
- 技术原理以图为主，图下用简短步骤讲清组件关系和过程；背景、痛点与设计动机用必要的文字铺垫，避免读者只看图却无法理解因果。
- 论文原图直接复用并说明怎么看、得到什么信息；复杂机制用新增原理图。每篇纳入讨论中确认的关键解释和已画的图，保持简明，不把延伸细节挤进主笔记。
