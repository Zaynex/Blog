项目简介： 类似于百度脑图的思维导图应用。不涉及到异步请求，所有的数据变更都是同步的。

在项目初始阶段，我们没有考虑其他状态管理方案，直接采用 redux 以及 react-redux。

在重新思考项目的时候，反思了使用 redux(react-redux) 的几个问题
1. 杀鸡焉用牛刀

只是给全局的 store 新增一个数据，并且提供简单的一些数据操作。你需要
- reducer
- reducer 中的 action 需要和 dispatch action 打配合
- actionCreator(可选)
- mapStateToProps(可选)

而假设你的操作是 sideEffect 的，则需要借助 redux-thunk/redux-saga 等 middleware。

对于一个只有同步数据操作，且需要管理的状态不多的应用而言，这些概念复杂且不实用。

2. 性能问题

一、当 dispatch action 成功处理之后，redux 会通知每个 subscribes (在项目中就是包裹 connect 的组件) ，这会触发组件的变更。
因此，当我们同时 dispatch 多个 action 时，就会产生一些性能问题：组件会 re-render 多次。它缺乏 React 中 setState 的 batch 能力。不过这也有解决办法，比如在 `ReactDOM.unstable_batchedUpdates()` 中调用，或者是引入一些 `middleware` 如 `redux-batched-actions`, 它能支持 dispatch 一个 action 数组。

二、V6的性能比V5版本的要差

解决以上两个问题的办法就是升级到`React-Redux v7`，connect 已经重新设计并做了大量性能优化，而 v7 提供了新的 batch API 也可以解决第一个问题。（升到 v7 的前提是 React 的版本升级到 16.8.3及以上）。


3. 基于 Action 的方式难以直观追踪数据变更
当我们 dispatch action 时，我们会看到 prev state 和 next state 以及 payload。但我们很难明确到底改 store 中的哪个对象。如果仅仅通过 payload 去对比前后对象差异是相当低效的，所以可能会通过更具象的 action 名或者是封装了更友好的 actionCreator 名来表达。
在项目中，为了避免样板代码，我们没有采用 actionCreator 的方式。而是定义合理的 type 约束名。
假设我们有一个全局的 store
```
const initState = {
	fontSize: 16,
	fontColor: 'red',
}
function toolbar(state = initState, {action, payload} => {
	switch(action.type) {
		'toolbar:update:fontSize': {
			return {...state, {fontSize: payload}}
		}
	}
})
const rootReducer = combineReducer({
	toolbar,
	...// other reducers
})

const store = createStore(rootReducer);
```
当我们想要更新 toolbar 的数据时，我们的 dispatch 是如下
```
dispatch({type: 'toolbar:update:fontSize', payload: 14})
```
通过 type 的合理约束，即 "reducer:curd:key"，
我们就能知道到底要更改的是哪个 reducer 中的数据。
我们的这种做法相对简单，其他更为成熟的解决方案，如dva.js (基于 redux 和 redux-sage 的数据流方案)。

![dva](https://zos.alipayobjects.com/rmsportal/PPrerEAKbIoDZYr.png)

它引入了 Model 的概念，state 只是 Model 的状态数据，重点是`dva`将 action 中的 type 映射成了 reducer 中变更的具体方法。

```
// Model
app.model({
  namespace: 'count',
  state: 0,
  reducers: {
    add  (count) { return count + 1 },
    minus(count) { return count - 1 },
  },
});

// dispatch action
dispatch({type: 'count/add'})
```

## 寻找方案
在替换 redux 之前，我们有如下备选方案
1. useHook (useContext + useReducer)
2. Context API (React16.3)

第一种 hook 的方案成本比较高，需要将所有的组件都改写成 function component，暂且不深究。

第二种方式，是基于 React 的 Context API 实现。可以通过在 React 顶层组件中用 state/setState 生成并管理 Context 的数据。而且可以通过合理的分离 Context 使得组件只关注自身所需要的数据。

![context](https://miro.medium.com/max/1540/1*b6Ev2SZ8SBlqhKVrOGDZaA.jpeg)
### React.16.3 之前 Context 有什么问题？

> The problem is, if a context value provided by component changes, descendants that use that value won’t update if an intermediate parent returns false from shouldComponentUpdate. This is totally out of control of the components using context, so there’s basically no way to reliably update the context. This blog post has a good explanation of why this is a problem and how you might get around it.





### 原生的 context 实现

```
class CounterProvider extends React.Component {
  state = {
    count: 0
  };

  increment = () => {
    this.setState({ count: this.state.count + 1 });
  };

  decrement = () => {
    this.setState({ count: this.state.count - 1 });
  }

  render() {
    return (
      <CounterContext.Provider
        value={{ count: this.state.count, increment:this.increment, decrement:this.decrement }}
      >
        {this.props.children}
      </CounterContext.Provider>
    );
  }
}

export default function App() {
  return <CounterProvider>
    <Counter />
  </CounterProvider>
}

function Counter() {
  return (
    <CounterContext.Consumer>
      {counter => (
        <div>
          <h1>Count: {counter.count}</h1>
          <Button action={() => counter.increment()} buttonTitle="+" />
          <Button action={() => counter.decrement()} buttonTitle="-" />
        </div>
      )}
      </CounterContext.Consumer>
  );
}
```

借助 React 自身的 state 来管理状态的思路非常赞。然而实际会发现一些不足之处。

如果数据分离比较多，即 Context 较多的情况下，会发生多层嵌套。官方的例子给出了 2个 Context 的 demo。
```
class App extends React.Component {
  render() {
    const {signedInUser, theme} = this.props;

    // App component that provides initial context values
    return (
      <ThemeContext.Provider value={theme}>
        <UserContext.Provider value={signedInUser}>
          <Layout />
        </UserContext.Provider>
      </ThemeContext.Provider>
    );
  }
}

// A component may consume multiple contexts
function Content() {
  return (
    <ThemeContext.Consumer>
      {theme => (
        <UserContext.Consumer>
          {user => (
            <ProfilePage user={user} theme={theme} />
          )}
        </UserContext.Consumer>
      )}
    </ThemeContext.Consumer>
  );
}
```
但是如果是4个、5个甚至更多呢？


## unstated
接下来要介绍的便是 unstated。

首先看下该怎么用
```
import {Container} from 'unstated';

class CounterContainer extends Container {
  state = {
    count: 0
  };

  increment() {
    this.setState({ count: this.state.count + 1 });
  }

  decrement() {
    this.setState({ count: this.state.count - 1 });
  }
}

function Counter() {
  return (
    <Subscribe to={[CounterContainer]}>
      {counter => (
        <div>
          <h1>Count: {counter.state.count}</h1>
          <Button action={() =>counter.increment()} buttonTitle="+" />
          <Button action={() => counter.decrement()} buttonTitle="-" />
        </div>
      )}
    </Subscribe>
  );
}

function App() {
  <Provider>
    <Counter />
  </Provider>
}
```

### 优雅在何处？
1. 通过 <Subscribe to={[]}/> 
Container 组件。相对应的理解为 容器的状态组件
Componeent 理解为 视图组件
动态注入所有的 生产者（Provider）

## 后续思考
整体迁移 hook ?
自己基于 context api 做轻量的状态管理


![image.png](http://note.youdao.com/yws/res/12017/WEBRESOURCE5fb28476e0978dd7d525d627b63393a7)
(取自 youtube)

参考资料
- https://www.youtube.com/watch?v=IxDWVQfe3M8
- https://blog.bitsrc.io/build-our-own-react-redux-using-usereducer-and-usecontext-hooks-a5574b526475
- https://redux.js.org/faq/performance#how-can-i-reduce-the-number-of-store-update-events
- https://github.com/reduxjs/react-redux
- https://dvajs.com/guide/concepts.html#action
- https://stackoverflow.com/questions/39977540/can-redux-be-seen-as-a-pub-sub-or-observer-pattern
- https://stackblitz.com/edit/dva-example-count
- https://zhuanlan.zhihu.com/p/50336226
- https://medium.com/@mweststrate/how-to-safely-use-react-context-b7e343eff076