简单来说，高阶组件就是接受组件作为参数并且返回新组件的方法。
Redux 中的 connect 就是高阶组件。它接受一个组件，并返回一个新组件。

不过需要注意的是，我们不应该对传入的组件的生命周期有任何的修改，而是将这些副作用全都定义在新生成的组件里，否则原组件将变得不可控制。详细了解高阶组件的原则推荐阅读官方文档。


### 提取 context 传递逻辑

我们会发现 ToggleOn, ToggleOff 和 ToggleButton 这几个组件全是依赖 context的。能不能有一个专门将 context 转换为 props，这样传入参数的数据结构会清晰很多。

### 高阶组件
我们创建一个函数，它接受一个组件作为参数，这个函数返回一个新组件（Wrapper,其实就是一个纯函数组件，它的职责就是将原有组件接受的数据重组，并返回新的组件），新组件接收 props 和 context 作为参数。并且将两者统一作为传入组件的props。
```
function WithToogle(Component) {
	function Wrapper(props, context) {
		const toggleContext = context[TOGGLE_CONTEXT]
		return (<Component {...toggleContext} {...props}/>)
	}
	 Wrapper.contextTypes = {
    [TOGGLE_CONTEXT]: PropTypes.object.isRequired,
  }
	return Wrapper
}
```
注意，我们仍需定义 contextTypes，否则 context 的数据为空对象，这个在上一节已有介绍。


### 应用高阶组件
```
const ToggleOn = withToggle(({children, on}) => {
  return on ? children : null
})
// 或者是如下写法
class ToggleOnWithClass extends React.Component {
  render() {
    const {children, on} = this.props
    return on? children: null
  }
}
const ToggleOn = withToggle(ToggleOnWithClass)
// 以上两者的区别是前者的组件名称由于是匿名函数，在debug时显示的是 <Unknown>,而后者则显示 <ToggleOnWithClass>下节将继续介绍名称
const ToggleOff = withToggle(({children, on}) => {
  return on ? null : children
})
```

对比下上一节的结构，
```
function ToggleOn({children}, context) {
  const {on} = context[TOGGLE_CONTEXT]
  return on ? children : null
}
```
我们发现 context 传递的那部分逻辑已经交给 withToggle 里的 Wrapper 组件去处理了，这使得组件结构精简了些。

假如我们想新创建一个组件，这个组件的所有交互和原来的组件一样，我们可以借助高阶组件快速生成。
```
const MyToggle = withToggle(({on ,toggle}) => {
//这里的 on 和 toggle，其实都是 context 的数据，只是通过高阶组件转化成了 props
 return <button onClick={toggle}>
    {on ? 'on' : 'off'}
  </button>
})
```


### 小结
高阶组件本质就是创建一个新的包裹组件，该组件的职责是将一系列相似的操作抽象成的逻辑统一处理，最后 render 传入的组件。

### 参考资料
1. https://medium.com/@mweststrate/how-to-safely-use-react-context-b7e343eff076
2. https://doc.react-china.org/docs/context.html#%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%A6%81%E4%BD%BF%E7%94%A8context
3. https://reactjs.org/docs/context.html#why-not-to-use-context
4. http://zhaozhiming.github.io/blog/2017/02/19/how-to-safely-use-react-context-zh-cn/














