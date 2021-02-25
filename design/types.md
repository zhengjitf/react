
## [FiberRoot](../packages/react-reconciler/src/ReactInternalTypes.js)

```ts
type FiberRoot = {
  // The type of root (legacy, batched, concurrent, etc.)
  // LegacyRoot, BlockingRoot, ConcurrentRoot
  tag: RootTag;

  // Any additional information from the host associated with this root.
  // container dom 元素，即 ReactDOM.render() 的第二个参数
  containerInfo: any;
  // Used only by persistent updates.
  pendingChildren: any;
  // The currently active root fiber. This is the mutable root of the tree.
  // 指向 rootFiber
  current: Fiber;

  pingCache: WeakMap<Wakeable, Set<mixed>> | Map<Wakeable, Set<mixed>> | null;

  // A finished work-in-progress HostRoot that's ready to be committed.
  finishedWork: Fiber | null;
  // Timeout handle returned by setTimeout. Used to cancel a pending timeout, if
  // it's superseded by a new one.
  timeoutHandle: TimeoutHandle | NoTimeout;
  // Top context object, used by renderSubtreeIntoContainer
  context: Object | null;
  pendingContext: Object | null;
  // Determines if we should attempt to hydrate on the initial mount
  +hydrate: boolean;

  // Used by useMutableSource hook to avoid tearing during hydration.
  mutableSourceEagerHydrationData?: Array<
    MutableSource<any> | MutableSourceVersion,
  > | null;

  // Node returned by Scheduler.scheduleCallback. Represents the next rendering
  // task that the root will work on.
  callbackNode: *;
  callbackPriority: LanePriority;
  eventTimes: LaneMap<number>;
  expirationTimes: LaneMap<number>;

  pendingLanes: Lanes;
  suspendedLanes: Lanes;
  pingedLanes: Lanes;
  expiredLanes: Lanes;
  mutableReadLanes: Lanes;

  finishedLanes: Lanes;

  entangledLanes: Lanes;
  entanglements: LaneMap<Lanes>;
}
```

## Fiber
```ts
type Fiber = {
  // These first fields are conceptually members of an Instance. This used to
  // be split into a separate type and intersected with the other Fiber fields,
  // but until Flow fixes its intersection bugs, we've merged them into a
  // single type.

  // An Instance is shared between all versions of a component. We can easily
  // break this out into a separate object to avoid copying so much to the
  // alternate versions of the tree. We put this on a single object for now to
  // minimize the number of objects created during the initial render.

  // Tag identifying the type of fiber.
  // FunctionComponent, ClassComponent, HostRoot, HostComponent, LazyComponent....
  tag: WorkTag;

  // Unique identifier of this child.
  key: null | string;

  // The value of element.type which is used to preserve the identity during
  // reconciliation of this child.
  elementType: any;

  // The resolved function/class/ associated with this fiber.
  // 除了 LazyComponent 的 `type !== elementType`，其余都相等
  // 对于 FunctionComponent，指函数本身，对于 ClassComponent，指 class，对于HostComponent，指 DOM 节点 tagName
  type: any;

  // The local state associated with this fiber.
  // rootFiber: stateNode = FiberRoot
  // 类组件 Fiber: stateNode = instance
  // 其余 fiber: 真实 DOM 节点
  stateNode: any;

  // Conceptual aliases
  // parent : Instance -> return The parent happens to be the same as the
  // return fiber since we've merged the fiber and instance.

  // Remaining fields belong to Fiber

  // The Fiber to return to after finishing processing this one.
  // This is effectively the parent, but there can be multiple parents (two)
  // so this is only the parent of the thing we're currently processing.
  // It is conceptually the same as the return address of a stack frame.
  // 父级 fiber
  return: Fiber | null;

  // Singly Linked List Tree Structure.
  // 第一个子 fiber
  child: Fiber | null;
  // 兄弟 fiber
  sibling: Fiber | null;
  index: number;

  // The ref last used to attach this node.
  // I'll avoid adding an owner field for prod and model that as functions.
  ref:
    | null
    | (((handle: mixed) => void) & {_stringRef?: string, ...})
    | RefObject;

  // Input is the data coming into process this fiber. Arguments. Props.
  pendingProps: any; // This type will be more specific once we overload the tag.
  memoizedProps: any; // The props used to create the output.

  // A queue of state updates and callbacks.
  updateQueue: 
    | UpdateQueue<any> 
    | FunctionComponentUpdateQueue 
    | HostComponentUpdateQueue;

  // The state used to create the output
  // rootFiber: memoizedState = { element: ** (ReactDOM.render 的第一个参数) }, 
  // 函数组件 fiber: memoizedState = hook
  // 类组件 fiber: 
  memoizedState: any;

  // Dependencies (contexts, events) for this fiber, if it has any
  dependencies: Dependencies | null;

  // Bitfield that describes properties about the fiber and its subtree. E.g.
  // the ConcurrentMode flag indicates whether the subtree should be async-by-
  // default. When a fiber is created, it inherits the mode of its
  // parent. Additional flags can be set at creation time, but after that the
  // value should remain unchanged throughout the fiber's lifetime, particularly
  // before its child fibers are created.
  mode: TypeOfMode;

  // Effect
  flags: Flags; // Placement、Update....
  subtreeFlags: Flags;
  deletions: Array<Fiber> | null;

  // Singly linked list fast path to the next fiber with side-effects.
  nextEffect: Fiber | null;

  // The first and last fiber with side-effect within this subtree. This allows
  // us to reuse a slice of the linked list when we reuse the work done within
  // this fiber.
  firstEffect: Fiber | null;
  lastEffect: Fiber | null;

  // 调度优先级相关
  lanes: Lanes;
  childLanes: Lanes;

  // This is a pooled version of a Fiber. Every fiber that gets updated will
  // eventually have a pair. There are cases when we can clean up pairs to save
  // memory if we need to.
  // workInProgress fiber
  alternate: Fiber | null;

  // Time spent rendering this Fiber and its descendants for the current update.
  // This tells us how well the tree makes use of sCU for memoization.
  // It is reset to 0 each time we render and only updated when we don't bailout.
  // This field is only set when the enableProfilerTimer flag is enabled.
  actualDuration?: number;

  // If the Fiber is currently active in the "render" phase,
  // This marks the time at which the work began.
  // This field is only set when the enableProfilerTimer flag is enabled.
  actualStartTime?: number;

  // Duration of the most recent render time for this Fiber.
  // This value is not updated when we bailout for memoization purposes.
  // This field is only set when the enableProfilerTimer flag is enabled.
  selfBaseDuration?: number;

  // Sum of base times for all descendants of this Fiber.
  // This value bubbles up during the "complete" phase.
  // This field is only set when the enableProfilerTimer flag is enabled.
  treeBaseDuration?: number;

  // Conceptual aliases
  // workInProgress : Fiber ->  alternate The alternate used for reuse happens
  // to be the same as work in progress.
  // __DEV__ only
  _debugID?: number;
  _debugSource?: Source | null;
  _debugOwner?: Fiber | null;
  _debugIsCurrentlyTiming?: boolean;
  _debugNeedsRemount?: boolean;

  // Used to verify that the order of hooks does not change between renders.
  _debugHookTypes?: Array<HookType> | null;
}

type Update<State> = {
  // TODO: Temporary field. Will remove this by storing a map of
  // transition -> event time on the root.
  eventTime: number
  lane: Lane

  tag: UpdateState | ReplaceState | ForceUpdate | CaptureUpdate
  // ClassComponent: 调用 setState 传递的参数，当为函数时，接受 (prevState, nextProps) 参数
  // HostRoot: payload 为 ReactDOM.render 的第一个传参
  payload: any
  callback: (() => mixed) | null

  next: Update<State> | null
};

export type SharedQueue<State> = {
  pending: Update<State> | null
  interleaved: Update<State> | null
};

type UpdateQueue<State> = {
  baseState: State
  firstBaseUpdate: Update<State> | null
  lastBaseUpdate: Update<State> | null
  shared: SharedQueue<State>
  // 有 callback （this.setState 的第二个参数或者 ReactDom.render 的第三个参数） 的 update 对象列表
  // callback 在 commit 的 layout阶段被执行 （通过调用 commitUpdateQueue）
  effects: Array<Update<State>> | null
};

type Effect = {
  tag: HookFlags
  create: () => (() => void) | void
  destroy: (() => void) | void
  deps: Array<mixed> | null
  // Circular
  next: Effect
};

type FunctionComponentUpdateQueue = {
  lastEffect: Effect | null
};

type PropType = string
type Payload = any
type HostComponentUpdateQueue = [PropType, Payload, PropType, Payload, .....]

type ContextDependency<T> = {
  context: ReactContext<T>,
  observedBits: number,
  next: ContextDependency<mixed> | null,
  ...
};

type Dependencies = {
  lanes: Lanes,
  firstContext: ContextDependency<mixed> | null,
  ...
}
```

## ReactElement
```ts
type ReactElement = {
  $$typeof: any,
  // node tag: 'div' or 'span'...
  // React component type: a class or a function
  // React fragment type
  type: any,
  key: any,
  ref: any,
  props: any,
  // ReactFiber
  _owner: any,

  // __DEV__
  _store: {validated: boolean, ...},
  _self: React$Element<any>,
  _shadowChildren: any,
  _source: Source,
}
```




## Hook

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

// UpdateQueue.pending 指向最后一个未处理的 update
// UpdateQueue.pending.next 指向第一个未处理的 update
type UpdateQueue<S, A> = {
  pending: Update<S, A> | null
  interleaved: Update<S, A> | null
  dispatch: (A => mixed) | null
  lastRenderedReducer: ((S, A) => S) | null
  lastRenderedState: S | null
};

type Effect = {
  tag: HookFlags
  create: () => (() => void) | void
  destroy: (() => void) | void
  deps: Array<mixed> | null
  next: Effect
};
```
