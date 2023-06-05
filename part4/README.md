## Reconciliation
至今我们只完成了 添加 东西到 DOM 上这个操作，更新和删除 node 节点呢？

我们还需要比较 render 中新接收的 element 生成的 fiber 树和上次提交到 DOM 的 fiber 树。上次提交到 DOM 节点的 fiber 树” 的”引用”（reference）。我们称之为 currentRoot。

在每一个 fiber 节点上添加 alternate 属性用于记录旧 fiber 节点（上一个 commit 阶段使用的 fiber 节点）的引用。

```js
function commitRoot() {
  commitWork(wipRoot.child);
+ currentRoot = wipRoot;
  wipRoot = null;
}


function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
+   alternate: currentRoot
  }
  nextUnitOfWork = wipRoot
}
​
```

root 节点已经记录好了，我们还需要给每一个 fiber 节点上添加 alternate 属性。

把 performUnitOfWork 中创建新 fiber 节点的代码抽出来放在 reconcileChildren 函数中。这个函数会调和（reconcile）旧的 fiber 节点 和新的 react elements。

```js
function reconcileChildren(wipFiber, elements) {
  let index = 0
  let oldFiber =
    wipFiber.alternate && wipFiber.alternate.child
}
```

element 是我们想要渲染到 DOM 上的东西，oldFiber 是我们上次渲染 fiber 树。我们需要比较这两者之间的差异，看看需要在 DOM 上应用哪些改变。

以下是比较的步骤：
* 对于新旧节点类型是相同的情况，我们可以复用旧的 DOM，仅修改上面的属性
* 如果类型不同，意味着我们需要创建一个新的 DOM 节点
* 如果类型不同，并且旧节点存在的话，需要把旧节点的 DOM 给移除

```js
function reconcileChildren(wipFiber, elements) {
  let index = 0
  let oldFiber =
    wipFiber.alternate && wipFiber.alternate.child
  let prevSibling = null
​
  while (
    index < elements.length ||
    oldFiber != null
  ) {
    const element = elements[index]
    let newFiber = null
​
    const sameType = oldFiber
      && element
      && element.type == oldFiber.type

    if (sameType) {
        // update the node
    }
    if (element && !sameType) {
        // add this node
    }
    if (oldFiber && !sameType) {
        // delete thi oldFiber's node
    }
​
    if (oldFiber) {
      oldFiber = oldFiber.sibling
    }
  }
}
```

当新的 element 和旧的 fiber 类型相同, 我们对 element 创建新的 fiber 节点，并且复用旧的 DOM 节点，但是使用 element 上的 props。

我们需要在生成的fiber上添加新的属性：effectTag。在 commit 阶段（commit phase）会用到它。

```js
const sameType =
    oldFiber &&
    element &&
    element.type == oldFiber.type
​
if (sameType) {
    newFiber = {
        type: oldFiber.type,
        props: element.props,
        dom: oldFiber.dom,
        parent: wipFiber,
        alternate: oldFiber,
        effectTag: "UPDATE"
    }
}
```

对于需要生成新 DOM 节点的 fiber，我们需要标记其为 PLACEMENT。
```js
if (element && !sameType) {
    newFiber = {
        type: element.type,
        props: element.props,
        dom: null,
        parent: wipFiber,
        alternate: null,
        effectTag: "PLACEMENT"
    }
}
```

对于需要删除的节点，我们并不会去生成 fiber，因此我们在旧的fiber上添加标记。

但是当我们提交（commit）整颗 fiber 树（wipRoot）的变更到 DOM 上的时候，并不会遍历旧 fiber。
```js
if (oldFiber && !sameType) {
    oldFiber.effectTag = "DELETION"
    deletions.push(oldFiber)
}
```

我们提交变更到 DOM 上的时候，也需要把这个数组中的 fiber 的变更（其实是移除 DOM）给提交上去。
```js
function commitRoot() {
  deletions.forEach(commitWork)
  commitWork(wipRoot.child)
  currentRoot = wipRoot
  wipRoot = null
}
```

现在，我们对 commitWork 函数略作修改来处理我们新添加的 effectTags。

如果 fiber 节点有我们之前打上的 PLACEMENT 标，那么在其父 fiber 节点的 DOM 节点上添加该 fiber 的 DOM。相反地，如果是 DELETION 标记，我们移除该子节点。如果是 UPDATE 标记，我们需要更新已经存在的旧 DOM 节点的属性值。


```js
function commitWork(fiber) {
  if (!fiber) {
    return;
  }

  const domParent = fiber.parent.dom
  if (
    fiber.effectTag === "PLACEMENT" &&
    fiber.dom != null
  ) {
    domParent.appendChild(fiber.dom)
  } else if (
    fiber.effectTag === "UPDATE" &&
    fiber.dom != null
  ) {
    updateDom(
      fiber.dom,
      fiber.alternate.props,
      fiber.props
    )
  } else if (fiber.effectTag === "DELETION") {
    domParent.removeChild(fiber.dom)
  }
​
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
```

updateDom 函数中。比较新老 fiber 节点的属性， 移除、新增或修改对应属性。比较特殊的属性值是事件监听，如果属性值以 “on” 作为前缀，我们需要以不同的方式来处理这个属性。

```js
const isEvent = key => key.startsWith("on")
const isProperty = key =>
  key !== "children" && !isEvent(key)
const isNew = (prev, next) => key =>
  prev[key] !== next[key]
const isGone = (prev, next) => key => !(key in next)
function updateDom(dom, prevProps, nextProps) {
  //Remove old or changed event listeners
  Object.keys(prevProps)
    .filter(isEvent)
    .filter(
      key =>
        !(key in nextProps) ||
        isNew(prevProps, nextProps)(key)
    )
    .forEach(name => {
      const eventType = name
        .toLowerCase()
        .substring(2)
      dom.removeEventListener(
        eventType,
        prevProps[name]
      )
    })
  // Remove old properties
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach(name => {
      dom[name] = ""
    })
​
  // Set new or changed properties
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      dom[name] = nextProps[name]
    })

   // Add event listeners
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      const eventType = name
        .toLowerCase()
        .substring(2)
      dom.addEventListener(
        eventType,
        nextProps[name]
      )
    })
}
```