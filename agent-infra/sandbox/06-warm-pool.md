# 预热池如何加速启动

[返回目录](./README.md) · [术语速查](./glossary.md)

> 本页问题：为什么提前启动实例，就能让用户更快拿到沙箱？

![Agent Sandbox WarmPool 流程](./assets/agent-sandbox-warmpool-flow.png)

预热池保存的是**已经完成冷启动、尚未分配给用户的 Ready Sandbox**。

下面以图中的 Kubernetes WarmPool / Claim 模型为例说明；具体资源名称和字段取决于项目与版本。

## 后台预热

```text
SandboxTemplate 定义蓝图
        ↓
SandboxWarmPool 声明 replicas=N
        ↓
Controller 创建缺少的 Sandbox
        ↓
调度、镜像、网络、Runtime、应用初始化
        ↓
保持 N 个未领取的 Ready Sandbox
```

## 用户领取

用户创建 `SandboxClaim` 后，Controller 从可用队列选择一个候选实例，将所有权从 WarmPool 转移给 Claim，并返回 Sandbox 名称、Pod IP 和 Service 地址。原有 Pod 不需要重建或重启。

领取后池中出现缺口，Controller 再异步创建新实例补池。

## 命中与击穿

- **命中**：主要成本是查找、校验和所有权转移，通常是最低延迟路径。
- **击穿**：没有可领取实例，退化到新建 Sandbox；但仍可能命中节点、镜像和网络资源缓存。
- **容量边界**：瞬时可直接领取的实例数约等于池中 Ready 数量 `N`。

> 核心认识：预热池没有让冷启动步骤更快，而是把冷启动提前到用户请求之前完成。

---

相关内容：[预热池未命中，为什么仍可能在 10 秒内启动](./07-warm-pool-miss.md) · [如何组合预热池、快照与冷启动](./12-combined-startup.md)

