# 为什么是 React Hook ，我们又是如何走到这一步的？

原文：[https://medium.freecodecamp.org/why-react-hooks-and-how-did-we-even-get-here-aa5ed5dc96af](https://medium.freecodecamp.org/why-react-hooks-and-how-did-we-even-get-here-aa5ed5dc96af)

对原文有删减。

`Hooks` 从 `mixin`、高阶组件和 `render props` 的权衡中吸收了经验，为我们带来了创建可包含、可组合的行为的方法。它以一种扁平化、声明式的方式使用。

然而， 使用 `hooks` 本身也得付出一些代价。它并不是万能药，有时候我们需要权衡。


为了理解为什么 `Hooks` 很棒，我希望本文能窥探 `Hooks` 是如何解决 React 历程中的一些问题。

这里有一种场景。你必须要显示用户当前鼠标点击的位置。

https://github.com/zaynex/blog/raw/master/react/2019-04-02/assets/1.png

**然而这里有一些地方恶心到了我：**

- 如果你需要这些鼠标移动的行为在其他组件上复用，你不得不重新写一样的代码。
- 如果你想加更多类似的行为，那么之后第一眼上去的时候就会变得越来越难以理解。这是因为我们的行为逻辑都分布在 componentDidMount 和 componentWillUnmount 里。

然而我们是工程师，并且又好多工具可以帮助我们打破这种模式。我们回顾下过去的一些做法以及他们的优缺点。

### Mixins

Mixins 遭到了很多批评。他们把生命周期的hooks 聚集在一起去描述一种 副作用。
https://github.com/zaynex/blog/raw/master/react/2019-04-02/assets/2.png

虽然封装逻辑的思想很棒，但我们最终还是从 mixin 中吸取了一些教训。

我们并不能直观得知道 `this.state.x` 是从哪里来的。 在 `mixins` 中，它也可能盲目得依赖组件中存在的属性。

当我们还是使用大量 `mixins` 时就会带来非常严重的问题。你不能简单的搜索一个文件，然后假设你没有其他地方破坏逻辑。

重构它也很简单。这些 `mixed-in` 的行为需要更明显，它们不属于任何组件，也就是不能使用组件中的数据。

### 高阶组件

我们可以实现类似的效果，通过创建一个 `container` 来传递这些 `props` 使它变得不那么神奇。继承的主要代价就是增加了重构的难度，因此我们需要尝试 `composition`（组合）

https://github.com/zaynex/blog/raw/master/react/2019-04-02/assets/3.png

当这样的代码越多，我们也越来越走向正确的道路。我们拥有所有 `mixins` 的优势。现在我们有 `<MouseRender>` 组件，它不再与 订阅的行为紧密耦合。

但如果我们想要 render 不同的东西呢？我们真的需要新建一个组件么？

### Render Props & Children as a Functioneh

这种模式常常让我感到眼前一亮。

我们想要的是一个组件能够处理 mouse move 行为，并且有能力渲染任何我们想要的东西。

https://github.com/zaynex/blog/raw/master/react/2019-04-02/assets/4.png

这些细微的差别带来一些很赞的好处

我们能很清楚的知道谁提供了 `x` 和 `y`。你可以很方便得重命名这些属性避免冲突。

我们可以灵活的控制 `rendering` 的东西。我们不需要新建一个组件，如果有需要，只要 copy 一份代码就行。

你可以可以在组件渲染功能中查看所有内容。它很明显，很容易让开发者识别， `cmd+f` 就可以查看。

这种模式的主要问题是，你的组件必须在 `render` 中嵌套组件。一旦你开始嵌套多个 `component`，你就很难直观地解释到底发生了什么。

另外，它还造成了层级感的错觉。仅仅是因为组件在其他组件下嵌套并不意味着他们依赖父组件的行为。

https://github.com/zaynex/blog/raw/master/react/2019-04-02/assets/5.png

如果这里有一种方式能够拥有所有能力，并且是通过扁平化的方式就好了！

### Hooks

如果我们想要移除嵌套，并且把副作用都移到了顶部该怎么办？这样我们JSX 中唯一的render function 就是纯的渲染逻辑。

https://github.com/zaynex/blog/raw/master/react/2019-04-02/assets/6.png

这就是我想要的。

- 将副作用包含在了自己的包中， `useEffect` 还避免了三个不同生命周期钩子（`DidMount`、`DidUpdate`、`WillUnmount`）的分散。

- 组件获取数据的位置非常清晰，它整齐坐落在 render function 中。

- 不论我想加入多少东西，我的代码都不会嵌套。

#### 然而，这里也有些陷阱

当我们开始使用 `hooks`，你必须要记住以下的规则，它乍一看很奇怪。

##### 你应该在 render function 的顶层调用 `hooks`。

这意味着没有条件 `hooks` 。我们和 react 的约定是我们都会以相同的顺序调用相同数量的钩子。

当我们和 mixins 以及 Hocs 的工作原理相比较的时候，这条规则就变得很有意义。你不能有条件得使用它们并且对它们重新排序。

如果你想要 条件性的副作用，你可以分离你的 `hooks` 到另外的组件，并且考虑其他的模式。

##### 你只能在 `react function components` 以及 自定义的 `hooks` 中使用 `hooks` 

我不太确定是否有任何技术上的原因导致我们没法在普通的函数中调用。但这确保了数据总是在组件中可见。

##### `Hooks` 目前没有 componentDidCatch 和 getSnapshotBeforeUpdate。。

React 团队表示他们已经在开发了。

对于 `componentDidCatch` 的用例，你可以创建一个  `Error Boundary component`, `getSnapshotBeforeUpdate` 有点棘手，不过还好我们很少用到。

### 最后的一些笔记

毫无疑问 `hooks` 改变了我们看待 react 的方式，并且改变了一些最佳实践。这些新的方式的和库的出现是令人激动的。

然而，我在过去见到这些设计模式的大肆宣传。虽然大部分成了我的工具集中有用的一部分，但他们都有一定的成本。

我仍然没有全全理解 `hooks` 的利弊，这使我感到害怕。我强烈建议你和他们一起尝试，通过实例学习。但在重构之前，应该保持冷静。

[https://medium.freecodecamp.org/why-react-hooks-and-how-did-we-even-get-here-aa5ed5dc96af](https://medium.freecodecamp.org/why-react-hooks-and-how-did-we-even-get-here-aa5ed5dc96af)