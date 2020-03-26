原文： [Steps to Develop Global State for React With Hooks Without Context](https://blog.axlight.com/posts/steps-to-develop-global-state-for-react/)

翻译：2020.03.25

## 介绍
使用 React hooks 开发非常有意思。我已经开发了几个库。其中第一个库就是用于状态管理。 它是 "react-hooks-global-state"，名字确实有些长...

最初的版本发布是在2018年10月。现在我学到了很多，并且 v1.0.0 已经发布了。

https://github.com/dai-shi/react-hooks-global-state

这篇文章一步一步地展示了代码的简化版本。它有助于理解这库的目的，而实际的代码用 TypeScript 会稍微有点复杂。

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

通常，全局变量会直接声明在文件中。 不过我们可以把它放到函数作用域里，使其有更好的复用性（此处用上闭包）。

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


We don’t go in detail about TypeScript in this post, but this form allows to annotate types of useGlobalState by inferring types of initialState.

Step 4: Scoped access
Although we can create multiple containers, usually we put several items in a global state.

Typical global state libraries have some functionality to scope only a part of the state. For example, React Redux uses selector interface to get a derived value from a global state.

We take a simpler approach here, which is to use a string key of a global state. In our example, it’s like count and text.

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
We omit the use of useCallback in this code for simplicity, but it’s generally recommended for a library.

Step 5: Functional Updates
React useState allows functional updates. Let’s implement this feature.

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
Step 6: Reducer
Those who are familiar with Redux may prefer reducer interface. React hook useReducer also has basically the same interface.

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
Step 6: Concurrent Mode
In order to get benefits from Concurrent Mode, we need to use React state instead of an external variable. The current solution to it is to link a React state to our global state.

The implementation is very tricky, but in essence we create a hook to create a state and link it.

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
The patchedReducer is required to allow setGlobalState to update global state. The useGlobalStateProvider hook should be used in a stable component such as an app root component.

Note that this is not a well-known technique, and there might be some limitations. For instance, invoking listeners in render is not actually recommended.

To support Concurrent Mode in a proper way, we would need core support. Currently, useMutableSource hook is proposed in this RFC.

Closing notes
This is mostly how react-hooks-global-state is implemented. The real code in the library is a bit more complex in TypeScript, contains getGlobalState for reading global state from outside, and has limited support for Redux middleware and DevTools.

Finally, I have developed some other libraries around global state and React context, as listed below.

https://github.com/dai-shi/reactive-react-redux
https://github.com/dai-shi/react-tracked
https://github.com/dai-shi/use-context-selector