## 创建 update 的方式
- HostRoot: `ReactDOM.render`
- ClassComponent: `this.setState`
- ClassComponent: `this.forceUpdate`
- FunctionComponent: `useState`
- FunctionComponent: `useReducer`

流程：
```
触发状态更新（根据场景调用不同方法）

    |
    |
    v

创建Update对象

    |
    |
    v

scheduleUpdateOnFiber

    |
    |
    v

从fiber到root（`markUpdateLaneFromFiberToRoot`）

    |
    |
    v

调度更新（`ensureRootIsScheduled`）

    |
    |
    v

render 阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）

    |
    |
    v

commit阶段（`commitRoot`）
```

### Update

`ClassComponent` 与 `HostRoot`（即 `rootFiber.tag` 对应类型）共用同一种 `Update` 结构，如下：

```ts
type Update<State> = {
  // TODO: Temporary field. Will remove this by storing a map of
  // transition -> event time on the root.
  eventTime: number
  lane: Lane

  tag: UpdateState | ReplaceState | ForceUpdate | CaptureUpdate
  // ClassComponent: 调用 setState 传递的参数，当为函数时，接受 (prevState, nextProps) 参数
  // HostRoot:  { element: ReactDOM.render 的第一个传参 }
  payload: any
  // 更新的回调函数
  // ClassComponent: this.setState 的第二个传参
  // FunctionComponent: 
  // HostRoot： ReactDOM.render 的第三个传参
  callback: (() => mixed) | null

  next: Update<State> | null
};
```

Fiber 上的多个 `Update` 会组成链表并被包含在 `fiber.updateQueue` 中

### UpdateQueue
`updateQueue` 有三种类型：

**HostComponent**: 
```ts
workInProgress.updateQueue = updatePayload;
```
其中 `updatePayload` 为数组形式，他的偶数索引的值为变化的 `prop key`，奇数索引的值为变化的 `prop value`

**ClassComponent** 与 **HostRoot** 使用的 `UpdateQueue` 结构如下：

```ts
type UpdateQueue<State> = {
  // 本次更新前该 fiber 的 state，Update 基于该 state 计算更新后的 state 
  baseState: State
  firstBaseUpdate: Update<State> | null
  lastBaseUpdate: Update<State> | null
  shared: {
    // 触发更新时，产生的 Update 会保存在 shared.pending 中形成单向环状链表。当由 Update 计算 state 时这个环会被剪开并连接在 lastBaseUpdate 后面
    pending: Update<State> | null
    interleaved: Update<State> | null
  }
  // 有 callback （this.setState 的第二个参数或者 ReactDom.render 的第三个参数） 的 update 对象列表
  // callback 在 commit 的 layout阶段被执行 （通过调用 commitUpdateQueue）
  effects: Array<Update<State>> | null
};
```

### 更新流程
#### Legacy 模式（同步）
##### 情况一：事件回调中多次调用 `setState`
```jsx
function Counter() {
  const [count, setCount] = useState(0)
  
  return (
    <button 
      onClick={() => {
        setCount(1)
        setCount(2)
      }}
    >
      { count }
    </button>
  )
}
```
**render次数**：1
**render时机**：所有 `setState` 调用后
**分析**：React 事件回调内部使用了 `batchedEventUpdates` 会为 `executionContext` 添加 `EventContext`，在调度更新过程中（`scheduleUpdateOnFiber`）根据判断会调度 `performSyncWorkOnRoot`（而不是同步调用），在 `batchedEventUpdates` 在回调最后再来调用 `flushSyncCallbacks` 从而调用 `performSyncWorkOnRoot`（同步）进入 `render` 阶段

##### 情况二：非 `React` 上下文下多次调用 `setState`

```jsx
function Counter() {
  const [count, setCount] = useState(0)
  
  return (
    <button 
      onClick={() => {
        setTimeout(() => {
          setCount(1)
          setCount(2)
        })
      }}
    >
      { count }
    </button>
  )
}
```

**render次数**：同 `setState` 调用次数
**render时机**：每次 `setState` 调用后会同步 `render`
**分析**：在调度更新过程中（`scheduleUpdateOnFiber`）根据判断（`executionContext === NoContext`）会同步调用 `performSyncWorkOnRoot`

##### 情况三：render 过程中调用 `setState`

```jsx
function Counter() {
  const [count, setCount] = useState(0)

  if (count < 1) {
    setCount(1)
  }
  
  return (
    <div>
      count: { count }
    </div>
  )
}
```
**render次数**：在第一次 `render` 后，`setState` 会再同步触发一次 `render`，然后再进入 `commit` 阶段
**分析**：组件第一次 `render` 后，在这次 `render` 过程中如果同步触发了更新，则再次 `render`
> 具体见 `renderWithHooks` 对 `didScheduleRenderPhaseUpdateDuringThisPass` 的判断


#### Concurrent 模式（异步）
##### 情况一：事件回调中多次调用 `setState`
同 `Legacy` 模式

##### 情况二：非 `React` 上下文下多次调用 `setState`
表现同情况一

**分析**：核心判断逻辑在 `ensureRootIsScheduled` 方法中

```js
const existingCallbackPriority = root.callbackPriority;
if (existingCallbackPriority === newCallbackPriority) {
  // The priority hasn't changed. We can reuse the existing task. Exit.
  return;
}
```

##### 情况三：render 过程中调用 `setState`
同 `Legacy` 模式

##### 情况四：异步调用 `setState`
> 注：以下表现是基于当前内部实现，不稳定，后期可能会变化

###### 1. 在 `react` 事件回调上下文下
> `lane === SyncLane`

会同步执行更新

```js
const handleClick = () => {
  setState(1)
  Promise.resolve().then(() => {
    setState(2)
  })
}
```

**render次数**：2 次

###### 2. 非 `react` 事件回调上下文下
> `lane === DefaultLane`

因为内部调度使用的是 `MessageChannel`（如果不支持使用 `setTimeout`），~~而且根据 `shouldYieldToHost()` 来判断当前帧剩余的时间是否用尽从而暂停未完成的任务（在源码中每个时间片时 5ms，这个值会根据设备的 fps 调整）~~，所以只要异步任务回调早于 `MessageChannel` 回调，如 `Promise`，就只会执行一次 `render`，其余情况如 `setTimeout`、`requestAnimationFrame`
等就会执行两次

```js
// 总共执行一次 render

const handleClick = () => {
  setTimeout(() => {
    setState(1)
    Promise.resolve().then(() => {
      setState(2)
    })
  })
}
```

```js
// 总共执行一次 render

const handleClick = () => {
  Promise.resolve().then(() => {
    setState(1)
  })
  Promise.resolve().then(() => {
    setState(2)
  })
}
```
