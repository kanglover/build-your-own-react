## Render and Commit Phases
前面我们完成了 fiber 节点的创建。

这里有个问题：
一边遍历 element，一边生成新的 DOM 节点并且添加到其父节点上。 在完成整棵树的渲染前，浏览器还要中途阻断这个过程。 那么用户就有可能看到渲染未完全的 UI。我们不想让这个事情发生。

因此我们把修改 DOM 节点的这部分给单独移出。

```js
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​
-  if (fiber.parent) {
-    fiber.parent.dom.appendChild(fiber.dom)
-  }
......
```

我们把修改 DOM 这部分内容记录在 fiber tree 上，通过追踪这颗树来收集所有 DOM 节点的修改，这棵树叫做 wipRoot（work in progress root）。
```js
function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
  }
  nextUnitOfWork = wipRoot
}
​
let nextUnitOfWork = null
let wipRoot = null
```

一旦完成了 wipRoot 这颗树上的所有任务（next unit of work 为 undefined），我们把整颗树的变更提交（commit）到实际的 DOM 上。
```js
function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
​
+  if (!nextUnitOfWork && wipRoot) {
+    commitRoot()
+  }
​
  requestIdleCallback(workLoop)
}
```

这个提交操作都在 commitRoot 函数中完成。我们递归地将所有节点添加到 dom 上。
```js
function commitRoot() {
  commitWork(wipRoot.child)
  wipRoot = null
}
​
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  domParent.appendChild(fiber.dom)
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
```