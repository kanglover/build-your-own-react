## Hooks
函数组件添加状态。

让我们把例子改成经典的计数组件。每点击一次，状态自增一。

我们使用 Didact.setState 来读取和修改计数器的值。
```js
function Counter() {
  const [state, setState] = Didact.useState(1)
  return (
    <h1 onClick={() => setState(c => c + 1)}>
      Count: {state}
    </h1>
  )
}
const element = <Counter />
```

我们在这里调用 Counter 函数，并且在这个函数中调用 useState。
在调用函数组件前需要初始化一些全局变量。我们需要在 useState 函数中用到这些全局变量。

首先我们设置 work in progress fiber。

在对应的 fiber 上加上 hooks 数组以支持我们在同一个函数组件中多次调用 useState。然后我们记录当前 hook 的序号。
```js
let wipFiber = null
let hookIndex = null
​
function updateFunctionComponent(fiber) {
  wipFiber = fiber
  hookIndex = 0
  wipFiber.hooks = []
  const children = [fiber.type(fiber.props)]
  reconcileChildren(fiber, children)
}
​
```

当函数组件调用 useState，我们校验 fiber 对应的 alternate 字段下的旧 fiber 是否存在旧 hook。hook 的序号用以记录是该组件下的第几个 useState。

如果存在旧 hook，我们将旧 hook 的值拷贝一份到新的 hook。 如果不存在，将 state 初始化。

然后在 fiber 上添加新 hook，自增 hook 序号，返回状态。
```js
function useState(initial) {
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex]
  const hook = {
    state: oldHook ? oldHook.state : initial,
  }
​
  wipFiber.hooks.push(hook)
  hookIndex++
  return [hook.state]
}
```

useState 还需要返回一个可以更新状态的函数，我们定义 setState，它接收一个 action参数。（在 Counter 的例子中， action 是自增state 的函数）

我们将 action 推入刚才添加的 hook 里的队列。

之后和之前在 render 函数中做的一样，我们将 wipRoot 设置为当前 fiber，之后我们的调度器会帮我们开始新一轮的渲染的。

```js
......
const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex]
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: [],
  }
​
  const setState = action => {
    hook.queue.push(action)
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot,
    }
    nextUnitOfWork = wipRoot
    deletions = []
  }
​
  wipFiber.hooks.push(hook)
  hookIndex++
  return [hook.state, setState]
}
```