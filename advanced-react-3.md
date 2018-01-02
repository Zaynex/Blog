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
Context 是直接从React Data tree 中顶层数据的API。有了它，我们就不用一层一层往组件里传 props。使用过 React-Redux 的人应该都是知道，其实这个库的核心就是提供一个高阶组件`<Provider>`，用来包裹我们的业务组件，高阶组件内部通过 context 把数据提升到顶层结构。因此所有组件通过 context 便可获得整个树的数据（这一部分的操作就藏在 redux 的 connect 里)
