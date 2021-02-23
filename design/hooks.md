## dispatcher
以 `useState` [code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/wJWGhlDSQTCz1s3vCQ_mSQ) 为例：

```ts
export function useState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```
在使用 `useState` 时，内部其实调用的是 `dispatcher.useState`，不同情况下 `dispatcher` 不同，运行时通过 `ReactCurrentDispatcher.current` 设置和获取 `dispatcher`，我们只关注其中的 `mount` 和 `update` 类型

```ts
const HooksDispatcherOnMount: Dispatcher = {
  readContext,

  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  useImperativeHandle: mountImperativeHandle,
  useLayoutEffect: mountLayoutEffect,
  useMemo: mountMemo,
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  useDebugValue: mountDebugValue,
  useDeferredValue: mountDeferredValue,
  useTransition: mountTransition,
  useMutableSource: mountMutableSource,
  useOpaqueIdentifier: mountOpaqueIdentifier,

  unstable_isNewReconciler: enableNewReconciler,
};

const HooksDispatcherOnUpdate: Dispatcher = {
  readContext,

  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  useImperativeHandle: updateImperativeHandle,
  useLayoutEffect: updateLayoutEffect,
  useMemo: updateMemo,
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  useDebugValue: updateDebugValue,
  useDeferredValue: updateDeferredValue,
  useTransition: updateTransition,
  useMutableSource: updateMutableSource,
  useOpaqueIdentifier: updateOpaqueIdentifier,

  unstable_isNewReconciler: enableNewReconciler,
};
```

#### 设置 dispatcher
函数组件的 `render` 逻辑主要在函数 [`renderWithHooks`](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/IUjU2eGQS_2Tgl7bjcb6kA) 中：

```ts
ReactCurrentDispatcher.current =
      current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;
```
[code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/PaHMnqp-Rw-iaIDrcGpv0A)

**在调用函数组件前**，会根据 `current === null || current.memoizedState === null` 判断 `dispatcher` 是 `mount` 还是 `update` 类型

**在函数组件调用后**，会设置 `ReactCurrentDispatcher.current = ContextOnlyDispatcher` [code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/vCp6PsLER6aoOQkNm8tslQ)，以防止如下的使用情况：

```ts
useEffect(() => {
  useState(0);
})
```
调用 `useState` 时，直接抛出异常

```ts
const ContextOnlyDispatcher: Dispatcher = {
  useCallback: throwInvalidHookError,
  useContext: throwInvalidHookError,
  useEffect: throwInvalidHookError,
  useImperativeHandle: throwInvalidHookError,
  useLayoutEffect: throwInvalidHookError,
  // ...省略
}
```

## mount
mount 阶段，hooks 函数会调用 `mountWorkInProgressHook` [code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/SfKPJNGRSbGSnt3JQI0ekw) 生成一个对应的 `hook` 对象，结构如下：

```ts
type Hook = {
  memoizedState: any
  // 第一个未被应用的 update（baseQueue.next） 之前的 state
  // 假设当前 state = 1，相继触发三次 dispatchAction，创建三个 update： Q1 Q2 Q3
  // 其中 Q1 和 Q2 优先级不够被当前更新跳过，而 Q3 会被应用
  // 则 baseQueue 对应的链表如下：
  //      Q1 -> Q2 -> Q3
  // 此时，memoizedState 为按顺序计算 Q1、Q2、Q3 之后得到的 state
  // baseState 是 Q1 之前的 state 即 baseState = 1
  baseState: any
  // baseQueue 表示优先级不够未被应用的 update 链表
  // 指向的是最后一个 update
  // 注意：为了保证执行顺序，链表上保留了已经应用的中间态的 update
  baseQueue: Update<any, any> | null
  // 主要存储 pending update
  queue: UpdateQueue<any, any> | null
  next: Hook | null
};
```
每个 `hooks` 函数都会生成对应的 `hook` 对象，以链表结构连接，挂载到函数组件对应 `fiber` 的 `memoizedState` 属性上

## update
update 阶段，`hooks` 函数会通过 `updateWorkInProgressHook` [code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/FHZprRpXSLm2v6FdxLunaA) 获取对应的 `hook` 对象

由于 `hook` 对象形成了链表结构，通过 `next` 可以将当前的 `hook` 指针进行移动，这种机制也带来了如下限制：
1. `hooks` 函数不能被包含在条件判断块下
2. `hooks` 函数必须是在函数组件 `render` 时同步执行（在 `render` 同步执行后会重置 `dispatcher`）

## useState
#### mount 阶段
[code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/pREMyfBkT3aKRzw4GpBfag)
`mount` 阶段，调用的是 `mountState`，主要逻辑如下：

- 创建 `hook` 对象
- 根据初始值，初始化 `hook.memoizedState` 和 `hook.baseState`
- 创建 `updateQueue`（`hook.queue`）

`mountState` 函数返回的是 `[hook.memoizedState, dispatch]`，
`dispatch` 内部已经绑定了当前的 `fiber` （函数组件对应的 `fiber`）和 `hook.queue`，并挂载到 `hook.queue.dispatch`，在 `update` 阶段时直接返回该 `dispatch`，所以 `useState` 返回的数组对象的第二项（即 `dispatch`）引用不会变

#### update 阶段
`update` 阶段，调用的是 `updateState`：

```ts
function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```
内部调用的是 `updateReducer` [code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/EDdfRirjRRSpWg-li4-Acg)

### 以 useState 为例
1. 函数组件第一次调用 (`mount`) 时，`useState` 对应的是 `mountState` 返回 `[初始值, dispatchAction]`
2. 调用 `dispatchAction` 会创建一个 `update` 对象放入 `hook.queue.pending` 末尾，然后调用 `scheduleUpdateOnFiber` 开始 `render` 过程，
3. render 过程中再次调用函数组件，再次调用 `useState`，此时对应的是 `updateState`，内部调用 `updateReducer`，根据 `hook.baseQueue` 和 
`hook.queue.pending` 更新状态
