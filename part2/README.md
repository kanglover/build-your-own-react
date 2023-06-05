## Concurrent Mode
递归调用会导致一些问题。

一旦开始渲染，在我们将 react element 数渲染出来之前没法暂停。如果这颗树很大，可能会对主线程进行阻塞。这意味着浏览器的一些高优先级任务会一直等待渲染完成，如：用户输入，保证动画顺畅。


因此我们需要将整个任务分成一些小块，每当我们完成其中一块之后需要把控制权交给浏览器，让浏览器判断是否有更高优先级的任务需要完成。
```js
let nextUnitOfWork = null
​
function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
  requestIdleCallback(workLoop)
}
​
requestIdleCallback(workLoop)
​
function performUnitOfWork(nextUnitOfWork) {
  // TODO
}
​
```

## Fibers
为了把所有任务单元组织起来我们需要一个数据结构： fiber 树

每一个 element 都是一个 fiber，每一个 fiber 都是一个任务单元。

在 render 中，我们创建了 根fiber，并且将其设为 nextUnitOfWork 作为第一个任务单元，剩下的任务单元会通过 performUnitOfWork 函数完成并返回，每个 fiber 节点完成下述三件事：
1. 把 element 添加到 DOM 上
2. 为该 fiber 节点的子节点新建 fiber
3. 挑出下一个任务单元

这个数据结构的其中一个目的是为了更方便地找到下一个任务单元。因此每个 fiber 都会指向它的第一个子节点、它的下一个兄弟节点 和 父节点。

下一个任务单元的选择：先选择它的 child fiber 节点将作为下一个任务单元，如果没有 child，那么它的兄弟节点（sibling）会作为下一个任务单元。如果一个 fiber 既没有 child 又没有 sibling，它的 “uncle” 节点（父节点的兄弟）将作为下一个任务单元。如果 parent 节点没有 sibling，就继续找父节点的父节点，直到该节点有 sibling，或者直到达到根节点。


render 函数把 nextUnitOfWork 置为 fiber 树的根节点。
```js
function render(element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element],
    },
  }
}
​
let nextUnitOfWork = null
```

当浏览器有空闲的时候，会调用 workLoop 就开始遍历整颗树。

通过 fiber.dom 这个属性来维护创建的 DOM 节点。
```js
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }
​
  // 为每个子节点创建对应的 fiber 节点
  const elements = fiber.props.children
  let index = 0
  let prevSibling = null
​
  while (index < elements.length) {
    const element = elements[index]
​
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    }

    // 设置指向
    if (index === 0) {
      fiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }
​
    prevSibling = newFiber
    index++
  }


  // 返回下一个工作单元
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }

}
```