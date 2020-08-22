#! https://zhuanlan.zhihu.com/p/194202131

原文:

[Records & Tuples for React](https://sebastienlorber.com/records-and-tuples-for-react)

Records 和 Tuples，是一个非常有趣的提案，目前已经在TC39 达到第二阶段（ stage2 ）。

它们为 JavaScript 带来了深度不可变的数据结构。

但不要忽略它们的平等属性，这对 React 来说非常有趣。

一系列关于 React 的 bug 都与不稳定的对象有关。

性能：本可以避免的 re-render

行为：无用的副作用再次执行，无限循环

API 表层：当一个对象的对象身份非常重要的时候 ，没法给予特定的表达

我将解释 Records 和 Tuples 的基本概念，已经它们如何解决现实世界的 React 的问题。

## Records & Tuples 101

这篇文章关于在 React 的 Records 和 Tuples。我在这里只介绍基本概念。

它们看起来就像带 # 前缀的常规对象和数组。

```jsx
const record = #{a: 1, b: 2};
record.a;
// 1
const updatedRecord = #{...record, b: 3};
// #{a: 1, b: 3};
const tuple = #[1, 5, 2, 3, 4];
tuple[1];
// 5
const filteredTuple = tuple.filter(num => num > 2)
// #[5, 3, 4];
```

默认情况下，他们是完全不可变的。

```jsx
const record = #{a: 1, b: 2};
record.b = 3;
// throws TypeError
```

它们可以被看作 “复合基本元素”，并且可以通过值进行比较。

非常重要：两个深度相等的 records 用 === 永远相等。

```jsx
{a: 1, b: [3, 4]} === {a: 1, b: [3, 4]}
// with objects => false

#{a: 1, b: #[3, 4]} === #{a: 1, b: #[3, 4]}
// with records => true
```

我们可以认为在某种程度上，Record 的身份就是它的实际值，就像常规的 JS 基本类型。

我们将会看到，这个属性对 React 有深远的影响。

它们也可以和 JSON 进行交互操作

```jsx
const record = JSON.parseImmutable('{a: 1, b: [2, 3]}');
// #{a: 1, b: #[2, 3]}

JSON.stringify(record);
// '{a: 1, b: [2, 3]}'
```

它们只能包含其他 records和 tuples 或基本类型。

```jsx
const record1 = #{
  a: {
    regular: 'object'
  },
};
// throws TypeError, because a record can't contain an object

const record2 = #{
  b: new Date(),
};
// throws TypeError, because a record can't contain a Date

const record3 = #{
  c: new MyClass(),
};
// throws TypeError, because a record can't contain a class

const record4 = #{
  d: function () {
    alert('forbidden');
  },
};
// throws TypeError, because a record can't contain a function
```

注意：通过使用符号作为WeakMap键（单独的建议），并在记录中引用这些符号，您也许可以将这样的可变值添加到记录中。

想要了解更多？ 直接阅读该[提案](https://github.com/tc39/proposal-record-tuple)，或阅读Axel Rauschmayer的[这篇文章](https://2ality.com/2020/05/records-tuples-first-look.html)。

## Records & Tuples for React

现在 React 开发人员已经习惯了 immutability.

每次以不变的方式更新某些状态时，都会创建新的对象标识。

不幸的是，这种不可变性模型在React应用中引入了一类全新的bug，以及性能问题。仅在 props 随着时间保留标识的情况下，一个组件才能正确工作，并且性能良好。

我喜欢把 Records & Tuples 当作一种方便的方式，让对象身份更加 "稳定"。

让我们用实际的用例来看看这个建议会对你的React代码产生怎样的影响。

注：有一个 Records & Tuples [Playground](https://rickbutton.github.io/record-tuple-playground/#eyJjb250ZW50IjoiaW1wb3J0IFJlYWN0IGZyb20gXCJodHRwczovL2Nkbi5za3lwYWNrLmRldi9yZWFjdFwiO1xuaW1wb3J0IFJlYWN0RE9NIGZyb20gXCJodHRwczovL2Nkbi5za3lwYWNrLmRldi9yZWFjdC1kb21cIjtcblxuXG5jb25zdCBIZWxsbyA9ICh7dXNlcn0pID0+IHtcbiAgcmV0dXJuIDxwPkhlbGxvIHt1c2VyLm5hbWV9PC9wPlxufTtcblxuY29uc3QgdXNlciA9ICN7bmFtZTogXCJTZWJhc3RpZW5cIn1cblxuUmVhY3RET00ucmVuZGVyKFxuICAgIDxIZWxsbyB1c2VyPXt1c2VyfS8+LFxuICAgIGRvY3VtZW50LmJvZHksXG4pO1xuXG4iLCJzeW50YXgiOiJoYXNoIiwiZG9tTW9kZSI6dHJ1ZX0=)，可以运行 React。

## Immutability

执行不可变可以通过递归调用 Object.freeze() 来实现。

但在实践中，我们却经常没有严格使用不可变模型，因为每次更新后使用 Object.freeze() 并不是很方便。 然而，直接改变 React state 是对 React 新手最常见的错误。

Records & Tuples 提案将强制执行不可变性，并且防止常见的状态突变错误。

```jsx
const Hello = ({ profile }) => {
  // prop mutation: throws TypeError
  profile.name = 'Sebastien updated';
  return <p>Hello {profile.name}</p>;
};
function App() {
  const [profile, setProfile] = React.useState(#{
    name: 'Sebastien',
  });
  // state mutation: throws TypeError
  profile.name = 'Sebastien updated';
  return <Hello profile={profile} />;
}
```

## Immutable updates

有许多种方式在 React 中实现 immutable state update： vanilla JS, Lodash Set, ImmerJS, ImmutableJS...

Records & Tuples 支持与 ES6 对象和数组相同的 immutable update

```jsx
const initialState = #{
  user: #{
    firstName: "Sebastien",
    lastName: "Lorber"
  }
  company: #{
    name: "Lambda Scale",
  }
};
const updatedState = {
  ...initialState,
  company: {
    ...initialState.company,
    name: 'Freelance',
  },
};
```

截止目前，由于 immerJS 已经赢得了 immutable updates 之战，因为它处理嵌套属性的简单方式以及与常规 JS 代码的互操作性。 

目前还不清楚 Immer 准备如何与Records & Tuples 合作，但这是提案作者正在探索的事情。

Michael Weststrate自己也强调，一个单独的提案（[proposal-deep-path-properties-for-record](https://github.com/tc39/proposal-deep-path-properties-for-record)）可能由于 Records 和Tuples的存在而让 ImmerJS 变得不再必要。

```jsx
// with Records & Tuples
const initialState = #{
  counters: #[
    #{ name: "Counter 1", value: 1 },
    #{ name: "Counter 2", value: 0 },
    #{ name: "Counter 3", value: 123 },
  ],
  metadata: #{
    lastUpdate: 1584382969000,
  },
};
// Vanilla JS updates
// using deep-path-properties-for-record proposal
const updatedState = #{
  ...initialState,
  counters[0].value: 2,
  counters[1].value: 1,
  metadata.lastUpdate: 1584383011300,
};

// with immer
const state1 = {
	counters: [
		{name: 'Counter 1', value: 1},
		{name: 'Counter 2', value: 0},
		{name: 'Counter 3', value: 123},
	]
}
const state2 = Immer.produce(state1, draft => {
	draft.counter[0].value = 2;
	draft.counter[1].value = 1;
	draft.metadata.lastUpdate = 1584383011300
})
```

## useMemo

除了记住昂贵的操作以外，useMemo() 在避免创建新的对象标识上也很有用，它可以避免触发无用的计算，重复渲染或树中更深层的副作用执行。

让我们考虑以下用例：你有一个带有多个过滤器的UI，并想从后端获取一些数据。

现有的React代码库可能包含这样的代码

```jsx
// Don't change apiFilters object identity,
// unless one of the filter changes
// Not doing this is likely to trigger a new fetch
// on each render
const apiFilters = useMemo(
  () => ({ userFilter, companyFilter }),
  [userFilter, companyFilter],
);

const { apiData, loading } = useApiData(apiFilters);
```

有了 Record & Tuples，它可以简化成：

```jsx
const {apiData,loading} = useApiData(#{ userFilter, companyFilter })
```

## useEffect

我们继续使用 Api filters 的例子。

```jsx
const apiFilters = { userFilter, companyFilter };

useEffect(() => {
  fetchApiData(apiFilters).then(setApiDataInState);
}, [apiFilters]);
```

不幸的是，这个 fetch 的副作用会重新执行，因为每次这个组件时 apiFilters 对象的身份都会发生改变，setApiDataInState 会触发重新渲染，你最终会陷入一个 fetch/render 的无限循环。

这个错误在 React 开发者中如此常见以至于 Google search 里上千条 useEffect + "infineite loop"(无限循环）

Kent C Dodds 为此还写了一个[工具](https://github.com/kentcdodds/stop-runaway-react-effects)可以在开发环境里中断无限循环。

非常常见的解决方案：直接在 useEffect 的回调中创建apiFilters。

```jsx
useEffect(() => {
  const apiFilters = { userFilter, companyFilter };
  fetchApiData(apiFilters).then(setApiDataInState);
}, [userFilter, companyFilter]);
```

另一个创造性的解决方案（性能不高，在Twitter上发现的）。

```jsx
const apiFiltersString = JSON.stringify({
  userFilter,
  companyFilter,
});

useEffect(() => {
  fetchApiData(JSON.parse(apiFiltersString)).then(
    setApiDataInState,
  );
}, [apiFiltersString]);
```

我最喜欢的方案是：

```jsx
// We already saw this somewhere, right? :p
const apiFilters = useMemo(
  () => ({ userFilter, companyFilter }),
  [userFilter, companyFilter],
);
useEffect(() => {
  fetchApiData(apiFilters).then(setApiDataInState);
}, [apiFilters]);
```

有很多花哨的方法可以解决这个问题，但随着滤镜数量的增加，它们都会变得很烦人。

use-deep-compare-effect(来自Kent C Dodds)可能是比较不烦人的方法，但是在每次重新渲染时都要运行深度相等检测，我不愿为此付出代价。

它们比对应的 Records & Tuples 更加啰嗦，也更不习惯。

```jsx
const apiFilters = #{ userFilter, companyFilter };
useEffect(() => {
  fetchApiData(apiFilters).then(setApiDataInState);
}, [apiFilters]);
```

## Props and React.memo

在 Props 中保存对象的身份对于 React 性能也非常重要。

另一个非常常见的错误：在 render 中创建新的对象标识。

```jsx
const Parent = () => {
  useRerenderEverySeconds();
  return (
    <ExpensiveChild
      // someData props object is created "on the fly"
      someData={{ attr1: 'abc', attr2: 'def' }}
    />
  );
};

const ExpensiveChild = React.memo(({ someData }) => {
  return <div>{expensiveRender(someData)}</div>;
});
```

大多数时候，这都不是问题，因为 React 的速度足够快。

但有时你想优化你的应用程序，这种新对象的创建使得React.memo()毫无用处。最糟糕的是，它实际上会让你的应用程序变得更慢一些（因为它现在必须运行一个额外的 shallow equal，它总是返回false）。

另一种模式我经常在客户端代码库中看到。

```jsx
const currentUser = { name: 'Sebastien' };
const currentCompany = { name: 'Lambda Scale' };
const AppProvider = () => {
  useRerenderEverySeconds();
  return (
    <MyAppContext.Provider
      // the value prop object is created "on the fly"
      value={{ currentUser, currentCompany }}
    />
  );
};
```

尽管currentUser或currentCompany从未被更新，但每次这个提供者重新渲染时，你的 Context 值都会发生变化，从而触发所有上下文订阅者的重新渲染。

所有这些问题都可以通过 memoization 来解决。

```
const someData = useMemo(
  () => ({ attr1: 'abc', attr2: 'def' }),
  [],
);

<ExpensiveChild someData={someData} />;

```

```
const contextValue = useMemo(
  () => ({ currentUser, currentCompany }),
  [currentUser, currentCompany],
);

<MyAppContext.Provider value={contextValue} />;

```

有了 Records & Tuples，编写高性能代码很容易成为习惯。

```jsx
<ExpensiveChild someData={#{ attr1: 'abc', attr2: 'def' }} />;

<MyAppContext.Provider value={#{ currentUser, currentCompany }} />;
```

## Fetching and re-fetching

在React中获取数据的方法有很多：useEffect、HOC、Render props、Redux、SWR、React-Query、Apollo、Relay、Urql、......。

大多数情况下，我们发一个请求到后端，然后获取一些JSON数据回来。

为了说明这部分内容，我将使用react-async-hook，这是我自己非常简单的获取库，但这也适用于其他库。

让我们考虑一个经典的异步函数来获取一些API数据。

```jsx
const fetchUserAndCompany = async () => {
  const response = await fetch(
    `https://myBackend.com/userAndCompany`,
  );
  return response.json();
};
```

这个应用可以获取数据，并确保这些数据在一段时间内保持 "新鲜"（非陈旧）。

```jsx
const App = ({ id }) => {
  const { result, refetch } = useAsync(
    fetchUserAndCompany,
    [],
  );

  // We try very hard to not display stale data to the user!
  useInterval(refetch, 10000);
  useOnReconnect(refetch);
  useOnNavigate(refetch);

  if (!result) {
    return null;
  }

  return (
    <div>
      <User user={result.user} />
      <Company company={result.company} />
    </div>
  );
};

const User = React.memo(({ user }) => {
  return <div>{user.name}</div>;
});

const Company = React.memo(({ company }) => {
  return <div>{company.name}</div>;
});
```

问题：出于性能的考虑，你使用了React.memo，但每次重新获取时，你最终都会得到一个新的JS对象，有一个新的身份，一切都会重新渲染，尽管获取的数据和以前一样（深度相等的有效载荷）。

让我们想象一下这个场景：

- 你使用 "Stale-While-Revalidate "模式（先显示缓存/陈旧数据，然后在后台刷新数据）。
- 您的页面是复杂的，渲染密集型的，有大量的后台数据显示。

你导航到一个页面，这个页面第一次渲染的成本已经很高了（有缓存数据）。一秒钟后，刷新的数据又来了。尽管与缓存的数据深度相等，但一切又重新渲染。如果没有并发模式和时间切片，有些用户甚至会发现他们的UI冻结了几百毫秒。

现在，让我们把取函数转换为返回Record来代替。

```jsx
const fetchUserAndCompany = async () => {
  const response = await fetch(
    `https://myBackend.com/userAndCompany`,
  );
  return JSON.parseImmutable(await response.text());
};
```

偶然的是，JSON与Records & Tuples是兼容的，你应该能够使用JSON.parseImmutable将任何后端响应转换为Records。

注意：Robin Ricard，提案作者之一，正在推动一个新的response.immutableJson()函数。

使用Records & Tuples，如果后端返回相同的数据，你根本不需要重新渲染任何东西!

另外，如果响应中只有一个部分发生了变化，响应的其他嵌套对象仍然会保持它们的身份。这意味着，如果只有user.name发生了变化，User组件将重新渲染，但Company组件不会!

想象一下这一切对性能的影响，考虑到类似 "Stale-While-Revalidate "这样的模式正变得越来越流行，并且由SWR、React-Query、Apollo、Relay等库提供了开箱即用的功能。

## Reading query strings

在搜索用户界面中，保留querystring中过滤器的状态是一个很好的做法。然后，用户可以将链接复制/粘贴到别人那里，刷新页面，或者将其加入书签。

如果你有1或2个过滤器，这很简单，但是一旦你的搜索UI变得复杂（10多个过滤器，能够用AND/OR逻辑组成查询......），你最好使用一个好的抽象来管理你的querystring。

我个人喜欢qs：它是少数几个处理嵌套对象的库之一。

```jsx
const queryStringObject = {
  filters: {
    userName: 'Sebastien',
  },
  displayMode: 'list',
};

const queryString = qs.stringify(queryStringObject);

const queryStringObject2 = qs.parse(queryString);
// 深度对比每个嵌套结构的基本类型值
assert.deepEqual(queryStringObject, queryStringObject2); 

assert(queryStringObject !== queryStringObject2);
```

queryStringObject和queryStringObject2是深度平等的，但它们的身份已经不一样了，因为qs.parse创建了新的对象。

你可以将querystring解析集成在一个 hooks 里，用useMemo()或者use-memo-value等库来 "稳定 "querystring对象。

```jsx
const useQueryStringObject = () => {
  // Provided by your routing library, like React-Router
  const { search } = useLocation();
  return useMemo(() => qs.parse(search), [search]);
};
```

现在，假设你在树的某个深度中

```jsx
const { filters } = useQueryStringObject();
useEffect(() => {
  fetchUsers(filters).then(setUsers);
}, [filters]);
```

这里有点讨厌，但同样的问题一再发生。

尽管为了保留queryStringObject的身份使用了 useMemo()，但你最终依然会产生不必要的fetchUsers 调用。

当用户更新了 displayMode（这种模式只改变 re-rendering，而不是触发 re-fetch)，querystring 将会变化，导致 querystring 会被再次解析，又会导致新的包含 filter 属性的对象生成，从而产生不必要的 useEffect 执行。

再次， Records & Tuples 可以阻止此类情况发生。

```jsx
// This is a non-performant, but working solution.
// Lib authors should provide a method such as qs.parseRecord(search)
const parseQueryStringAsRecord = (search) => {
  const queryStringObject = qs.parse(search);
  // Note: the Record(obj) conversion function is not recursive
  // There's a recursive conversion method here:
  // https://tc39.es/proposal-record-tuple/cookbook/index.html
  return JSON.parseImmutable(
    JSON.stringify(queryStringObject),
  );
};
const useQueryStringRecord = () => {
  const { search } = useLocation();
  return useMemo(() => parseQueryStringAsRecord(search), [
    search,
  ]);
};
```

现在，即使用户更新了 displayMode,  这个 filters 对象依然保持原有的身份，而不是触发任何不必要的 re-fetch。

注意：如果“ Records & Tuples”提议被接受，则诸如qs之类的库可能会提供qs.parse Record（search）方法。

## Deeply equal JS transformations

假设在一个组件中进行以下 JS 转换

```jsx
const AllUsers = [
  { id: 1, name: 'Sebastien' },
  { id: 2, name: 'John' },
];

const Parent = () => {
  const userIdsToHide = useUserIdsToHide();

  const users = AllUsers.filter(
    (user) => !userIdsToHide.includes(user.id),
  );

  return <UserList users={users} />;
};

const UserList = React.memo(({ users }) => (
  <ul>
    {users.map((user) => (
      <li key={user.id}>{user.name}</li>
    ))}
  </ul>
));
```

每当 Parent 组件 re-render，UserList 组件也会跟着 re-render，因为 filter 后每次都产生一个新数组实例。

即使 userIdsToHide 是空的，AllUsers 是稳定的，情况也是如此。

在这种情况下， filter 操作并没有真正的过滤任何东西，它只是新增了无用的数组实例，干扰了 React.memo 优化。

这种转换操作在 React 代码中非常常见，像 filter 或者是 map 的操作符，在组件中， reducers 里， selectors，Redux 中等等...

Memoization 可以解决，但有了 Records & Tuples 我们将习惯。

```jsx
const AllUsers = #[
  #{ id: 1, name: 'Sebastien' },
  #{ id: 2, name: 'John' },
];
const filteredUsers = AllUsers.filter(() => true);
AllUsers === filteredUsers;
// true
```

## Records as React key

假设我们有一个 list 要渲染：

```jsx
const list = [
  { country: 'FR', localPhoneNumber: '111111' },
  { country: 'FR', localPhoneNumber: '222222' },
  { country: 'US', localPhoneNumber: '111111' },
];
```

你会用什么 key ？

考虑到 country 和 localPhoneNumber 在列表里不是绝对的唯一，你有两种选择.

### 数组 Index 作为 key:

```jsx
<>
  {list.map((item, index) => (
    <Item key={`poormans_key_${index}`} item={item} />
  ))}
</>
```

这通常可以工作，但它并不理想，尤其是如果 list 中每项数据如果重新排序的话。

### Composite key:

```jsx
<>
  {list.map((item) => (
    <Item
      key={`${item.country}_${item.localPhoneNumber}`}
      item={item}
    />
  ))}
</>
```

这种解决方案能更好地解决 list 重新排序，但只有在我们确定 tuple 组合是唯一的。

在这种情况下，直接用 Records 作为 key  不是更方便呢？

```jsx
const list = #[
  #{ country: 'FR', localPhoneNumber: '111111' },
  #{ country: 'FR', localPhoneNumber: '222222' },
  #{ country: 'US', localPhoneNumber: '111111' },
];
<>
  {list.map((item) => (
    <Item key={item} item={item} />
  ))}
</>
```

这个[建议](https://twitter.com/barklund/status/1289273309889064960)是Morten Barklund 提出的。

## Explicit API surface

我们来看看 Typescript 的组件：

```jsx
const UsersPageContent = ({
  usersFilters,
}: {
  usersFilters: UsersFilters,
}) => {
  const [users, setUsers] = useState([]);

  // poor-man's fetch
  useEffect(() => {
    fetchUsers(usersFilters).then(setUsers);
  }, [usersFilters]);

  return <Users users={users} />;
};
```

这段代码能否产生无限循环，取决于 userFiilers 这个 props 是否稳定。这创造了一个隐含的API约定，应该被父组件的实现者记录并清楚地理解，尽管使用TypeScript，但这并没有反映在类型系统中。

下列代码将导致无限循环，但是 TypeScript 却无法阻止它。

```jsx
<UsersPageContent
  usersFilters={{ nameFilter, ageFilter }}
/>
```

有了 Records & Tuples，我们可以告诉 TypeScript 期望获得一个  Record:

```jsx
const UsersPageContent = ({
  usersFilters,
}: {
  usersFilters: #{nameFilter: string, ageFilter: string}
}) => {
  const [users, setUsers] = useState([]);

  // poor-man's fetch
  useEffect(() => {
    fetchUsers(usersFilters).then(setUsers);
  }, [usersFilters]);

  return <Users users={users} />;
};
```

注意：`#{nameFilter: string, ageFilter: string}` 是我自己发明的，我们还不知道 TypeScript 的语法是怎样的。

TypeScript 将会对以下代码编译失败

```jsx
<UsersPageContent
  usersFilters={{ nameFilter, ageFilter }}
/>
```

但 TypeScript 预期会接收

```jsx
<UsersPageContent
  usersFilters={#{ nameFilter, ageFilter }}
/>
```

有了 Records & Tuples，我们可以在编译期间避免无限循环。

我们将拥有显式的方式告诉编译器，我们的实现是对象身份铭感的（或者依赖于值比较）

注意： readonly 不会解决问题：它只能防止 mutation(突变），但无法确保一个稳定的身份。

## Serialization guarantee （序列化保证）

你可能想要确认团队中的开发者不会把不可序列化的东西放到全局 state 中。如果你打算将 state

发送给后端，或者想要持久化存储在本地的 localStorage( 或者是 React-Native 的 AsyncStorage)

这一点很重要。

为确保这一点，你只需要确保根对象是一个 Record。这将保证所有嵌套属性也是基本元素，包括嵌套的 records 和  tuples。

这里有一个 Redux 的例子，保证 Redux 存储的 store 可以一直被序列化。

```jsx
if (process.env.NODE_ENV === 'development') {
  ReduxStore.subscribe(() => {
    if (typeof ReduxStore.getState() !== 'record') {
      throw new Error(
        "Don't put non-serializable things in the Redux store! " +
          'The root Redux state must be a record!',
      );
    }
  });
}
```

注意： 这不是一个完美的承诺，因为 Symbole 也可以被存入 Record，而且它不可被序列化。

## CSS-in-JS performances

让我们考虑一些主流的 CSS-in-JS  库，使用 css prop

```jsx
const Component = () => (
  <div
    css={{
      backgroundColor: 'hotpink',
    }}
  >
    This has a hotpink background.
  </div>
);
```

你的 CSS-in-JS 库在重新渲染的时候都会接受一个新的 CSS 对象。

在首次渲染时，它会 hash 这个对象作为唯一的 class名，并且插入这段 css。 style 对象在每次 re-render 时都拥有不同的标识，然后 CSS-in-JS 库就会反复地生成 hash。

```jsx
const insertedClassNames = new Set();

function handleStyleObject(styleObject) {
  // computeStyleHash re-executes every time
  const className = computeStyleHash(styleObject);

  // only insert the css for this className once
  if (!insertedClassNames.has(className)) {
    insertCSS(className, styleObject);
    insertedClassNames.add(className);
  }

  return className;
}
```

有了 Records & Tuples， style 对象的身份就会被保留。

```jsx
const Component = () => (
  <div
    css={#{
      backgroundColor: 'hotpink',
    }}
  >
    This has a hotpink background.
  </div>
);
```

Records & Tuples 可以作为  Map 的key 使用。这可以让 CSS-in-JS 库的实现更快。

```jsx
const insertedStyleRecords = new Map();
function handleStyleRecord(styleRecord) {
  let className = insertedStyleRecords.get(styleRecord);
  if (!className) {
    // computeStyleHash is only executed once!
    className = computeStyleHash(styleRecord);
    insertCSS(className, styleRecord);
    insertedStyleRecords.add(styleRecord, className);
  }
  return className;
}
```

我们目前还不知道 Records & Tuples 的性能（这取决于浏览器厂商的实现），但我认为可以肯定地说，它将比创建等价对象，然后将其 hash 为 className 更快。

注意：有些 CSS-in-JS 库通过 Babel Plugin 在编译阶段转换静态的对象结构作为常量，但针对动态样式就会非常困难。

```jsx
const staticStyleObject = { backgroundColor: 'hotpink' };
const Component = () => (
  <div css={staticStyleObject}>
    This has a hotpink background.
  </div>
);
```

## Conclusion

很多React的性能和行为问题都与对象身份有关。

Records & Tuples 将通过提供某种 "自动记忆 "的方式，确保对象身份开箱即 "更稳定"，帮助我们更容易地解决这些React问题。

使用TypeScript，我们可能会更好地表达你的API 对于对象身份敏感。

我希望你现在和我一样对这个提议感到兴奋！

谢谢你的阅读！