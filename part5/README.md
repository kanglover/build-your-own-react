## Function Components
接下来我们需要支持函数组件。

再来变更下这个例子，我们使用简单的函数组件，返回一个 h1 元素。
```js
function App(props) {
  return <h1>Hi {props.name}</h1>
}
const element = <App name="foo" />
const container = document.getElementById("root")
Didact.render(element, container)
```
jsx 转换成 js 如下
```js
function App(props) {
  return Didact.createElement(
    "h1",
    null,
    "Hi ",
    props.name
  )
}
const element = Didact.createElement(App, {
  name: "foo",
})
```

函数组件的不同点在于：
* 函数组件的 fiber 没有 DOM 节点
* 并且子节点由函数运行得来而不是直接从 props 属性中获取

当 fiber 类型为函数时，我们使用不同的函数来进行 diff。
```js
function performUnitOfWork(fiber) {
  const isFunctionComponent =
    fiber.type instanceof Function
  if (isFunctionComponent) {
    updateFunctionComponent(fiber)
  } else {
    updateHostComponent(fiber)
  }
```

运行这个函数会返回 element，一旦我们拿到了这个子节点，剩下的调和（reconciliation）工作和之前一致，我们不需要修改任何东西了。
```js
function updateFunctionComponent(fiber) {
  const children = [fiber.type(fiber.props)]
  reconcileChildren(fiber, children)
}
```


另外我们需要修改 commitWork 函数。

当我们的 fiber 没有 DOM 的时候需要修改两个东西：

首先找 DOM 节点的父节点的时候我们需要往上遍历 fiber 节点，直到找到有 DOM 节点的 fiber 节点。

```js
function commitWork(fiber) {
  if (!fiber) {
    return
  }
​

- const domParent = fiber.parent.dom
+ let domParentFiber = fiber.parent
+ while (!domParentFiber.dom) {
+   domParentFiber = domParentFiber.parent
+ }
+ const domParent = domParentFiber.dom
```

移除节点也同样需要找到该 fiber 下第一个有 DOM 节点的 fiber 节点。
```js
function performUnitOfWork(fiber) {
  ......
  else if (fiber.effectTag === "DELETION") {
+   commitDeletion(fiber, domParent)
  }
​
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
​
function commitDeletion(fiber, domParent) {
  if (fiber.dom) {
    domParent.removeChild(fiber.dom)
  } else {
    commitDeletion(fiber.child, domParent)
  }
}
```