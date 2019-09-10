# 有道思维导图
思维导图的编辑器工具

用途：
- 工作计划
- 头脑风暴
- 会议纪要
- 读书笔记


# 有道云笔记思维导图的实现与思考

## 摘要
本次 TTT 分享思维导图项目从 0 到 1 的实现过程。分享内容包含思维导图的数据格式与架构的设计，如何配合 React + Redux  做数据渲染和状态管理，以及基于 Canvas 的图形绘制实践等。此外还将探索一些性能优化的技巧、React 的新特性，总结关于 React 配合 Redux 使用的心得体会和常见误区。

## 关键词
关键词：React Redux Canvas 单页应用 数据状态管理

## 推荐人群
推荐人群：客户端以及前端研发同学

## 提纲
### 介绍项目背景，整体功能架构 2 min

### 调研。 借助开源社区来实现 MVP。 （开源协议问题，和选择取舍问题） 3min

### 调研之后，对脑图有个基本框架的理解。 5min
    1. 数据格式的确定
    2. 框架的选型 （基于团队、社区以及未来的发展和对项目未来发展趋势的思考），immutable
    3. 渲染？ Canvas/Svg （其中我们并不明确，但可以先确定一种做着）

### 数据部分的实战思考 10min
1. 基于 immutable的思想以及配合 redux 的优势——数据回溯。
2. 数据格式新旧版本如何兼容？
3. 借助 redux middleware
4. 统一数据操作入口，方便单元测试，以及后期方便去 redux 化
5. 如何做其他数据格式的转换，便于兼容其他的文件格式（xmind等）。



### 图形绘制实现原理  10min

1. 节点计算策略
2. 如何更新脑图的节点
3. 如何绘制曲线
4. Canvas Or SVG ?
5. 加餐：借助 Canvas，还能生成高清图片

以BTree 为例，我们想要知道整体的布局，首先，我们得需要从子节点的布局。
先获取子节点的宽高，再将子节点部分作为一个整体，然后根据我们的布局策略，再决定父节点的位置。
由此递归，我们得到一个整体的脑图布局。


canvas 绘制问题。
我们的测试，在IE浏览器下，canvas 最大只能支持单边 25800 像素。超出该画布之后的线条就不再显示。
其次画布越大，canvas 的重绘成本就更高。
这也是为什么我我们后续会去尝试调研 svg 试图解决画布太大时，节点丢失的问题。


借助 Canvas，生成图片。
目前有开源的库有两个。
domtoimage
和 html2canvas

从社区活跃度上来看以及维护状况来看， 推荐 html2canvas。
但是这些类库都有一定的局限性，比如无法获取特定的字体，或者阴影或者 svg 的图标。
想要了解支持哪些css属性可以查阅文档。

http://html2canvas.hertzen.com/features/

如果对精准度要求较高，推荐服务端使用无头浏览器（puppeptter) 导出图片。
我们的 markdown 导出 PDF 就是使用 无头浏览器实现。

### 性能优化实战 10min

1. Dynamic Load
好处：减少文件体积
在`16.6`的 `react` 版本中，我们可以使用 `React.lazy` 配合 `Suspense` 进行组件的动态加载。
16.6 之前或者有需要 server render `react-loadable.`

> React.lazy takes a function that must call a dynamic import(). This must return a Promise which resolves to a module with a default export containing a React component.


我们在 toolbar 以及导图导出图片的功能中均有使用
```
// 代码
```

3. 事件代理
react 的事件代理和我们常见的事件代理还是有微妙的区分的。
对于拖拽而言，我们需要绑定多个事件来实现拖拽的事件监听
在 react 的源码中
```
onDragStart
onDragOver
onDragEnter
onDragEnd
```
react 的事件代理:
将所有事件都挂载在 document 对象上。 其实已经实现了事件代理。
唯一的区别是每次当我们在组件里挂载的事件，都会被 addEventlistener 在 document 对象上。
completeWork 之后，分析出事件类型，比如 onClick、dbClick
然后在 document 对象上监听该事件。 对应的事件类型是 通过 点击的DOM 获得 FiberNode.然后执行 FiberNode 里的方法。

4. 从深拷贝到浅拷贝
其实从深拷贝到浅拷贝，出于性能的考虑就不用多说了。
想要实现浅拷贝的方式，主要是两类：
1. 使用 immutable.js 的数据类型操作，每次生成的数据都是
我们用的是 immutability-helper 。
体积上其实都很小，但是从api过度来看，一个是 db 的操作语法，一个更贴近原生操作。
immutable 体积较大。
发现其实还是 immer 更合适。

https://www.reddit.com/r/javascript/comments/96xqnu/immer_or_immutablejs/
https://reactjs.org/docs/update.html
https://www.npmtrends.com/immer-vs-immutability-helper-vs-immutable

5. 查询缓存
memorize-one

re-selector

6. 框架是否有原罪？我就是要更新 dom。能不能别让我 diff 了。 从编辑态的优化开始
比如想要选中一个节点，回到原始的DOM 的时代，我们可以直接给这个元素添加选中样式，但是点击下一个元素时，我们需要清除上一次添加的选中的样式。这样的话，但是有时也会存在多选的状态，每次清除上一次选中然后再全选，有些不太划算。
在数据驱动视图变更的模式里，我们选中节点之后，将节点id放到一个数组里。然后在 map 函数里 render 时判断，该元素是否为选中的节点，如果是的话则添加选中的 class 样式。
```
const NodeMap = (props) => props.nodes.map(node => <Node node={node} selected={selectedNodes.includes(node.id)}>)
```
这会带来一个问题：
单纯的样式选择会引发整个 tree 的 diff。
所以我们把组件拆成两个
```
const NodeMap = (props) => props.nodes.map(node => <Node node={node}/>)
class SelectNodes extends Component {
  componentDidMount() {
    this.addSelectedNodes();
  }

  componentDidUpdate(prevProps) {
    const unUseIds = difference(prevProps.selectedNodes, this.props.selectedNodes);
    unUseIds.forEach(id => {
      const $ele = document.getElementById(id);
      $ele && $ele.classList.remove('selected');
    })
    this.addSelectedNodes();
  }

  addSelectedNodes = () => {
    this.props.selectedNodes.forEach(id => {
      const $ele = document.getElementById(id);
      $ele && $ele.classList.add('selected');
    })
  }

  return this.props.children
}
```
这样，核心的组件 render 逻辑不会受到任何影响，当选中节点时，核心的 nodeMap 不会更新。只是 SelectNodes 组件会更新。

7. PureComponent 和 Component
我们知道 pureComponent 会在 shouldComponentUpdate 里增加一个 shdowEqual
如果你想要自定义需要diff哪些对象的属性，那么推荐使用 Component。
以及如果你知道该组件，每次都会重新渲染，那就直接用 Component, 来减少shdowEuql。

8. 关于 connect
why? 在connect 里添加了数据，但是在组件中并未使用，也依然会触发组件的更新
虽然我们没有在组件的 props 里使用数据，但是它确实会应该我们的 shdowEqual.

### 心得体会 5min

1. 数据驱动是银弹么？有时候是时机（事件）驱动，动态组件（图标的显示）
数据驱动的问题，其实之前的分离选中态就是数据驱动最大的问题。很多时候，我们往往知道想要变更的特定的数据。
但由于 react virtual DOM，会对比前后两份数据来得到新的UI。减少 virtual DOM diff 是我们需要考虑的。
以节点组件的备注弹窗为例，假设我们通过数据驱动的方式，标记 内链/备注的 弹窗是否显示。如果给每个节点下面增加一个状态表示是否显示弹窗，并且在每个组件下面都添加一个弹窗组件作为UI 呈现。
我们的 node 代码只会越来越冗余，并且每次节点更新都要判断弹窗是否变更。
而且由于我们加了setTimeout的消失，容易导致弹窗会累加显示。

总结下来的问题是
- 需要维护状态（open/close）以及添加默认状态
- 需要预先在判断里写好组件,代码丑陋
- 增加 diff 成本

而脱离数据驱动ui 的桎梏，采用时机驱动的方式，
```js

showComponent = () => {
  mindController.emit('controller:render:component', {
    component: (
      <Popper open={true}
        anchorEl={anchorEl}
        placement="bottom"
        style={{ zIndex: 1000 }}>
        <RemarkComponent/>
      </Popper>
    ),
    triggerElement: anchorEl,
  });
}

hideComponent = () => {
  mindController.emit('controller:unmount:component');
}




```

```js
class NodeGlobalHover extends PureComponent {
  componentDidMount() {
    mindController.on('controller:render:component', this.handleComponentRender);
    mindController.on('controller:unmount:component', this.handleComponentUnmount);
  }
  createReactElementInDom = component => {
    this.dynamicComponent = component;
    this.container = document.createElement('div');
    document.body.append(this.container);
    ReactDOM.render(this.dynamicComponent, this.container);
  };

  handleComponentRender = ({component, triggerElement}) => {
    this.createReactElementInDom(component);
  };

  handleComponentUnmount = () => {
    if (this.container) {
      ReactDOM.unmountComponentAtNode(this.container);
      this.container.remove();
    }
  };
  render() {
    return this.props.children;
  }
}
```

其实现的本质就是利用 ReactDOM.render 挂载在一个我们动态生成的 DIV 上。

```js
ReactDOM.render(<Component />, container);
ReactDOM.unmountComponentAtNode(container);
```

这里演示的是比较简单的动态组件方法，负责一些的还可以维护一个队列，来持续新建动态组件和注销动态组件。
因为我们的弹窗只能存在一种，要么显示备注，要么显示内链，所以就采用了简单的方式。

3. 如何减少样板代码

### 想象空间 5min

1. 如何做成一个优雅的 component，如何迁移到其他库

### QA


参考资源
https://nalwaya.com/javascript/2016/05/02/react-js-best-practices.html



分片加载
```
function arrSlice(arr, num) {
    let result = []

    let chunkSize = Math.floor(arr.length / num)
    chunkSize = chunkSize ? chunkSize : 1 // 如果chunkSize为0，则手动变为1

    for(var i = 0; i < num; i ++) {
         let tmpArr = arr.slice(chunkSize * i, chunkSize * i + chunkSize)
         result.push(tmpArr)
    }
    return result
}
```









背景
第一版
缺陷
为什么第二版

实际遇到的问题。


性能优化放到后面。


1. 框架选型/开源之类的 (2-3m)



## 如何渲染成最终的脑图
先看数据结构。
计算的时候的数据， render 时的数据。
render() 时是 一个 array.
计算是嵌套的结构。

再介绍如何更新坐标信息，得到新的数据

图片 DOM 位置信息精确

再介绍图形是如何绘制的。





在实际过程中，遇到的一些问题（数据，绘制）
1. 数据的问题，提供思路 去解决
2. 新旧版本字段兼容
3. undo/redo
4. runtime check。
比如我们在上层组件里传递了 props 给子组件，但是子组件对传递了什么props 是无感知的。
我们只能通过 prop-types 来鉴别。但


性能优化


数据驱动

todo: redux






