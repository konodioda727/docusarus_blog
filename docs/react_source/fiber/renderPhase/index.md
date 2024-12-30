---
description: render 阶段, 负责 fiber 树的构建
sidebar_position: 1
---
# Render 阶段
之前提到, `renderRootSync` 会调用 `workloop`

在本章,我们会深入理解 `workloop` 究竟做了什么工作

但是首先,我们先了解一下`workloop`的大致逻辑流程,这对我们深入代码层面很有帮助

:::info[workloop 流程]
```mermaid
---
title: 整体流程
---
flowchart TD
    classDef visited fill:#f96;
    classDef target fill:#ff4;
    classDef completed fill:#008000;
    A((1))
    B((2))
    C((3))
    D((4))
    E((5))
    A1((1))
    B1((2))
    C1((3))
    D1((4))
    E1((5))
    A2((1))
    B2((2))
    C2((3))
    D2((4))
    E2((5))
    A3((1))
    B3((2))
    C3((3))
    D3((4))
    E3((5))
    A4((1))
    B4((2))
    C4((3))
    D4((4))
    E4((5))
    A5((1))
    B5((2))
    C5((3))
    D5((4))
    E5((5))
    A6((1))
    B6((2))
    C6((3))
    D6((4))
    E6((5))
    A7((1))
    B7((2))
    C7((3))
    D7((4))
    E7((5))
    subgraph 标记节点1完成
    A7:::completed --> B7
    A7 --> C7:::completed
    B7:::completed --> D7:::completed
    B7 --> E7:::completed
    _A7@{ shape: text, label: "节点1完成,将target置为null,退出循环" }
    end
    subgraph 标记节点3完成
    A6:::target --> B6:::visited
    A6 --> C6:::completed
    B6:::completed --> D6:::completed
    B6 --> E6:::completed
    _A6@{ shape: text, label: "节点3无亲无故,执行\nbeginWork、completeWork\n并将 target 设为节点1\n(接下来 target 节点只执行 completeWork)" }
    end
    subgraph 标记节点2完成
    A5:::visited --> B5:::visited
    A5 --> C5:::target
    B5:::completed --> D5:::completed
    B5 --> E5:::completed
    _A5@{ shape: text, label: "节点2有兄弟节点3,因此将 target 定为节点3" }
    end
    subgraph 标记节点5完成
    A4:::visited --> B4:::target
    A4 --> C4
    B4 --> D4:::completed
    B4 --> E4:::completed
    _A4@{ shape: text, label: "节点5没有子节点也没有兄弟节点,因此在 beiginWork 后会执行 completeWork, 并将下一个目标设置为父节点2(此时接下来的目标只会执行completeWork)" }
    end
    subgraph 标记节点4完成
    A3:::visited --> B3:::visited
    A3 --> C3
    B3 --> D3:::completed
    B3 --> E3:::target
    _A3@{ shape: text, label: "节点4执行beiginWork\n由于无子节点但有兄弟节点\n因此进入 complete 阶段\n并将下一个目标设置为兄弟节点 5" }
    end
    subgraph 访问节点2
    A2:::visited --> B2:::visited
    A2 --> C2
    B2 --> D2:::target
    B2 --> E2
    _A2@{ shape: text, label: "节点2做 beiginWork, 由于有4、5子节点\n因此不做 completeWork \n将下一个目标设置为第一个子节点4" }
    end
    subgraph 访问节点1
    A1:::visited --> B1:::target
    A1 --> C1
    B1 --> D1
    B1 --> E1
    _A1@{ shape: text, label: "节点1做 beginWork, 由于有2、3子节点\n因此将第一个子节点2设为下一个目标" }
    end
    subgraph 初始状态
    A --> B
    A --> C
    B --> D
    B --> E
    _A@{ shape: text, label: "一共五个节点,开始构造 fiberTree" }
    end
    初始状态 --> 访问节点1 --> 访问节点2
    标记节点4完成 --> 标记节点5完成 --> 标记节点2完成
    标记节点3完成 --> 标记节点1完成
```
:::

## workloop/performUnitOfWork

`workloop`很纯粹，循环`workInProgress`应用更改

```tsx
function workLoopSync() {
  // Perform work without checking if we need to yield between fiber.
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

`performUnitOfWork`中可以看到两个在[前备知识](../intro/index.md)中熟悉的身影:`beginWork`和`completeWork`

从此处开始，更新粒度深入为`fiberNode`级别

```tsx
  // current 树上节点
  // 通过 wip 的 alternate 指向
  const current = unitOfWork.alternate;

  let next;
  // 除去 dev 以及 profiler 逻辑
  // 主体逻辑仅有一行
  // 对当前节点做 beginWork
  // beginWork 返回下一个节点
  next = beginWork(current, unitOfWork, entangledRenderLanes);

  if (!disableStringRefs) {
    resetCurrentFiber();
  }
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // 如果后续没有节点,代表本节点的 unitWork 完成
    // 调用 completeWork 进入下一阶段
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
```
其中 `beginWork` 即为 beginWork 阶段入口
`completeUnitOfWork` 即为 completeWork 阶段入口

## 小结
在本节中,我们分析了 `workloop`的结构

并将渲染流程分为 `beginWork` 以及 `completeWork`阶段

结构如下:
```mermaid
flowchart TD
    subgraph workloop;
        A{是否存在下一个节点}
        B[退出]
        A --> |是|performUnitOfWork --> A
        A --> |否|B
        subgraph performUnitOfWork
            beginWork[beginWork: 探寻过程,深度优先遍历]
            completeWork[completeWork: 回溯过程,上移 effectList]
        end
    end
```
:::info[主要函数为 `performUnitOfWork`, 这也是本篇的重点 ]

整理出其具体流程如下,可在阅读完本章内容后回头阅读本图,相信会有不同收获:
```mermaid
---
title: performUnitOfWork 逻辑
---
flowchart TD
    A0{是否还有子节点}
    A1[beginWork: 递归构建子节点]
    A{是否为根节点}
    B[代表节点构建完成]
    C{是否有 sibling 节点}
    D[将 wip 指向 第一个 sibling 节点]
    E[将 wip 回退到父级节点]
    F[执行 completeWork 将节点冒泡]
    _E[updateMemoComponent]
    _F[updateFunctionComponent]
    _G[updateClassComponent ...]
    _H[renderWithHooks: 执行组件,构建 fiber 节点]
    _I[reconcileChildren: 协调其子组件]
    A0 --> |是|beginWork
    beginWork --> completeWork
    A --> |否|completeWork
    A --> |是|B
    subgraph beginWork
    A1 --> _E;
    A1 --> _F;
    A1 --> _G;
    _F --> _H;
    _F --> _I;
    end
    A0 --> |否|A
    subgraph completeWork
    F --> C
    C --> |是|D
    C --> |否|E
    end
```
:::

那么就让我们趁热打铁,看看 [beiginWork](./beginWork.md) 中, React 都做了哪些工作吧
