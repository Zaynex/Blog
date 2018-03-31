### 前言
之前在团队做分享的时候分介绍了 `hyperapp`， 一款类似于 React + Redux 集成的 迷你框架。
但是在讲解`Virtual DOM`算法时，并没有把 react 中类似 Key 的概念整明白。于是又自己认真回顾将其中的模棱两可的地方整理了下。

在此之前，就已经有非常多的文章介绍，其中热度比较高的就是这篇`https://github.com/livoras/blog/issues/13`
大家可以了解下 Virtual DOM 存在的意义。

为什么还会有这篇文章？本文章所有代码都是来自 `hyperapp` 这个库。因为它的实现足够精简，并且实现了 react 中 key 的理念，对于初学者的理解帮助非常大。Virtual DOM 只是本文讲解的重点，读者依然可以从这个库中了解更多（比如生命周期的实现）。


### 准备工作
我们在书写 React 组件时用到的便是 JSX 的写法。

```js
const Container = () => <div>Container</div>
const App = () => <div><Container/>App</div>
```

使用`babel plugin` 的 `transform-react-jsx` 会将上述代码转换为
```js
var Container = function Container() {
  return React.createElement(
    'div',
    null,
    'Container'
  );
};
var App = function App() {
  return React.createElement(
    'div',
    null,
    React.createElement(Container, null),
    'App'
  );
};
```

简言之，在React 中构建 Virtual DOM 的秘诀就在 React.createElement.

我们通过配置 `parmga` 让 React.createElement 使用我们自己定义的 `h` 函数去实现。
```js
"babel": {
    "presets": "env",
    "plugins": [
      [
        "transform-react-jsx",
        {
          "pragma": "h"
        }
      ]
    ]
  },
```

### 构建 Virtual DOM
Virtual DOM 的本质就是 JavaScript Object。 其中维护的对象形式大概是这个样子
```js
{
  "nodeName": "div",
  "attributes": {},
  "children": [
    {
      "nodeName": "div",
      "attributes": {},
      "children": [
        "Container"
      ],
      "key": null
    },
    "App"
  ],
  "key": null
}
```

我们将 h 替换成 React.createElement

```js
function h(name, attributes) {
  var rest = []
  var children = []
  var length = arguments.length

  while (length-- > 2) rest.push(arguments[length])

  while (rest.length) {
    var node = rest.pop()
    // note 对应的就是 cdhilren 的 [] 内的数组
    // 由于我们从数组的末端开始取节点，所以最先渲染的节点应该要被放到数组的末尾从而 pop 出正确的 DOM
    if (node && node.pop) {
      for (length = node.length; length--; ) {
        rest.push(node[length])
      }
    } else if (node != null && node !== true && node !== false) {
      children.push(node)
    }
  }

  return typeof name === "function" // 如果是一个函数，比如 Container
    ? name(attributes || {}, children)
    : {
        nodeName: name,
        attributes: attributes || {},
        children: children,
        key: attributes && attributes.key
      }
}
```

### Diff Virtual DOM
对比过程
1. 是否全等？
2. 是否为第一次渲染或者节点名称不一致？ 即不存在 old Virtual DOM，直接删除和ceateElement
3. 只是为文本节点？
4. 节点名称一致，只需 updateElement 以及继续 patch 子代节点


#### key 的真实作用

key 值不是当前自身节点所有状态的一个特殊标记，它存在的意义是对于其他兄弟节点，相对其他同级元素的一个特殊标识，
比如排序/插入节点/移除节点，通过key可以起到标志作用以提高 diff 算法。


```js
function patch(parent, element, oldNode, node) {
  if(oldNode === newNode) {
    // 啥也不做
  } else if(oldNode == null || oldNode.nodeName !== node.nodeName) {
    // 这部分主要逻辑是
    // 如果原先就没有 oldNode 也就是第一次渲染，直接 创建 DOM
    // 但是如果原先已经存在旧的 virtual DOM,但不是相同的标签名，
    // 直接删掉旧的，新建 DOM.
    var newElement = createElement(node)
    parent.insertBefore(newElement, element)

    if(oldNode != null) {
      removeElement(parent, element, oldNode)
    }

    element = newElement
  } else if(oldNode.nodeName == null) {
    // 这部分是针对新旧的virtual DOM 名称都不存在的情况
    // 也就是字符串，直接在原来的 element 上改下 nodeValue 即可
    element.nodeValue = node
  } else {
    // 好了，新旧节点没太大不一样，用不着删除新建DOM元素，咱们只是更新下就可以了~

    // 排除两个 virtual DOM 完全相同的情况
    // 排除第一次创建，没有virtual DOM 的情况
    // 再排除是个文本节点的情况
    // 排除标签名不相同的情况
    // 剩下的就是标签名一样的情况了
    updateElement(
      element,
      oldNode.attributes,
      node.attributes
    )

    // 理解 Key 的实际作用
    // key 值不是当前自身节点所有状态的一个特殊标记，它存在的意义是对于其他兄弟节点，相对其他同级元素的一个特殊标识，
    // 比如排序/插入节点/移除节点，通过key可以起到标志作用以提高 diff 算法

    // 放一个所有带key的 old virtual DOM 集合
    var oldKeyed = {}
    // 放一个所有带key的 new virtual DOM 集合
    // 就是之前没出现过的都是 new key
    var newKeyed = {}
    var oldELements = []
    var oldChildren = oldNode.children
    var chilren = node.children

    // 遍历children，收集所有old Virtual DOM 中带 key 的virtual DOM
    for(var i = 0; i < oldChildren.length; i++) {
      oldELements[i] = element.childNodes[i]

      var oldKey = getKey(oldChildren[i])
      if(oldKey != null) {
        oldKeyed[oldKey] = [oldELements[i], oldChildren[i]]
      }
    }

    // oldNode 循环标记
    var i = 0

    // newNode 循环标记
    var k = 0

    // 新的 virtual DOM children 为标准开始挨个循环，去 diff 旧的 virtual DOM children
    while(k < children.length) {
      var oldKey = getKey(oldChildren[i])
      var newKey = getKey(children[k] =resolveNode(children[k]))

      // 如果遍历的第一个oldNode 的 key 在 newKeyed集合中存在
      // 我们将i++
      // 第一次进来的时候肯定不会到这 怎么解释？
      if(newKeyed[oldKey]) {
        i++
        continue
      }

      if(newKey == null || isRecycling) {
        // 如果新节点没有 key 并且旧的节点也没有 key。这个节点要 patch 了
        if(oldKey == null) {
          patch(element, oldELements[i], oldChildren[i], children[k])
          k++
        }
        i++
      } else {
        // 哟，这个新节点也有key。
        // 我看看老节点里面维护的key集合里有没有这个key
        var keyedNode = oldKeyed[newKey] || []

        // 新节点和旧节点的 key 是一样的
        if(oldKey === newKey) {
          // 为什么咱俩 key 一样，还要patch 呢？
          // key 的作用其实是在相对的兄弟节点而言的，而不是对于它本身的前后渲染
          patch(element, keyedNode[0], keyedNode[1], children[k])
          i++
        } else if(keyedNode[0]) {
          // 如果 当前的virtual DOM 的 new key 在 old Virtual DOM 中存在
          // 我们就进行简单的移位操作，把 oldKey的那个组件插入到当前的 old DOM 前
          // 然后 diff 下属性
          // 就是把新的节点插入到旧的key 的位置 要切记 key 是标记位置的
            patch(element,
            element.insertBefore(keyedNode[0], oldELements[i]),
            keyedNode[1],
            children[k]
          )
        } else {
          // 这是个新 key，之前的 virtual DOM 中没有
          patch(element, oldELements[i], null, childrend[k])
        }

        // 新 key 里面存一份
        newKey[newKey] = children[k]
        k++
      }
    }

    // 新的children 已经遍历完了，但是 oldChilren 还有，那么就是多余的节点
    // 如果没有 key，那就删掉
    while(i < oldChildren.length) {
      if(getKey(oldChildren[i]) == null) {
        removeElement(element, oldELements[i], oldChildren[i])
      }
      i++
    }

    // 在所有 oldNode 维护的 oldKed 列表里面没有在下一次render的 Node 中使用到
    // 移除节点
    for(var i in oldKeyed) {
      if(!newKey[i]) {
        removeElement(element, oldKeyed[i][0], oldKeyed[i][1])
      }
    }
  }

  return element
}
```

### 参考资料
- [hyperApp](https://github.com/hyperapp/hyperapp)