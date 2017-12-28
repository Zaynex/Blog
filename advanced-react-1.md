## 介绍
本文主要是为了学习最近比较热门的`advanced-react-pattern`，希望借助该项目，学习一些比较使用的 React 方法，让项目精简易读。

## 工具
1. http-server
2. React-dev-tool


http-server 可以帮助我们在本地快速启动server，可以理解为静态资源服务器。
React-dev-tool 便于我们在浏览器中查看React的结构，追踪React 数据变更。


## 第一篇
1. 如果只是纯UI渲染并且是无状态的组件，可以直接采用 function 创建组件类。
```
function App() {
  return (
    <Toggle
      onToggle={on => console.log('toggle', on)}
    />
  )
}
```

2. `defaultProps` 和 `propTypes` 推荐写在组件顶层结构。（不过有些情况下，会把定义的 propTypes 存放到指定的定义文件里）

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

3. this.setState 的参数
this.setState 除了可以传入对象来更改 state 以外，还可以传入函数。具体的使用请参照官方文档，简单介绍如下
```
this.setState( (prevState) => { /* setState */},  () => {/*在this.setState 渲染新的 state后，这个回调函数便会调用*/})
```
第一个回调函数的好处是当我们想要基于当前 state 修改下一个 state 时，可以通过第一个回调函数的参数确保你拿到的是当前的state。






























