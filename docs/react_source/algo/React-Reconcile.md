---
description: reconcile 算法,将 o(n^3) 复杂度 降低为 o(n)
---

# React-Reconcile
这可以说是 React 中最重要的部分,承担了主要的渲染逻辑
但是它的作用也很简单: 负责创建、修改 VDOM

## 