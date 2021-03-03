## hooks 的限制
- hooks 函数不能被包含在条件判断语句中，要确保每次 render 时，hooks 函数被调用的顺序和个数要相同
- hooks 函数必须在函数组件 render 时同步执行
不能出现如下这些情况：
  ```ts
  useEffect(() => {
    const [count, setCount] = useState(0)
  })

  setTimeout(() => {
    const [count, setCount] = useState(0)
  })
  ```
- rerender 的次数有限，目前内部指定最大次数为 25


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

## HooksDispatcherOnMount
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

## HooksDispatcherOnUpdate
update 阶段，`hooks` 函数会通过 `updateWorkInProgressHook` [code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/FHZprRpXSLm2v6FdxLunaA) 获取对应的 `hook` 对象

在这里分为两种情况：
1. 组件 update
2. 组件 rerender

**组件 update 时**，之前 mount 阶段生成的 hook 链表已挂载到 `current fiber`，update 时会按顺序从之前的 hook 链表拷贝生成 `workInProgress fiber` 对应的 hook 链表

**rerender 时**，`workInProgress fiber` 已存在 hook 链表，直接使用

注：`rerender` 是在函数组件 `render` 过程中同步调用了更新方法（如 `setState`）， 这会导致在本次 `render` 结束后再次 `render`

```ts
function updateWorkInProgressHook(): Hook {
  let nextCurrentHook: null | Hook;
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    nextCurrentHook = currentHook.next;
  }

  let nextWorkInProgressHook: null | Hook;
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // 这里属于 rerender 的情况

    // There's already a work-in-progress. Reuse it.
    workInProgressHook = nextWorkInProgressHook;
    nextWorkInProgressHook = workInProgressHook.next;

    currentHook = nextCurrentHook;
  } else {
    // 这里属于 update 的情况

    // Clone from the current hook.
    
    // 如果 nextCurrentHook 为 null，表示当前调用的 hook 函数个数大于第一次调用个数，这是不允许的，一般是使用了条件判断语句控制 hook 函数的执行
    invariant(
      nextCurrentHook !== null,
      'Rendered more hooks than during the previous render.',
    );
    currentHook = nextCurrentHook;

    const newHook: Hook = {
      memoizedState: currentHook.memoizedState,

      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,

      next: null,
    };

    if (workInProgressHook === null) {
      // This is the first hook in the list.
      currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
    } else {
      // Append to the end of the list.
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }
  return workInProgressHook;
}
```

## useState
#### mount 阶段
[code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/pREMyfBkT3aKRzw4GpBfag)
`mount` 阶段，调用的是 `mountState`，主要逻辑如下：

- 创建 `hook` 对象
- 根据初始值，初始化 `hook.memoizedState` 和 `hook.baseState`
- 创建 `updateQueue`（`hook.queue`）


`updateQueue` 类型如下：
```ts
// UpdateQueue.pending 指向最后一个未处理的 update
// UpdateQueue.pending.next 指向第一个未处理的 update
type UpdateQueue<S, A> = {
  pending: Update<S, A> | null
  interleaved: Update<S, A> | null
  dispatch: (A => mixed) | null
  lastRenderedReducer: ((S, A) => S) | null
  lastRenderedState: S | null
};

// Update 是一个环状链表
type Update<S, A> = {
  lane: Lane
  action: A
  eagerReducer: ((S, A) => S) | null
  // 提前计算的新 state
  eagerState: S | null
  next: Update<S, A>
  priority?: ReactPriorityLevel
};
```

`mountState` 函数返回的是 `[hook.memoizedState, dispatch]`，
`dispatch` 内部已经绑定了当前的 `fiber` （函数组件对应的 `fiber`）和 `hook.queue`，并挂载到 `hook.queue.dispatch`，在 `update` 阶段时直接返回该 `dispatch`，所以 `useState` 返回的数组对象的第二项（即 `dispatch`）引用不会变

```ts
const dispatch: Dispatch<
  BasicStateAction<S>,
> = (queue.dispatch = (dispatchAction.bind(
  null,
  currentlyRenderingFiber,
  queue,
): any));
```

所以，调用 `setState` 时内部执行的是 `dispatchAction`，主要逻辑如下：
1. 创建 update 对象追加到 updateQueue
2. 如果 updateQueue 为空，则提前计算新的 state，如果新的 state 和当前 state 相同，则直接跳出，不再执行后续逻辑
3. 执行 `scheduleUpdateOnFiber`，调度更新操作

#### update 阶段
`update` 阶段，调用的是 `updateState`：

```ts
function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```
内部调用的是 `updateReducer` [code](https://api.codestream.com/c/X-vb0kCFvXvIJGWz/EDdfRirjRRSpWg-li4-Acg)，主要逻辑如下：
1. 根据当前 hook 对象上的 `baseState` 、`baseQueue` 以及 `queue.pending` 计算新的 `state` 和更新 `baseState` 、 `baseQueue` 以及清空 `queue.pending`
2. 如果新 state 与当前 state 不相同则标记更新，执行后续更新操作，否则直接跳出

## useContext
`mount` 和 `update` 调用的都是 `readContext`：

主要逻辑：
1. 在 fiber 对象上标记该 context 依赖 (设置 `fiber.dependencies`)
2. 返回 `context.value`

```ts
function readContext<T>(
  context: ReactContext<T>,
  observedBits: void | number | boolean,
): T {
  if (lastContextWithAllBitsObserved === context) {
    // Nothing to do. We already observe everything in this context.
  } else if (observedBits === false || observedBits === 0) {
    // Do not observe any updates.
  } else {
    let resolvedObservedBits; // Avoid deopting on observable arguments or heterogeneous types.
    if (
      typeof observedBits !== 'number' ||
      observedBits === MAX_SIGNED_31_BIT_INT
    ) {
      // Observe all updates.
      lastContextWithAllBitsObserved = ((context: any): ReactContext<mixed>);
      resolvedObservedBits = MAX_SIGNED_31_BIT_INT;
    } else {
      resolvedObservedBits = observedBits;
    }

    const contextItem = {
      context: ((context: any): ReactContext<mixed>),
      observedBits: resolvedObservedBits,
      next: null,
    };

    if (lastContextDependency === null) {
      invariant(
        currentlyRenderingFiber !== null,
        'Context can only be read while React is rendering. ' +
          'In classes, you can read it in the render method or getDerivedStateFromProps. ' +
          'In function components, you can read it directly in the function body, but not ' +
          'inside Hooks like useReducer() or useMemo().',
      );

      // This is the first dependency for this component. Create a new list.
      lastContextDependency = contextItem;
      currentlyRenderingFiber.dependencies = {
        lanes: NoLanes,
        firstContext: contextItem,
        responders: null,
      };
    } else {
      // Append a new context item.
      lastContextDependency = lastContextDependency.next = contextItem;
    }
  }
  return isPrimaryRenderer ? context._currentValue : context._currentValue2;
}
```

## useEffect

### 整体流程

**render 阶段**
1. 重置 `fiber.updateQueue` 为 null
2. mount 时调用 `mountEffect` 创建一个 tag 添加了 `HookHasEffect` 的 effect 对象；
update 时调用 `updateEffect` 创建一个 effect 对象，如果 `deps` 变化则会为其 tag 添加`HookHasEffect`
3. 然后将创建的 effect 添加到 `fiber.updateQueue`


**commit 阶段**：
1. 遍历 `fiber.deletions` , 再遍历每个 `fiberToDelete` 的 `updateQueue`，调用  `destroy`（如果不为 null）
2. 遍历 `fiber.updateQueue`, 调用 tag 为 `HookPassive | HookHasEffect` 的 `effect`对象的 `create`，将返回值赋给 `effect.destory`

#### mount 阶段
```ts
function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(
    PassiveEffect | PassiveStaticEffect,
    HookPassive,
    create,
    deps,
  );
}

function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    undefined,
    nextDeps,
  );
}
```

```ts
function pushEffect(tag, create, destroy, deps) {
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    // Circular
    next: null,
  };
  let componentUpdateQueue: null | FunctionComponentUpdateQueue = currentlyRenderingFiber.updateQueue;
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = componentUpdateQueue;
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect;
}
```

#### update 阶段

```ts
function updateEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return updateEffectImpl(PassiveEffect, HookPassive, create, deps);
}

function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }

  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    destroy,
    nextDeps,
  );
}
```
