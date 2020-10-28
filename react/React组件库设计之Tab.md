Tabs，即选项卡，将同一层级的内容组通过选项卡分组，并支持切换展示。

常见的选项卡组件如 Material-UI 一样。

![material-ui-tab](https://github.com/Zaynex/Blog/blob/master/react/resources/material-tab.png?raw=true)

还有国产的 ant-design。

![ant-design-tab](https://github.com/Zaynex/Blog/blob/master/react/resources/ant-tab.png?raw=true)

我们将从最简单最挫的代码一步一步推到 Tabs 组件的设计，且满足常见的需求场景。

以下是全文实例代码地址，你可以配合本文使用。

[frosty-grass-gp6f5](https://codesandbox.io/s/frosty-grass-gp6f5?file=/src/Tab102.js)

## 从需求开始

假如我们接到了一个类似选项卡的页面开发，可能最简单暴力的方式是这么做。

```c
const Tab101 = () => {
  const [value, setValue] = useState(0);
  const className = (baseName, isSelected) => 
    (isSelected ? [baseName].concat('selected') : [baseName]).join(' ');
  return <div className="tabs">
    <div className="tablist">
      <div className={className('tab-list-item', value === 0)} onClick={() => setValue(0)}>tab0</div>
      <div className={className('tab-list-item', value === 1)} onClick={() => setValue(1)}>tab1</div>
      <div className={className('tab-list-item', value === 2)} onClick={() => setValue(2)}>tab2</div>
    </div>
    <div className="tabpanels">
    {value === 0 && <div className="tab-panel-item">TabContent 0</div>}
    {value === 1 && <div className="tab-panel-item">TabContent 1</div>}
    {value === 2 && <div className="tab-panel-item">TabContent 2</div>}
    </div>
  </div>
}
```

它看起来真的很“粗暴”，首先，我们在组件内部新建了选项卡的初始状态为 `0`，我们给每个选项卡的切换添加了显式赋予了 `index` 值，并且根据点击事件切换选中。选项卡的卡片内容则通过 `index` 来判断是否渲染。

这虽然实现了 `Tab` 组件的交互，但却没有任何可以复用的迹象。而且，假如设计稿中的 `Tab` 列表项特别多，我们需要显示的输入判断。

```c
// tab-list-item
<div className={className('tab-list-item', value === 3)} onClick={() => setValue(3)}>tab3</div>
<div className={className('tab-list-item', value === 4)} onClick={() => setValue(4)}>tab4</div>
// ...
<div className={className('tab-list-item', value === 9)} onClick={() => setValue(9)}>tab9</div>

// tabpanel
{value === 0 && <div className="tab-panel-item">TabContent 0</div>}
{value === 1 && <div className="tab-panel-item">TabContent 1</div>}
{value === 2 && <div className="tab-panel-item">TabContent 2</div>}
// {...}
{value === 8 && <div className="tab-panel-item">TabContent 8</div>}
{value === 9 && <div className="tab-panel-item">TabContent 9</div>}
```

首先，我们希望组件能够自动生成 `index`，并且处理 `click` 事件。

此外，希望屏蔽 `Tab` 的操作细节，隐藏 `click` 交互，并且帮助我们维护好 `index`，至少看起来应该是这样的。

```c
<TabList>

	<Tab>tab0</Tab>
	
	<Tab>tab1</Tab>
	
	<Tab>tab2</Tab>
		
</TabList>
	
<TabPanels>

	<TabPanel>TabContent 0</TabPanel>
	
	<TabPanel>TabContent 1</TabPanel>
	
	<TabPanel>TabContent 2</TabPanel>
	
</TabPanels>
```

看起来还不错，我们提供了 `TabList` 组件包裹每项 `Tab`，`Tab` 组件在内部接受 `click` 事件来完成选项卡切换交互。`TabPanels` 也包裹每项 `TabPanel`，`TabPanel` 内部决定渲染哪个选项卡。

但还少了点东西，数据源从哪里来？

我们在组件的顶层通过 `useState` 初始化选项卡的默认状态和数据交互的方法。

```c
const [value, setValue] = useState(0);
```

这些状态也应该交由组件库来实现，开发者只需接入相应的组件即可。（当然，如果开发者想要自定义事件交互的话，我们也应该预留这样的能力）

所以，我们还需要一个 `Provider`，一方面用来初始化选项卡数据，另一个方面用来做状态管理，刚才提到的那 4 个组件都可以通过  `Provider` 获得选项卡此时的状态。

最终的组件应该是这样：

```c
<Tabs defaultIndex={0}> // -> Prodiver，提供选项卡状态初始化、选项卡状态管理、数据传递以及 tabs 的实体组件
  <TabList>             // -> 渲染 tablist 的实体组件
    <Tab>Tab0</Tab>     // -> 用于处理点击事件以及渲染 tab-list-item 的实体组件
    <Tab>Tab1</Tab>    
    <Tab>Tab2</Tab>
  </TabList> 
  <TabPanels>           // -> 渲染 tabpanels 的组件
    <TanPanel>TabContent 0</TabPanel>  // -> 根据切换值进行决定是否渲染的 tab-panel-item 组件
		<TanPanel>TabContent 1</TabPanel>
		<TanPanel>TabContent 2</TabPanel>
  </TabPanels>
</Tabs>
```

我们先从 `Tabs` 开始。根据上面的描述，`Tabs` 应该是一个 `Provider` 组件，提供选项卡状态初始化、选项卡状态管理以及数据传递。

```c
const TabsContext = React.createContext({});

const Tabs = ({children, defaultIndex}) => {
  const [selectedIndex, setSelectedIndex] = 
    useState(typeof defaultIndex === 'number' ? defaultIndex : 0);
  const context = useMemo(() => ({selectedIndex, setSelectedIndex}), [selectedIndex, setSelectedIndex]);
  return <div className="tabs"><TabsContext.Provider value={context}>{children}</TabsContext.Provider></div>
}
```

有了 `Tabs` 之后，子组件都可以通过 `useContext(TabsContext)` 获得 `context` 值。

接下来是，`TabList` 和 `Tab` 组件。

我们借助 `React.Children.map` 来获取每个组件默认的 index 值。通过 `useContext` 获取当前已经选中的 index 以及切换选项卡的方法。

```c
const TabList = ({children}) => {
  const context = useContext(TabsContext);
  return <div className="tablist">{React.Children.map(children, (child, index) => {
    return React.cloneElement(child, {
      onClick: () => context.setSelectedIndex(index),
      isSelected: index === context.selectedIndex
    })
  })}</div>
}

const Tab = ({ onClick, children, isSelected }) => {
  const className = getClassName("tab-list-item", isSelected);
  return (
    <div className={className} onClick={onClick}>
      {children}
    </div>
  );
};
```

对于，`TabPanels` 以及 `TabPanel` 组件也如法炮制。

```c
const TabPanels = ({ children }) => {
  const context = useContext(TabsContext);

  return (
    <div className="tabpanels">
      {React.Children.map(children, (child, index) => {
        return React.cloneElement(child, {
          isSelected: index === context.selectedIndex
        });
      })}
    </div>
  );
};

const TabPanel = ({ children, isSelected }) => {
  if (!isSelected) return null;
  return <div className="tab-panel-item">{children}</div>;
};
```

我们来看最终的代码

```jsx
export const Tab102 = () => {
  return (
    <Tabs defaultIndex={1}>
      <TabList>
        <Tab>tab0</Tab>
        <Tab>tab1</Tab>
        <Tab>tab2</Tab>
      </TabList>
      <TabPanels>
        <TabPanel>TabContent 0</TabPanel>
        <TabPanel>TabContent 1</TabPanel>
        <TabPanel>TabContent 2</TabPanel>
      </TabPanels>
    </Tabs>
  );
};
```

目前我们已经提炼出了可复用的 Tabs 组件，至少是初具规模。接下来再实现更多的特性。

### 自定义 Tab 组件

```jsx
export const Tab102 = () => {
  const CoolTab = React.forwardRef((props, ref) => {
    // `isSelected` is passed to all children of `TabList`.
    return (
      <Tab ref={ref} isSelected={props.isSelected} {...props}>
        {props.isSelected ? "😎" : "😐"}
        {props.children}
      </Tab>
    );
  });
  return (
    <Tabs defaultIndex={1}>
      <TabList>
        <Tab>tab0</Tab>
        <Tab>tab1</Tab>
        <CoolTab>tab2</CoolTab>
      </TabList>
      <TabPanels>
        <TabPanel>TabContent 0</TabPanel>
        <TabPanel>TabContent 1</TabPanel>
        <TabPanel>TabContent 2</TabPanel>
      </TabPanels>
    </Tabs>
  );
};
```

### forceRender

当为 `true` 时支持选项卡隐藏时依然渲染 DOM 结构。

我们需要给 Tabs 的 Provider 增加 forceRender 参数。

```jsx
const Tabs = ({ children, defaultIndex, forceRender = false }) => {
	// ...
  const context = useMemo(() => ({ selectedIndex, setSelectedIndex, forceRender }), [
    selectedIndex,
    setSelectedIndex,
    forceRender
  ]);
  return (
    <TabsContext.Provider value={context}><div className="tabs">{children}</div></TabsContext.Provider>
  );
};
```

TabPanel 通过 context 获取 forceRender。

```jsx
const TabPanel = ({ children, isSelected }) => {
  const { forceRender } = useContext(TabsContext);
  if (!isSelected && !forceRender) return null;
  const showOrHidden = !isSelected && forceRender ?  {display: 'none'} : {}
  return <div className="tab-panel-item" style={showOrHidden}>{children}</div>;
};
```

### 禁用某项 Tab

```jsx
const Tab = ({ onClick, children, isSelected, diasbled = false}) => {
  const className = getClassName("tab-list-item", isSelected);
  const style = diasbled  ? { color: '#ccc'} : {}
  return (
    <div className={className} style={style}onClick={ !diasbled && onClick}>
      {children}
    </div>
  );
};
```

截止目前，我们已经实现了常见的特性，支持 forceRender，禁用 tab 以及自定义 Tab 组件。

## Ant-design 的实现方式

Ant-design 的 Tabs 实现应该借鉴了 Semantic-UI。我们可以看它提供的组件。

```jsx
import { Tabs } from 'antd';

const { TabPane } = Tabs;

const Demo = () => (
  <Tabs defaultActiveKey="1" onChange={callback}>
    <TabPane tab="Tab 1" key="1">
      Content of Tab Pane 1
    </TabPane>
    <TabPane tab="Tab 2" key="2">
      Content of Tab Pane 2
    </TabPane>
    <TabPane tab="Tab 3" key="3">
      Content of Tab Pane 3
    </TabPane>
  </Tabs>
);
```

Ant-design 中 Tabs 组件暴露的 API 只有 2个。但它是通过配置参数来实现自定义组件。

TabPane 的用法和 TabPanel 类似，但它的 tab 表示的是我们之前设计的 Tab 组件，也就是选项卡的导航条。这既是它的优点，也是它的软肋。

通过 props 的方式调整 tab 而少一次组件的 import，对使用者来说会觉得很方便。 tab 参数支持传入 ReactNode 或者 string。但是如果想实现自定义的 Tab 组件，可能没法满足需求。比如上文的自

定义 Tab 选中后的状态。

此外，因为它简化的结构，JSX 的结构和实际的 DOM 结构会不太一样。（实际生成的 DOM 结构和我们一样）

统一和自由里如何考量平衡点，这是每个组件库里自身的设计哲学。Ant-design 针对的是企业级产品，它已经尽量通过统一的设计和更多合理的参数为使用者提供自定义机制的同时又提升开发效率。

Tab 选项判定机制。我们的实现是通过 index 进行匹配。而 Ant-design 需要用户手动设置 key。这个 key 会牵扯到很多 Tabs 组件设计的细节（以及我们当前组件设计的缺陷）。

**通过 index 实现有什么问题？**

只要 TabList 和 TabPanels 中的 children 数据等量，那么 TabList 中的 index 和 TabPanels 的 index 就能始终匹配。

当然，它里面也有一种隐性的成本——**就是我们的 Tab 和 TabPanel 展示的顺序必须要一致，这样才能确保切换时显示正确的内容。**

可一旦组件之间的 index 不匹配了，那很有可能就会出现异常状态。比如我们在 TabList 中插入一项非 Tab 的元素，它也能正常工作。

```jsx
<Tabs defaultIndex={1}>
  <TabList>
    <div>Tab0</div>
    <Tab>tab0</Tab>
    <Tab>tab1</Tab>
  </TabList>
  <TabPanels>
    <TabPanel>TabContent 0</TabPanel>
    <TabPanel>TabContent 1</TabPanel>
    <TabPanel>TabContent 2</TabPanel>
  </TabPanels>
</Tabs>
```

**既然我们不需要 Tab 组件都能实现选中，那为什么我们还需要 Tab ? 不得不承认我们的实现太 naive。**

而且我们的预期，应该是第一个 Tab 匹配第一个 TabPanel 才对。

在 reactjs 官方旗下的 [react-tabs](https://github.com/reactjs/react-tabs#custom-components) 的的解决方案是通过给每个组件添加特定的属性 `tabsRoles` 来标记这些组件是带有 Tabs 相关的特殊逻辑。

我们已经知道 ant-design 中 Tab 的 key 到底是用来做什么的。那如果要按照 index  匹配的方式，我们该如何实现呢？其大致的实现方式（读者也可以自己思考实现方式），是保持对这些 Tab 元素的 ref。这样我们就能精确得知有哪些 Tab 组件以及 TabPanel 组件。不过听上去这种实现方式对内存占用比较高，我想这大概也是 ant-design 用 key 来匹配的原因吧？

### Material-UI 的实现

准确来说，Material-UI 只实现了 Tabs 的导航 TabList 部分，却没有把切换展示对应的 TabPanel 细节屏蔽掉，抛开了使用者。

它非常得轻量，因为做得事情比较少，所以我们自定义的部分就比较多（弥补切换展示的逻辑）。

但它的实现于我而言是启蒙参考。

## 总结

谈到总结前，本文其实还少了两点分析，一是性能，而是如何支持自定义样式。在性能方面刚才的 tab 匹配只是其中一点，而对于 tab 本身，由于 ant-design 传递的 tabs 属性是静态的，也是一定的优化，这其中就需要在 业务需求与性能间平衡组件的设计。当实现核心逻辑之后，我们或许考虑如何将样式和逻辑拆分以便支持自定义样式。目前只有水平布局的 Tab，如果有垂直布局 Tab 的需求，我们是否应该把这套布局方案写在组件内呢？

目前最新的一些组件已经基于 hooks 进行样式和逻辑的分离，让我们拭目以待。

## 参考资料

[reactjs/react-tabs](https://github.com/reactjs/react-tabs#custom-components)

[标签页 Tabs - Ant Design](https://ant.design/components/tabs-cn/)

[Tabs React component - Material-UI](https://material-ui.com/components/tabs/)

[Build accessible React apps & websites.css-1ocq770{color:#319795;} with speed](https://chakra-ui.com/)