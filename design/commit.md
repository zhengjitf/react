## commit 阶段
`commitRoot` => `commitRootImpl`

### `flushPassiveEffects`
> 执行条件：rootWithPendingPassiveEffects !== null


主要有两个操作：
1. 遍历 `pendingPassiveHookEffectsUnmount`，调用 `effect.destory`
2. 遍历 `pendingPassiveHookEffectsMount`，调用 `effect.create` 并将返回值赋值给 `effect.destory`

注：`pendingPassiveHookEffectsUnmount` 和 `pendingPassiveHookEffectsMount` 数组项是二维数组，第一项为 `effect` 对象，第二项为 `fiber`。向数组内 `push` 数据的操作发生在 layout 阶段 `commitLayoutEffectOnFiber` 方法内部的 `schedulePassiveEffects` 方法中

### 调度 passiveEffects
> 执行条件：当前 fiber 及其子树包含 passiveEffect flag 时


### before mutation阶段（执行DOM操作前）

#### `commitBeforeMutationEffects`
> **关键词**: `getSnapshotBeforeUpdate`

**类组件**：
1. 调用 `getSnapshotBeforeUpdate`，将返回值挂载到实例的 `__reactInternalSnapshotBeforeUpdate` 属性上，以便调用 `componentDidUpdate` 时作为第三个参数传递

**HostRoot**：
1. 清空子节点

<!-- 主要有两个操作：
1. 调用 `commitBeforeMutationLifeCycles` 方法，触发类组件的 `getSnapshotBeforeUpdate` 的调用
2. 如果有 `passive effects`, 调度 `flushPassiveEffects` （`scheduleCallback(..., () => flushPassiveEffects())`）

注：整个 `useEffect` 异步调用分为三步：

1. `before mutation` 阶段在 `scheduleCallback` 中调度 `flushPassiveEffects`
2. `layout` 阶段之后将 `effectList` 赋值给 `rootWithPendingPassiveEffects`
3. `scheduleCallback` 触发 `flushPassiveEffects`，`flushPassiveEffects` 内部遍历 `rootWithPendingPassiveEffects` -->

### mutation阶段（执行DOM操作）


#### `commitMutationEffects`
> **关键词**：`ref.current = null`, `useLayoutEffect 清理函数`,  `componentWillUnmount`

**类组件**：
1. 标记为删除的 `fiber` 会调用对应类组件的 `componentWillUnmount`
2. 设置 `ref` 为 `null`
3. 根据 `flags` 执行对应的 DOM 操作

**函数组件**：
1. 标记为删除的 `fiber` 会调用对应函数组件的 `useLayoutEffect` 的销毁函数
2. 设置 `ref` 为 `null`
3. 根据 `flags` 执行对应的 DOM 操作


<!-- commitMutationEffects 会遍历 effectList，对每个 `Fiber` 节点执行如下三个操作：

- 根据 ContentReset effectTag 重置文字节点
- 设置 `ref` 为 `null`
- 根据 effectTag 执行 DOM 操作，其中 effectTag 包括 (Placement | Update | Deletion | Hydrating)，
当 `effectTag === Placement`， 会清除 ref 和调用 `componentWillUnmount`

当Fiber节点含有Update effectTag，意味着该Fiber节点需要更新。调用的方法为commitWork，他会根据Fiber.tag分别处理

当 `fiber.tag` 为 `FunctionComponent`，会调用 `commitHookEffectListUnmount` 。该方法会遍历 `effectList`，执行所有`useLayoutEffect` hook的销毁函数 -->

### layout阶段（执行DOM操作后）
#### `commitLayoutEffects` => `commitLifeCycles`

> **关键词**：`componentDidMount`, `componentDidUpdate`,`useLayoutEffect`, `ref`

**函数组件**：
1. 同步执行 `useLayoutEffect` 的回调（通过调用 `commitHookEffectListMount`）
<!-- 2. 然后调度`useEffect` 的销毁与回调函数（通过调用 `schedulePassiveEffects`，执行 `pendingPassiveHookEffectsUnmount.push` 和 `pendingPassiveHookEffectsMount.push`） -->
3. 赋值 `ref`

**类组件**：
1. 会通过判断 `current === null` 区分是 `mount` 还是 `update`，调用 `componentDidMount` 或 `componentDidUpdate`
2. 触发 `this.setState` 第二个参数指定的回调函数
3. 赋值 `ref`

**HostRoot**：
1. 触发 `ReactDOM.render` 第三个参数指定的回调函数

