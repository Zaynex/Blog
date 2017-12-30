## Advanced-react-pattern-2

### Static 妙用
```
function App() {
  return (
    <Toggle
      onToggle={on => console.log('toggle', on)}
    >
      <Toggle.On>The button is on</Toggle.On>
      <Toggle.Off>The button is off</Toggle.Off>
      <Toggle.Button />
    </Toggle>
  )
}
```

首先外层是一个 Toggle组件，包括着内部的子组件。因为外层组件包裹的命名比较清晰。

注意这中间有不太常见的组件使用方式。
```
<Toggle.On>
```


Toggle组件是一个 component 类, On 是它的属性，不过既然作为组件使用，说明也是一个 component 类。
继续研究下 Toggle 组件的实现，去寻找答案。

```
class Toggle extends React.Component {
  static On = ToggleOn
  static Off = ToggleOff
  static Button = ToggleButton
  static defaultProps = {onToggle: () => {}}
  state = {on: false}
  toggle = () =>
    this.setState(
      ({on}) => ({on: !on}),
      () => this.props.onToggle(this.state.on),
    )
  render() {
    const children = React.Children.map(
      this.props.children,
      child =>
        React.cloneElement(child, {
          on: this.state.on,
          toggle: this.toggle,
        }),
    )
    return <div>{children}</div>
  }
}

```
在实际 debug 的时候组件渲染民时最终还是被替换成 <ToggleOn> 以及 <ToggleOff>。

static 是 ES6 中 Class 定义静态方法的关键字。静态方法的好处是在基于类创建新的实例时，新的实例是不会继承静态方法的。（简单来说，不是在原型链上定义的，而是直接挂载在这个类的属性中）

这么一看 Static 里面的静态属性引入了外部定义的类(如 ToggleOn)

```
function ToggleOn({on, children}) {
  return on ? children : null
}
function ToggleOff({on, children}) {
  return on ? null : children
}
function ToggleButton({on, toggle, ...props}) {
  return (
    <Switch on={on} onClick={toggle} {...props} />
  )
}
```
通过 Static 关联组件，是有原因的。

### 使用 React.cloneElement 减少子组件相同props传递
在 <Toggle.On> 等类似的组件中，我们发现 render 中并没有直接传入 props ，而ToggleOn 组件就是接受了props，第一个对象为 on, 第二个则是包裹的 children。

代码产生的实际效果应该是这样的。
```
render() {
	<Toggle.On on={this.state.on}>The button is on</Toggle.On>
}
```

我们再把思路回溯到 Toggle 的 render 这一方法。
```
render() {
  const children = React.Children.map(
    this.props.children,
    child =>
      React.cloneElement(child, {
        on: this.state.on,
        toggle: this.toggle,
      }),
  )
  return <div>{children}</div>
}
```

#### React.cloneElement 
React.cloneElement 接受一个组件，以及可选的props以及children。它的作用就是 clone 一份原先组件并将其替换成新组件。原先组件的 props 会被新传入的 props 浅合并。而新的children 会替换成原来的 children, 不过组件原先的 key 和 ref 会得到保留。
```
React.cloneElement(
  element,
  [props],
  [...children]
)
```

它的实际作用类似于
```
<element.type {...element.props} {...props}>{children}</element.type>
```

由于 `this.props.chidlren` 的数据类型是不确定的，可能是个 Object，也可能是 Array.官方文档推荐使用 React.Chidlren 为 children 提供特定的数据操作。

### 很有可能编写的代码结构
如果是我自己写的话，很有可能是以下结构，并且我相信大部分人一上来也是先写了这段代码。
```
class Toggle extends React.Component {
  static defaultProps = {onToggle: () => {}}
  state = {on: false}
  toggle = () =>
    this.setState(
      ({on}) => ({on: !on}),
      () => this.props.onToggle(this.state.on),
    )
  render() {
    const { on } = this.state
    return (<div>
    <ToggleOn on={on}>The button is on</ToggleOn>
    <ToggleOff on={on}>The button is off</ToggleOff>
    <ToggleButton toggle={this.props.onToggle} />
    </div>
    )
  }
}

function App() {
  return (
  <Toggle onToggle={on => console.log('toggle', on)} />
  )
}

```
我们再对比下，你就会发现，通过 cloneElemnt 确实可以让我们的结构更加清晰。 而且，还可以少一个包裹的 div 标签。虽然在 React16.2中出现了 `<><>` 的Fragment 表示方法可以避免此类问题，但是假设如果还有类似的10个组件需要接受相同的 props，我们要写10遍 `on={on}`,我们应该尽量避免重复代码。

### 分析
如果有批量的子组件需要传递相同的props,那么使用 cloneElemnt 再好不过了。但需要注意的是，这也会给子组件传递一些不必要的数据。比如在这里的 demo 中，ToggleOn 和 ToggleOff 只需要接受 on/off 的props 即可，传入的 toggle 方法是冗余的。我们假设它又接受了与自身所需数据无关的 其他 props, 而该 props 又频繁变更，那么每一次都会得到这些 props 的组件都会 re-render 的可能。


## 参考资料
1. https://reactjs.org/docs/react-api.html#cloneelement
2. https://doc.react-china.org/docs/react-api.html
3. https://mxstbr.blog/2017/02/react-children-deepdive/

































