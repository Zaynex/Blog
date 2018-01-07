### 温馨提示
具体的操作步骤见[第一篇](https://juejin.im/post/5a47ccd26fb9a045186b126c)。
[源码地址](https://github.com/Zaynex/advanced-react-patterns)。
文本只会对核心部分代码做详细讲解，希望读者能够边阅读源码以配合此文。

继续上一小节的代码。其实 React.cloneElement 组件会有一个潜在的问题。有时候会因为布局或者是动画样式等等需要给一个 React 组件添加额外的标签。
React.cloneElement传入的参数是一个 Component 类，这些 props 是显然无法传递给普通的 html 标签，因此导致程序出现意外。
比如我们把原先的代码改成下面这样
```
function App() {
  return (
    <Toggle
      onToggle={on => console.log('toggle', on)}
    >
      <Toggle.On key="on">The button is on</Toggle.On>
      <Toggle.Off key="off">The button is off</Toggle.Off>
      <div><Toggle.Button /></div>
    </Toggle>
  )
}
```
我们看看到类似如下的 warning:
```
Invalid value for prop `toggle` on <div> tag. Either remove it from the element, or pass a string or number value to keep it in the DOM. For details, see https://fb.me/react-attribute-behavior
```


本节将进入 context 来调整组件的数据传递。这也是为了下一节的高阶组件做铺垫。


### Context
Context 是直接从React Data tree 中顶层数据的API。有了它，我们就不用一层一层往组件里传 props。使用过 React-Redux 的人应该都是知道，其实这个库其中一个核心就是提供高阶组件`<Provider>`，用来包裹我们的业务组件，高阶组件内部通过 context 把数据提升到顶层结构。因此所有组件通过 context 便可获得整个树的数据（这一部分的操作就藏在 redux 的 connect 里)

### 使用 Context
```
const TOGGLE_CONTEXT = '__toggle__'
class Toggle extends React.Component {
  static childContextTypes = {
    [TOGGLE_CONTEXT]: PropTypes.object.isRequired,
  }

  state = {on: false}
  toggle = () =>
    this.setState(
      ({on}) => ({on: !on}),
      () => this.props.onToggle(this.state.on),
    )
  getChildContext() {
    return {
      [TOGGLE_CONTEXT]: {
        on: this.state.on,
        toggle: this.toggle,
      },
    }
  }
  render() {
    return <div>{this.props.children}</div>
  }
}
```

通过getChildContext方法来返回我们的 context 数据，还需要定义该组件的context 数据类型，即 childContextTypes。在子组件需要用 context 也必须要定义contextTypes, 如果未定义，该组件的 context 数据将会得到一个空对象.
getChildContext方法会在 state 或者是 props 变更的时候都被调用。为了在context中更新数据，使用 this.setState来更新本地state。
我们注意到上面的代码中，context 是和组件的 state 数据相关联的。


### 传递context给纯函数组件

```
function ToggleOn({children}, context) {
  const {on} = context[TOGGLE_CONTEXT]
  return on ? children : null
}
ToggleOn.contextTypes = {
  [TOGGLE_CONTEXT]: PropTypes.object.isRequired,
}
```

对于纯函数的组件，第一个参数表示接受的props，第二个参数则是 context.

### Context 的大坑
为了提高性能，一般我们都会用 PureComponent。它的好处就是自带了 shouldComponentUpdate 对于下一个状态的 props 和 states 进行 shadowEqual.
由于组件产生新的 context，那么假如中间有一个父组件的 shouldComponentUpdate 返回的是 false。这会导致其子组件 不再re-render，则子组件中的所依赖的 context 是不会被更新的。这样的话，组件就失控了。

如果你想知道如何优雅地同时使用 Context 和 PureComponent，强烈建议您阅读参考资料的第一篇文章。为了保持该项目的连贯，先说明到此。

### 总结
本文只是简单得介绍了 Context。虽然在实际开发中我们并不太常用，这并不代表它不好用，而是你需要明确的是 context 会有哪些隐患，又能在实际开发中带来哪些好处。
下一节，我们将基于 context 制作一个高阶组件。


### 参考资料
1. https://medium.com/@mweststrate/how-to-safely-use-react-context-b7e343eff076
2. https://doc.react-china.org/docs/context.html#%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%A6%81%E4%BD%BF%E7%94%A8context
3. https://reactjs.org/docs/context.html#why-not-to-use-context
4. http://zhaozhiming.github.io/blog/2017/02/19/how-to-safely-use-react-context-zh-cn/














