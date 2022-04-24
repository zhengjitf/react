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

## useReducer
#### mount 阶段
调用 `mountReducer`，与 `mountState` 逻辑大致相同，主要区别是：
1. `mountReducer` 需要设置 `reducer`，而 `mountState` 内置 `basicStateReducer`
2. `mountReducer` 可以指定初始值的生成函数 `init`

```ts
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const hook = mountWorkInProgressHook();
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = ((initialArg: any): S);
  }
  hook.memoizedState = hook.baseState = initialState;
  const queue = (hook.queue = {
    pending: null,
    interleaved: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });
  const dispatch: Dispatch<A> = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));
  return [hook.memoizedState, dispatch];
}
```

#### update 阶段
调用的是 `updateReducer`，在上面对 `useState` 分析已做过介绍

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

## useLayoutEffect
和 `useEffect` 相同，`mount` 阶段和 `update` 阶段内部分别调用的是 `mountEffectImpl` 和 `updateEffectImpl`，但传递的参数不同：

```ts
// useEffect
mountEffectImpl(/* fiberFlags */ PassiveEffect | PassiveStaticEffect,  /* hookFlags */ HookPassive, create, deps)
updateEffectImpl(/* fiberFlags */ PassiveEffect, /* hookFlags */ HookPassive, create, deps)

// useLayoutEffect
mountEffectImpl(/* fiberFlags */ UpdateEffect, /* hookFlags */ HookLayout, create, deps)
updateEffectImpl(/* fiberFlags */ UpdateEffect, /* hookFlags */ HookLayout, create, deps)
```
`create` 函数和 `destroy` 函数是在 `commit` 阶段被调用，具体调用时机和条件请看 `commit` 阶段解析 

## useCallback
`useCallback` 逻辑比较简单，主要是通过比较 `deps` 前后是否有变化，决定是否使用缓存的 `callback`
#### mount
```ts
function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

#### update
```ts
function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

## useMemo
与 `useCallback` 类似，区别是缓存的是 `nextCreate` 函数的返回值
#### mount

```ts
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

#### update

```ts
function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    // Assume these are defined. If they're not, areHookInputsEqual will warn.
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

## useRef
`useRef` 生成的是一个可变对象 `{ current: null }`，引用不变，但内部的 `current` 属性可被更改
#### mount

```ts
function mountRef<T>(initialValue: T): {|current: T|} {
  const hook = mountWorkInProgressHook();
  const ref = {current: initialValue};
  hook.memoizedState = ref;
  return ref;
}
```

#### update

```ts
function updateRef<T>(initialValue: T): {|current: T|} {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;
}
```

## useImperativeHandle

`useImperativeHandle(ref, create, [deps])` 主要逻辑：相当于在 `useLayoutEffect` 的回调中，将 `create` 返回数据挂载到 `ref` 上

#### mount
```ts
function mountImperativeHandle<T>(
  ref: {|current: T | null|} | ((inst: T | null) => mixed) | null | void,
  create: () => T,
  deps: Array<mixed> | void | null,
): void {
  // TODO: If deps are provided, should we skip comparing the ref itself?
  const effectDeps =
    deps !== null && deps !== undefined ? deps.concat([ref]) : null;

  return mountEffectImpl(
    UpdateEffect,
    HookLayout,
    imperativeHandleEffect.bind(null, create, ref),
    effectDeps,
  );
}

function imperativeHandleEffect<T>(
  create: () => T,
  ref: {|current: T | null|} | ((inst: T | null) => mixed) | null | void,
) {
  if (typeof ref === 'function') {
    const refCallback = ref;
    const inst = create();
    refCallback(inst);
    return () => {
      refCallback(null);
    };
  } else if (ref !== null && ref !== undefined) {
    const refObject = ref;
    const inst = create();
    refObject.current = inst;
    return () => {
      refObject.current = null;
    };
  }
}
```

#### update
```ts
function updateImperativeHandle<T>(
  ref: {|current: T | null|} | ((inst: T | null) => mixed) | null | void,
  create: () => T,
  deps: Array<mixed> | void | null,
): void {
  // TODO: If deps are provided, should we skip comparing the ref itself?
  const effectDeps =
    deps !== null && deps !== undefined ? deps.concat([ref]) : null;

  return updateEffectImpl(
    UpdateEffect,
    HookLayout,
    imperativeHandleEffect.bind(null, create, ref),
    effectDeps,
  );
}
```

## useId
### mount

```ts
function mountId(): string {
  const hook = mountWorkInProgressHook();

  const root = ((getWorkInProgressRoot(): any): FiberRoot);
  // TODO: In Fizz, id generation is specific to each server config. Maybe we
  // should do this in Fiber, too? Deferring this decision for now because
  // there's no other place to store the prefix except for an internal field on
  // the public createRoot object, which the fiber tree does not currently have
  // a reference to.
  const identifierPrefix = root.identifierPrefix;

  let id;
  if (getIsHydrating()) {
    const treeId = getTreeId();

    // Use a captial R prefix for server-generated ids.
    id = ':' + identifierPrefix + 'R' + treeId;

    // Unless this is the first id at this level, append a number at the end
    // that represents the position of this useId hook among all the useId
    // hooks for this fiber.
    const localId = localIdCounter++;
    if (localId > 0) {
      id += 'H' + localId.toString(32);
    }

    id += ':';
  } else {
    // Use a lowercase r prefix for client-generated ids.
    const globalClientId = globalClientIdCounter++;
    id = ':' + identifierPrefix + 'r' + globalClientId.toString(32) + ':';
  }

  hook.memoizedState = id;
  return id;
}
```

分为服务端渲染和客户端渲染两种情况，客户端渲染逻辑比较简单，只是使用一个全局自增的数字转换为32位的字符串作为 id（并不是一个稳定的 id），服务端渲染由 `treeId` 和 `localId` 拼接成 id。`treeId` 和 fiber 树以及当前组件在树中的位置相关，`localId` 和当前 `useId` 在当前层级的位置相关，(每次 render 前 `localId` 都会初始为 0 ?)。

`treeId` 的相关逻辑如下：

```ts
export function getTreeId(): string {
  const overflow = treeContextOverflow;
  const idWithLeadingBit = treeContextId;
  // 去掉第一位（最左位）：最左位的 1 只是作为结束位标识，用以表示前导 0，比如：00101 作为 Fork 5 of 20 ，treeContextId 即为 0b100101 用以表示需要 5 位，但使用时需去掉最左位
  const id = idWithLeadingBit & ~getLeadingBit(idWithLeadingBit);
  return id.toString(32) + overflow;
}
```

影响 `treeContextId` 取值的逻辑有以下函数：
注：只在服务端渲染时，以下函数才会被调用

```ts
export function pushTreeFork(
  workInProgress: Fiber,
  totalChildren: number,
): void {
  // This is called right after we reconcile an array (or iterator) of child
  // fibers, because that's the only place where we know how many children in
  // the whole set without doing extra work later, or storing addtional
  // information on the fiber.
  //
  // That's why this function is separate from pushTreeId — it's called during
  // the render phase of the fork parent, not the child, which is where we push
  // the other context values.
  //
  // In the Fizz implementation this is much simpler because the child is
  // rendered in the same callstack as the parent.
  //
  // It might be better to just add a `forks` field to the Fiber type. It would
  // make this module simpler.

  warnIfNotHydrating();

  forkStack[forkStackIndex++] = treeForkCount;
  forkStack[forkStackIndex++] = treeForkProvider;

  treeForkProvider = workInProgress;
  treeForkCount = totalChildren;
}
```
`pushTreeFork` 会在 `reconcileChildrenArray` 和 `reconcileChildrenIterator` 中调用，


```ts
export function pushTreeId(
  workInProgress: Fiber,
  totalChildren: number,
  index: number,
) {
  warnIfNotHydrating();

  idStack[idStackIndex++] = treeContextId;
  idStack[idStackIndex++] = treeContextOverflow;
  idStack[idStackIndex++] = treeContextProvider;

  treeContextProvider = workInProgress;

  // eg1: treeContextId = 0b101001000
  // eg2: treeContextId = 0b101001000(0*20)
  const baseIdWithLeadingBit = treeContextId;
  const baseOverflow = treeContextOverflow;

  // The leftmost 1 marks the end of the sequence, non-inclusive. It's not part
  // of the id; we use it to account for leading 0s.
  // eg1: baseLength = 9 - 1 = 8
  // eg2: baseLength = 29 - 1 = 28
  const baseLength = getBitLength(baseIdWithLeadingBit) - 1;
  // eg1: baseId = 0b001001000 = 0b01001000
  // eg2: baseId = 0b001001000(0*20) = 0b01001000(0*20)
  const baseId = baseIdWithLeadingBit & ~(1 << baseLength);

  // eg1: slot = 0b10 + 1 = 3 = Ob11
  // eg2: slot = 0b10 + 1 = 3 = Ob11
  const slot = index + 1;
  // eg1: length = getBitLength(0b10000) + 8 = 5 + 8 = 13
  // eg2: length = getBitLength(0b10000) + 28 = 5 + 28 = 33
  const length = getBitLength(totalChildren) + baseLength;

  // 30 is the max length we can store without overflowing, taking into
  // consideration the leading 1 we use to mark the end of the sequence.
  if (length > 30) {
    // We overflowed the bitwise-safe range. Fall back to slower algorithm.
    // This branch assumes the length of the base id is greater than 5; it won't
    // work for smaller ids, because you need 5 bits per character.
    //
    // We encode the id in multiple steps: first the base id, then the
    // remaining digits.
    //
    // Each 5 bit sequence corresponds to a single base 32 character. So for
    // example, if the current id is 23 bits long, we can convert 20 of those
    // bits into a string of 4 characters, with 3 bits left over.
    //
    // First calculate how many bits in the base id represent a complete
    // sequence of characters.
    // eg2: numberOfOverflowBits = 28 - (28 % 5) = 25
    // 每 5 位二进制位表示一个 32 位，numberOfOverflowBits 为可以完整转换 32 位的二进制位的数量（从右向左）
    const numberOfOverflowBits = baseLength - (baseLength % 5);

    // Then create a bitmask that selects only those bits.
    // eg2: newOverflowBits = 0b(1*25)
    const newOverflowBits = (1 << numberOfOverflowBits) - 1;

    // Select the bits, and convert them to a base 32 string.
    // eg2: newOverflow = (0b01001000(0*20) & 0b(1*25)).toString(32) = (0b01000(0*20)).toString(32)
    // 即保留右 25 位
    const newOverflow = (baseId & newOverflowBits).toString(32);

    // Now we can remove those bits from the base id.
    // eg2: restOfBaseId = 0b01001000(0*20) >> 25 = 0b010
    // 即删除右 25 位
    const restOfBaseId = baseId >> numberOfOverflowBits;
    // eg2: restOfBaseLength = 28 - 25 = 3
    // restOfBaseLength 为不能完整转换 32 位的二进制位数，即 5 的余数
    const restOfBaseLength = baseLength - numberOfOverflowBits;

    // Finally, encode the rest of the bits using the normal algorithm. Because
    // we made more room, this time it won't overflow.
    // eg2: restOfLength = getBitLength(0b10000) + 1 = 5 + 1 = 6
    const restOfLength = getBitLength(totalChildren) + restOfBaseLength;
    // eg2: restOfNewBits = Ob11 << 3 = 0b11000
    const restOfNewBits = slot << restOfBaseLength;
    // eg2: id = 0b11000 | 0b010 = 0b11010
    // 即 5 的余数位的二进制再在左边 + 新的 slot 表示的二进制
    const id = restOfNewBits | restOfBaseId;
    // eg2: 新的溢出 = treeContextId 从右到左以5为单位的二进制位转换的32位字符串 + 原有的溢出
    const overflow = newOverflow + baseOverflow;
    // eg2: treeContextId = (1 << 6) | 0b11010 = 0b111010  
    // (1 << restOfLength) 只是为了补0，最左位没有实际含义，使用时会去除这一位
    treeContextId = (1 << restOfLength) | id;
    treeContextOverflow = overflow;
  } else {
    // Normal path
    // eg1: newBits = Ob11 << 8 
    const newBits = slot << baseLength;
    // eg1: id = (Ob11 << 8) | 0b01001000 => 即将 newBits 放到 baseId 左位
    const id = newBits | baseId;
    // eg1: 没有新的溢出
    const overflow = baseOverflow;

    // 同上一个条件分支，(1 << length) 只是为了保留补零位
    treeContextId = (1 << length) | id;
    treeContextOverflow = overflow;
  }
}
```

`pushTreeId` 会在 `beginWork` 以及 `pushMaterializedTreeId` 中被调用，而 `pushMaterializedTreeId` 会在 `updateForwardRef` 和 `updateFunctionComponent` 以及 `mountIndeterminateComponent` 中被调用

即在 render 的深度遍历的‘递’过程，调用 `pushTreeId`（会跳过同级非数组/迭代类型的 fiber 节点，比如只有一个子节点），对应的 ‘归’ 过程，调用 `popTreeContext`

```ts
export function pushMaterializedTreeId(workInProgress: Fiber) {
  warnIfNotHydrating();

  // This component materialized an id. This will affect any ids that appear
  // in its children.
  const returnFiber = workInProgress.return;
  if (returnFiber !== null) {
    const numberOfForks = 1;
    const slotIndex = 0;
    pushTreeFork(workInProgress, numberOfForks);
    pushTreeId(workInProgress, numberOfForks, slotIndex);
  }
}
```

```ts
export function popTreeContext(workInProgress: Fiber) {
  // Restore the previous values.

  // This is a bit more complicated than other context-like modules in Fiber
  // because the same Fiber may appear on the stack multiple times and for
  // different reasons. We have to keep popping until the work-in-progress is
  // no longer at the top of the stack.

  while (workInProgress === treeForkProvider) {
    treeForkProvider = forkStack[--forkStackIndex];
    forkStack[forkStackIndex] = null;
    treeForkCount = forkStack[--forkStackIndex];
    forkStack[forkStackIndex] = null;
  }

  while (workInProgress === treeContextProvider) {
    treeContextProvider = idStack[--idStackIndex];
    idStack[idStackIndex] = null;
    treeContextOverflow = idStack[--idStackIndex];
    idStack[idStackIndex] = null;
    treeContextId = idStack[--idStackIndex];
    idStack[idStackIndex] = null;
  }
}
```

`popTreeContext` 会在 `completeWork`、`unwindWork`、`unwindInterruptedWork` 中被调用


```ts
export function restoreSuspendedTreeContext(
  workInProgress: Fiber,
  suspendedContext: TreeContext,
) {
  warnIfNotHydrating();

  idStack[idStackIndex++] = treeContextId;
  idStack[idStackIndex++] = treeContextOverflow;
  idStack[idStackIndex++] = treeContextProvider;

  treeContextId = suspendedContext.id;
  treeContextOverflow = suspendedContext.overflow;
  treeContextProvider = workInProgress;
}
```

`restoreSuspendedTreeContext` 会在 `reenterHydrationStateFromDehydratedSuspenseInstance` 中被调用

## useTransition
### mountTransition

```ts
function mountTransition(): [
  boolean,
  (callback: () => void, options?: StartTransitionOptions) => void,
] {
  const [isPending, setPending] = mountState(false);
  // The `start` method never changes.
  const start = startTransition.bind(null, setPending);
  const hook = mountWorkInProgressHook();
  hook.memoizedState = start;
  return [isPending, start];
}
```

```ts
function startTransition(setPending, callback, options) {
  const previousPriority = getCurrentUpdatePriority();
  setCurrentUpdatePriority(
    higherEventPriority(previousPriority, ContinuousEventPriority),
  );

  // 此处的更新优先级没有变
  setPending(true);

  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = {};
  const currentTransition = ReactCurrentBatchConfig.transition;

  try {
    // 此时优先级对应的 lane 已改为 TransitionLaneX，
    // TODO (待进一步确认): 如果之前是 defaultLane，则此时已有任务在调度中，这里不会产生新的任务，这里产生的 update 对象和之前的 update 对象都已经放入 updateQueue，但在 updateReducer 中会根据 renderLanes 选择对应优先级的 update 对象算出新的 state，低优先级（比如 Transition）的会被跳过
    setPending(false);
    callback();
  } finally {
    setCurrentUpdatePriority(previousPriority);

    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```
