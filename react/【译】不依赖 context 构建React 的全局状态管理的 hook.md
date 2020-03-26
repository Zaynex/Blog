原文： [Steps to Develop Global State for React With Hooks Without Context](https://blog.axlight.com/posts/steps-to-develop-global-state-for-react/)

翻译：2020.03.26


## 介绍
使用 React hooks 开发非常有意思。我已经开发了几个库。其中第一个库就是用于状态管理。 它是 [react-hooks-global-state](https://github.com/dai-shi/react-hooks-global-state)，名字确实有些长...

最初的版本发布是在2018年10月。现在我学到了很多，并且 v1.0.0 已经发布了。

https://github.com/dai-shi/react-hooks-global-state

这篇文章一步一步地展示了代码的简化版本。它有助于理解该库的目的，而实际的代码用 TypeScript 会稍微有点复杂。

## 步骤一：Global variable
```js
let globalState = {
  count: 0,
  text: 'hello',
};
```

让我们有一个像上面那样的全局变量。我们在整篇文章中采用了这种结构。可以创建一个 `React hook` 来读取这个全局变量。

```js
const useGlobalState = () => {
  return globalState;
}
```

这并不算一个真正的 React hook， 因为它没有依赖 React 自有的 hooks。
现在，这并非是我们想要的，因为当全局变量改变时它不会触发 re-render 。


## 步骤二：Re-render on updates

我们需要使用 React 的 useState hook 让它可响应。

```js
const listeners = new Set();
// 译者注： 每个接入 useGlobalState 的 组件都会在内部注册一个 listener
const useGlobalState = () => {
  const [state, setState] = useState(globalState);
  useEffect(() => {
    const listener = () => {
      setState(globalState);
    };
    listeners.add(listener);
    listener(); // in case it's already changed
    return () => listeners.delete(listener); // cleanup
  }, []);
  return state;
};
```

这将允许我们在外部更新 React state。如果你想更新全局变量，你需要触发 listeners。 让我们构建一个函数来处理更新。

```js
const setGlobalState = (nextGlobalState) => {
  globalState = nextGlobalState;
  listeners.forEach(listener => listener());
};
```

有了它，我们就可以让 useGlobalState 返回一个元组，就像 useState。

```js
const useGlobalState = () => {
  const [state, setState] = useState(globalState);
  useEffect(() => {
    // ...
  }, []);
  return [state, setGlobalState];
};
```

## 步骤三：Container

通常，全局变量会直接声明在文件中。 不过我们可以把它包裹在函数作用域里，使其有更好的复用性（此处用到闭包）。

```js
const createContainer = (initialState) => {
  let globalState = initialState;
  const listeners = new Set();

  const setGlobalState = (nextGlobalState) => {
    globalState = nextGlobalState;
    listeners.forEach(listener => listener());
  };

  const useGlobalState = () => {
    const [state, setState] = useState(globalState);
    useEffect(() => {
      const listener = () => {
        setState(globalState);
      };
      listeners.add(listener);
      listener(); // in case it's already changed
      return () => listeners.delete(listener); // cleanup
    }, []);
    return [state, setGlobalState];
  };

  return {
    setGlobalState,
    useGlobalState,
  };
};
```

## 步骤四：Scoped access
虽然我们可以创建多个 containers，但通常我们会把这些数据合并到一个全局 state 中。
典型的全局状态的库都拥有获取部分数据的能力。比如，像 React Redux 有 selector 接口从全局 state 中获取特定的数据。

在此我们采用这种比较简单的方法，就是直接使用字符串的 key 值获取部分全局 state 的数据。

```js
const createContainer = (initialState) => {
  let globalState = initialState;
  const listeners = Object.fromEntries(Object.keys(initialState).map(key => [key, new Set()]));

  const setGlobalState = (key, nextValue) => {
    globalState = { ...globalState, [key]: nextValue };
    listeners[key].forEach(listener => listener());
  };

  const useGlobalState = (key) => {
    const [state, setState] = useState(globalState[key]);
    useEffect(() => {
      const listener = () => {
        setState(globalState[key]);
      };
      listeners[key].add(listener);
      listener(); // in case it's already changed
      return () => listeners[key].delete(listener); // cleanup
    }, []);
    return [state, (nextValue) => setGlobalState(key, nextValue)];
  };

  return {
    setGlobalState,
    useGlobalState,
  };
};
```

简单起见，这里就省略了使用 useCallback, 但通常建议给库加上。

## 步骤五: Functional Updates
React useState allows functional updates. Let’s implement this feature.

React 的 useState 支持传入函数更新。我们来实现它。

```js
// ...
const setGlobalState = (key, nextValue) => {
  if (typeof nextValue === 'function') {
    globalState = { ...globalState, [key]: nextValue(globalState[key]) };
  } else {
    globalState = { ...globalState, [key]: nextValue };
  }
  listeners[key].forEach(listener => listener());
};

// ...

```

## 步骤六: Reducer
对于熟悉 Redux 的小伙伴可能更喜欢用 reducer。 React hook useReducer 也具备相同的功能。

```js
const createContainer = (reducer, initialState) => {
  let globalState = initialState;
  const listeners = Object.fromEntries(Object.keys(initialState).map(key => [key, new Set()]));

  const dispatch = (action) => {
    const prevState = globalState;
    globalState = reducer(globalState, action);
    Object.keys((key) => {
      if (prevState[key] !== globalState[key]) {
        listeners[key].forEach(listener => listener());
      }
    });
  };

  // ...

  return {
    useGlobalState,
    dispatch,
  };
};

```

## 步骤六: Concurrent Mode
为了享受 Concurrent Mode 带来的好处，我们需要使用 React state 而不是外部变量。
目前的解决办法就是将 React state 指向到我们的全局 state。

这种实现方式非常得 tricky，但本质上我们是自建了 hook 来新增一个状态连接全局数据。

```js
const useGlobalStateProvider = () => {
  const [state, dispatch] = useReducer(patchedReducer, globalState);
  useEffect(() => {
    linkedDispatch = dispatch;
    // ...
  }, []);
  const prevState = useRef(state);
  Object.keys((key) => {
    if (prevState.current[key] !== state[key]) {
      // we need to pass the next value to listener
      listeners[key].forEach(listener => listener(state[key]));
    }
  });
  prevState.current = state;
  useEffect(() => {
    globalState = state;
  }, [state]);
};
```

需要使用 patchedReducer 来允许 setGlobalState 更新全局状态。 `useGlobalStateProvider` hook 应该在一个稳定的组件中使用比如根组件。

注意这并不项众所周知的技术，并且可能会有些限制。比如，实际上并不推荐在 render 的时候调用 listeners。

为了能够合理得支持 Concurrent Mode，目前 `useMutableSource` hook 已经 RFC 中被提出。
To support Concurrent Mode in a proper way, we would need core support. Currently, useMutableSource hook is proposed in this [RFC](https://github.com/bvaughn/rfcs/blob/useMutableSource/text/0000-use-mutable-source.md).

## 尾声
这就是 `react-hooks-global-state` 的核心实现。真正的代码是用 Typescript 编写，会显得有些复杂，包括从外部读取数据的 `getGlobalState`，并且支持部分的 Redux middleware 和 DevTools 的代码。

最后，我已经开发了其他库关于全局状态和 React context，以下：

https://github.com/dai-shi/reactive-react-redux

https://github.com/dai-shi/react-tracked

https://github.com/dai-shi/use-context-selector