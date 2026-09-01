# Learning Notes

用于持续沉淀个人技术学习笔记。内容按技术领域分类，每篇笔记尽量以结构图建立整体认识，再用精简文字补充关键概念和结论。

## Agent Infra

### Sandbox

- [Agent 沙箱底层原理与启动加速机制](./agent-infra/sandbox/README.md)

  从 Agent 沙箱技术分层和 microVM 底层结构出发，梳理完整冷启动、预热池、快照恢复及两种加速机制的组合关系。

## 目录约定

```text
learning-notes/
├── README.md
└── agent-infra/
    └── sandbox/
        ├── README.md
        └── assets/
```

- 每个主题使用独立目录。
- 主题正文统一使用 `README.md`。
- 图片等配套资源放在同级 `assets/` 目录。
