#### 创建 update 的方式
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

从fiber到root（`markUpdateLaneFromFiberToRoot`）

    |
    |
    v

调度更新（`ensureRootIsScheduled`）

    |
    |
    v

render阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）

    |
    |
    v

commit阶段（`commitRoot`）
```

#### Update

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

#### UpdateQueue
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
