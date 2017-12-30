## Advanced-react-pattern-1
本文主要是为了学习最近比较热门的`advanced-react-pattern`，希望借助该项目，学习一些比较实用的 React 写法，优化代码结构。

## 工具
1. http-server
2. React-dev-tool


- http-server: 可以帮助我们在本地快速启动一个server,默认端口号是 8080，可以理解为静态资源服务器。(这只是个备选项)
- react-dev-tool: 便于我们在浏览器中查看React的数据结构以及调试。

步骤:
```
git clone https://github.com/kentcdodds/advanced-react-patterns
cd advanced-react-patterns
http-server ./
```

不过这个项目可以不用server,直接打开即可。


## 第一篇
1. 如果只是纯UI渲染并且是无状态的组件，可以直接采用纯函数创建组件类。
```
function App() {
  return (
    <Toggle
      onToggle={on => console.log('toggle', on)}
    />
  )
}
```


### 初始化数据放在首部声明
`defaultProps` 和 `propTypes` 推荐写在组件顶层结构。不过有些做法是把定义的 propTypes 存放到指定的定义文件里（只用来定义数据结构）

```
class Toggle extends React.Component {
  static defaultProps = {onToggle: () => {}}
  state = {on: false}
  // 在 setState 第二个参数传入一个函数，这个函数可以确保在 state 完成之后触发
  toggle = () =>
    this.setState(
      ({on}) => ({on: !on}),
      () => {
        this.props.onToggle(this.state.on)
      },
    )
  render() {
    const {on} = this.state
    return (
      <Switch on={on} onClick={this.toggle} />
    )
  }
}
```

关于组件中方法的定义位置，如果数据比较少的情况下，可以完全不用定义 constructor。但是当数据略有些复杂的时候，也可以把一些初始化的数据放在 constructor 里定义。 例如：
```
constructor(props) {
  super(props)
  this.state = {on: false}
}
```


### 正确 setState
在 React 的 State 变更策略中，setState()并不会立即更新组件，而是可能会延迟，进行批处理一次性更新。

this.setState 除了可以传入对象来更改 state 以外，还可以传入函数。具体的使用请参照官方文档，简单介绍如下
```
this.setState(updater, [callback])
```

`updater` 函数签名如下
```
(prevState, props) => stateChange
```

updater函数的的好处是当我们想要基于当前 state 修改下一个 state 时，可以通过第一个回调函数的参数确保你拿到的是原先的state。

第二个 callback 表示当 setState() 执行完成之后并且组件被重新渲染之后调用。
不过在官方文档中，这类逻辑推荐在 `componentDidUpdate` 中使用。

### 结语
只要数据(props/state)变更 React 就会触发 Re-render。因此 React 会尽量让数据批处理只要更新一次即可。在使用 state 时你也无需担心会多次 re-render。React 帮我们做了进一步优化。如果想基于当前 state 修改下一个状态时，尽量使用 setState 中第一个回调函数可以确保你获得的是当前的 state。

### 参考资料
1. https://reactjs.org/docs/react-component.html#setstate
2. https://doc.react-china.org/docs/react-component.html#setstate
3. https://github.com/kentcdodds/advanced-react-patterns




















