# Agent 沙箱学习笔记

每个学习单元只回答一个问题，可独立阅读。图与对应说明放在同一页；不熟悉的名词随时查[术语速查](./glossary.md)。

## 先查术语

- [核心术语与协作关系](./glossary.md)：VMM、KVM、Node Agent、envd、Kata 等分别负责什么、怎样配合。

## 一、建立架构骨架

- [云上 VM 如何承载 Kubernetes 容器](./01-kubernetes-on-vm.md)：一台物理机如何支撑云 VM、Worker Node、Pod 和容器？
- [Agent 沙箱技术全景](./02-sandbox-landscape.md)：如何区分编排管理、运行时接入和执行隔离？
- [Host、Guest、VMM 与 KVM 的位置](./03-host-guest-vmm.md)：用户进程、两套内核和虚拟化组件分别在哪里运行？
- [沙箱平台如何直接管理 Cloud Hypervisor](./04-cloud-hypervisor.md)：API 请求如何变成节点上的一个独立沙箱 VM？

## 二、理解启动与预热池

- [从零启动一个可用沙箱](./05-cold-start.md)：一次完整冷启动必须完成哪些工作？
- [预热池如何加速启动](./06-warm-pool.md)：为什么提前启动实例，就能让用户更快拿到沙箱？
- [预热池未命中，为什么仍可能在 10 秒内启动](./07-warm-pool-miss.md)：没有 Ready 实例可领，是否意味着所有东西都要从零准备？

## 三、理解快照

- [模板快照如何生成](./08-snapshot-template.md)：要从运行中继续执行，必须事先保存哪些状态？
- [快照恢复如何缩短启动](./09-snapshot-restore.md)：用户请求到来后，哪些冷启动步骤可以被替代？
- [CoW 为什么能减少克隆成本](./10-copy-on-write.md)：多个实例怎样复用模板，同时保留各自的修改？

## 四、比较与组合

- [预热池与快照：速度、成本和扩展能力](./11-pool-vs-snapshot.md)：为什么预热池命中可能更快，而快照更适合复用和突发扩展？
- [如何组合预热池、快照与冷启动](./12-combined-startup.md)：如何兼顾日常低延迟和突发流量？

## 附录：项目引子

- [项目引子：OpenSandbox](./appendix-opensandbox.md)：统一沙箱接口的平台，在整体架构中处于什么位置？
- [项目引子：CubeSandbox](./appendix-cubesandbox.md)：围绕 VM 与模板快照优化的平台，如何对应通用机制？

## 如何复习

首次阅读按上面顺序建立框架。复习时直接选择一个问题，先看图，再尝试用自己的话回答页首问题；每页底部提供相关单元链接。

## 参考项目

- [OpenKruise Agents](https://github.com/openkruise/agents)
- [Kubernetes SIG Agent-Sandbox](https://github.com/kubernetes-sigs/agent-sandbox)
- [Kata Containers](https://github.com/kata-containers/kata-containers)
- [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor)
- [OpenSandbox](https://github.com/alibaba/OpenSandbox)
- [CubeSandbox](https://github.com/TencentCloud/CubeSandbox)

Sandbox / Agent 沙箱是通用概念。OpenKruise Agents 和 SIG Agent-Sandbox 是不同项目；Kata 是容器运行时集成方案，Cloud Hypervisor 等是 VMM。
