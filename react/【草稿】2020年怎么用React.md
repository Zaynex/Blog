就写写自己是怎么用的吧。

在 React 中，我们其实可以用 React Context 帮助我们做跨组件数据传递。hooks 取代了
全局状态管理： `unstated-next`
immutable 操作: `immer`

如果你也采用这种方式，那可能还需要 `use-immer` 来帮助

实例如下：

1. Container —— 基于 React 的 context API 定义顶层数据结构和改变数据的方法。

```
// LessonContainer
import { useImmer } from 'use-immer'
import { createContainer } from 'unstated-next'

const defaultState = {
  now: [new Date().getFullYear(), new Date().getMonth(), new Date().getDate(), new Date().getDay()],
  count: 0,
  student: [
    {
      name: 'zaynex',
      age: '10',
    },
  ],
  mockLessons: [
    {
      teacher: '小李',
      count: 6, // 子课程
      name: '课节名称',
      time: '02-07 周五 12：00（30分钟）'
    }
  ]
}

/**
 * 对外只暴露具体改变的方法和数据，所有的副作用交由内部处理
 * 通过 dispatch 挂载方法，统一所有的副作用
 */
function useLesson() {
  let [data, _setData] = useImmer(defaultState)
  const dispatch = {
    decrement: () =>
      _setData(draftState => {
        draftState.count--
      }),
    increment: () =>
      _setData(draftState => {
        draftState.count++
      }),
    updateAge: () =>
      _setData(draftState => {
        draftState.student[0].age++
      }),
  }
  return {
    data,
    dispatch,
  }
}

const LessonContainer = createContainer(useLesson)

export default LessonContainer
```

然而，这里还可以再优化，我们每次都会重新生成一个 dispatch。

可以给 dispatch 稍加调整。

```
const dispath = useMemo(() => ({
  decrement: () =>
    _setData(draftState => {
      draftState.count--
    }),
  increment: () =>
    _setData(draftState => {
      draftState.count++
    }),
  updateAge: () =>
    _setData(draftState => {
      draftState.student[0].age++
    }),
}), [])
```

2. 顶层注入，类似 Redux 的 Provider

```
import LessonContainer from './LessonContainer'

function App() {
return <LessonContainer.Provider>
            <DetailView />
          </LessonContainer.Provider>
)
```

3. 业务层调用

```
function DetailView() {
  const lessonContainer = LessonContainer.useContainer()
  const mockLessons = lessonContainer.data.mockLessons
  return useMemo(
    () => (
      <div>
        {mockLessons.map(item => (
          <div>
            <span>{item.name}</span>
            <span>teacher: {item.teacher}</span>
          </div>
        ))}
      </div>
    ),
    [mockLessons],
  )
}
```
