---
description: beginWork 结束后回溯
sidebar_position: 3
---
# CompleteWork

在结束了 beginWork 之后,代表该节点已经是子树中最深的节点

换句话说,该棵子树已经遍历到底部了,是时候拓展广度了

接下来的工作很简单: ***寻找到下一个需要构建的兄弟节点,对它进行 beginWork,直到构建出一整棵树***

流程大概如下:
```mermaid
---
title: completeWork 阶段逻辑
---
flowchart TD
    C{是否有 sibling 节点}
    D[将 wip 指向 第一个 sibling 节点]
    E[将 wip 回退到父级节点]
    F[执行 completeWork 将节点冒泡]
    F --> C
    C --> |是|D
    C --> |否|E
```

## completeUnitOfWork

主要结构为如下循环:
```jsx
do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;
    // ...
    // 此处省略 profiler mode
    next = completeWork(current, completedWork, entangledRenderLanes);
     if (next !== null) {
      // Completing this fiber spawned new work. Work on that next.
      workInProgress = next;
      return;
    }

    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    }
    // Otherwise, return to the parent
    // $FlowFixMe[incompatible-type] we bail out when we get a null
    completedWork = returnFiber;
    // Update the next thing we're working on in case something throws.
    workInProgress = completedWork;
} while (completedWork !== null);
```

逻辑很简单:

1. 如果当前 `completeWork` 产生了新的节点任务,那么退出当前 `completeWork` 阶段, 对新节点重新 `performUnitOfWork`
2. 如果该节点 `completeWork` 做完,还有兄弟节点,也结束当前 `completeWork` 阶段, 则继续对兄弟节点做 `performUnitOfWork`
3. 如果没有兄弟节点,则将下一个`completeWork`的对象设置为父节点,直到到达根节点(`returnFiber === completeWork`)

:::info[串联一下]
***这一部分也是 react 遍历的时候能够回溯的原因***

了解过 beginWork 与 completeUnitOfWork 之后, 会发现原来那幅图的实现依靠的是两个循环:

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

- **第一个循环**: 最外层的 workloop, ***负责深度优先遍历***, 对每个组件做 beginWork, 创建 fiber 树, 找到最深的节点
- **第二个循环**: completeWork 循环, ***负责广度优先遍历以及回溯***, 对每一个组件做 completeUnitOfWork, 并确定应该回溯的节点,直到回到根节点

然而 completeWork 的工作不止于此, 我们深入到代码层面观察 completeWork 的工作
:::

## completeWork

代码主题内容如下:
```jsx
    case IncompleteFunctionComponent: {
      if (disableLegacyMode) {
        break;
      }
      // Fallthrough
    }
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case Profiler:
    case ContextConsumer:
    case MemoComponent:
      bubbleProperties(workInProgress);
      return null;
```

在大多数情况下,只会执行 `bubbleProperties` 一个函数

## bubbleProperties

```jsx
    let child = completedWork.child;
      while (child !== null) {
        newChildLanes = mergeLanes(
          newChildLanes,
          mergeLanes(child.lanes, child.childLanes),
        );

        subtreeFlags |= child.subtreeFlags;
        subtreeFlags |= child.flags;

        // Update the return pointer so the tree is consistent. This is a code
        // smell because it assumes the commit phase is never concurrent with
        // the render phase. Will address during refactor to alternate model.
        child.return = completedWork;

        child = child.sibling;
      }
```

:::tip[fiber flag]
在讲解代码作用之前,我们先了解一下代码中的 flag, flag 是 react 中代表节点操作类型的标志,有以下几种值:

- Placement（插入）
- Update（更新）
- Deletion（删除）
- Ref（引用更新）
- Callback（回调）
:::

在代码中, react 将子节点的 child flag 添加到父节点中,这是一个很奇妙的操作,我们来详细讲解一下

## bubblePropperties

先来考虑没有 flag 的情况: 在更新时, react 不知道哪个节点需要更新, 需要遍历一整棵树,并且需要依次对比 baseProps 以及 baseState 才能得出结果,这样操作十分费时

那 react 这种做法,起到的就是 ***剪枝*** 的效果: 

**父节点中的 flag 包含子节点的 flag, 那么寻找子树中是否需要更新时可以直接判断 flag 中包不包含 placement 等操作,如果不包含就舍弃遍历子树**

## completeUnitOfWork 概览

```mermaid
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
    subgraph 对节点1做completeWork
    A4:::completed --> B4:::completed
    A4 --> C4:::completed
    B4 --> D4:::completed
    B4 --> E4:::completed
    _A4@{ shape: text, label: "没有 sibling 节点, 同时 return 节点为 null, 代表 completeWork 结束, 结束" }
    end
    subgraph 对节点3做performUnitOfWork
    A3:::target --> B3:::completed
    A3 --> C3:::completed
    B3 --> D3:::completed
    B3 --> E3:::completed
    _A3@{ shape: text, label: "节点3做 beginWork, 在 completeWork 阶段\n没有兄弟节点,将父节点设置为下一个target" }
    end
    subgraph 对节点2做completeWork
    A2 --> B2:::completed
    A2 --> C2:::target
    B2 --> D2:::completed
    B2 --> E2:::completed
    _A2@{ shape: text, label: "节点2做 completeWork, 有 sibling, 由此退出 completeWork 阶段, 对节点3做 performUnitOfWork" }
    end
    subgraph 对节点5做performUnitOfWork
    A1 --> B1:::target
    A1 --> C1
    B1 --> D1:::completed
    B1 --> E1:::completed
    _A1@{ shape: text, label: "节点5 beiginWork 做完\n 在 completeWork 阶段\n没有兄弟节点,将父节点设置为下一个target" }
    end
    subgraph 对节点4做completeWork
    A --> B
    A --> C
    B --> D:::completed
    B --> E:::target
    _A@{ shape: text, label: "有 sibling 节点,退出 completeWork 阶段, 对 5 节点 performUnitOfWork" }
    end
    对节点4做completeWork --> 对节点5做performUnitOfWork --> 对节点2做completeWork
    对节点3做performUnitOfWork --> 对节点1做completeWork 
```