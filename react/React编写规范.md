
## React 编写规范 
这是本人在 墨刀 工作时编写 React 时的一些心得。

### PureComponent 和 Component
外层的 container 推荐使用 Component 或者是 PureComponent.
children 尽量是无状态的组件，推荐使用 func 或者 PureComponent.

**如果你已经知晓一个组件必定每次都会re-render, 推荐使用 Component 而不是 PureComponent, 可以省去 shadowEqual 的对比时间**

### render 中对数据的处理尽量要小
这是之前一位实习新人的代码（简化版）
```
class A extends PureComponent {
	render() {
		return 'n,w,s,e'.split(',').map(item => <div>{item}</div>)
	}	
}
```

这段代码会导致每次组件 re-render 时都会重新 split 一次。此外，这些数据可以作为组件无关的数组，也不会在创建组件实例时被多次调用。

```
const DIRECTION = ['n', 'w', 's', 'e'] 
class A extends PureComponent {
	render() {
		return DIRECTION.map(item => <div>{item}</div>)
	}	
}
```

### 除了re-selector, 还可以在组件的props中使用
除了在类redux的库中使用 re-selector 以外，在组件传递 props 时对于一些操作成本比较大的 filter/find 等类似操作时, 可以缓存上一次的输入以保留结果。

```
// only good for full array (no holes) like arguments
function shallowEqualArguments (prevArguments, nextArguments) {
  if (!prevArguments || prevArguments.length !== nextArguments.length) return false
  for (let i = 0, length = prevArguments.length; i < length; i++) if (prevArguments[i] !== nextArguments[i]) return false
  return true
}

function immutableTransformCache (transformFunc) {
  let cacheResult
  let cacheArguments
  return function () { // drop context for immutable transform should not need <this>
    if (!shallowEqualArguments(cacheArguments, arguments)) {
      cacheResult = transformFunc.apply(null, arguments)
      cacheArguments = arguments
    }
    return cacheResult
  }
}
```

使用方法很简单，在一个函数外部包裹`immutableTransformCache`即可
```
const handleArray = arr => arr.map(item => handleItem(item))
const handleArrayFromCached = immutableTransformCache(handleArray)
const needProps = handleArrayFromCached(...some props)
```


### 当函数传入的数据较多时建议使用 object 形式
```
const fun1 = (age, name, family, address, sex, country) => {
  // handle
}

const func(obj) => {
  const {age, name, family, address, sex, country} = obj
}
```



