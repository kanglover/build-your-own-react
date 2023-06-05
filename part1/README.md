## The createElement Function
JSX 通过构建工具 Babel 转换成 JS。

这个转换过程很简单：将标签中的代码替换成 createElement，并把标签名、参数和子节点作为参数传入。
```js
const element = <h1 title="foo">Hello</h1>
const container = document.getElementById("root")
ReactDOM.render(element, container)


=>
const element = React.createElement(
    'h1',
    {title: 'foo'},
    'Hello'
)
```

createElement 生成一个 JSX 元素。type 属性是个字符串，指定了 DOM 节点的类型。props 也是个对象，从 JSX 属性中接收所有 key、value，并且有一个特别的属性 children。
```js
const element = {
    type: "h1",
    props: {
        title: "foo",
        children: "Hello",
    }
}
```

## The render Function

Render 函数里生成 DOM。根据传入的 type 创建了一个 node。这是一个简单的思路:
```js
container = document.getElementById("root")
​
const node = document.createElement(element.type)
node["title"] = element.props.title
​
const text = document.createTextNode("")
text["nodeValue"] = element.props.children
​
node.appendChild(text)
container.appendChild(node)
```

先根据 element 中的 type 属性创建 DOM，对每一个子节点递归做相同的处理。
```js
function render(element, container) {
  const dom = document.createElement(element.type)
​
  element.props.children.forEach(child =>
    render(child, dom)
  )
​
  container.appendChild(dom)
}
```