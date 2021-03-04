## context

### mount 流程

### update 流程

```
更新 Provider value

    |
    |
    v

调度更新

    |
    |
    v

render 阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）
        |
    |   | 
    |    —— ——> 遇到 Provider 调用 updateContextProvider； 遇到 Consumer 调用 updateContextConsumer
    |
    v

commit阶段（`commitRoot`）
```


### 创建 context 对象
```ts
export function createContext<T>(
  defaultValue: T,
  calculateChangedBits: ?(a: T, b: T) => number,
): ReactContext<T> {
  // ....

  const context: ReactContext<T> = {
    $$typeof: REACT_CONTEXT_TYPE,
    _calculateChangedBits: calculateChangedBits,
    // As a workaround to support multiple concurrent renderers, we categorize
    // some renderers as primary and others as secondary. We only expect
    // there to be two concurrent renderers at most: React Native (primary) and
    // Fabric (secondary); React DOM (primary) and React ART (secondary).
    // Secondary renderers store their context values on separate fields.
    _currentValue: defaultValue,
    _currentValue2: defaultValue,
    // Used to track how many concurrent renderers this context currently
    // supports within in a single renderer. Such as parallel server rendering.
    _threadCount: 0,
    // These are circular
    Provider: (null as any),
    Consumer: (null as any),
  };

  context.Provider = {
    $$typeof: REACT_PROVIDER_TYPE,
    _context: context,
  };

  context.Consumer = context;

  return context;
}
```
从源码可以看出 `context.Consumer` 其实就是 `context` 本身

### Provider
`render` 阶段对 `Provider` 的处理在 `updateContextProvider` 中：

1. 判断 `value` 是否变化
2. 如果没变化，且 `props.children` 未变化，则直接跳出，复用下级 fiber （从 `current fiber` 树）
3. 否则往下遍历寻找所有包含该 `context` 依赖的 `fiber` 节点（`fiber.dependencies` 为依赖的 `context` 列表），更新其 `lanes` 和 `dependencies.lanes`（即标记需要更新），以及循环往上更新`fiber.childLanes`（遍历过程中会通过该属性判断下级是否包含更新），如果是 `ClassComponent`，为其添加一个 `tag` 为 `ForceUpdate` 的 `update` 对象
4. 然后继续 `render` 过程 (`reconcileChildren`)

```ts
function updateContextProvider(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  const providerType: ReactProviderType<any> = workInProgress.type;
  const context: ReactContext<any> = providerType._context;

  const newProps = workInProgress.pendingProps;
  const oldProps = workInProgress.memoizedProps;

  const newValue = newProps.value;

  pushProvider(workInProgress, context, newValue);

  if (oldProps !== null) {
    const oldValue = oldProps.value;
    const changedBits = calculateChangedBits(context, newValue, oldValue);
    if (changedBits === 0) {
      // No change. Bailout early if children are the same.
      // 如果 Provider 的 value 未变化，则直接跳出
      if (
        oldProps.children === newProps.children &&
        !hasLegacyContextChanged()
      ) {
        return bailoutOnAlreadyFinishedWork(
          current,
          workInProgress,
          renderLanes,
        );
      }
    } else {
      // The context value changed. Search for matching consumers and schedule
      // them to update.
      // context 的更新处理：
      // 1. 从当前 fiber （Provider）的子节点开始往下遍历寻找所有含有该 context 依赖的 fiber 节点
      // 2. 更新这些 fiber 的 lanes 和 dependencies.lanes，以及循环往上更新 fiber 的 childLanes
      // 3. 如果是 ClassComponent，添加一个 tag 为 ForceUpdate 的 update 对象
      propagateContextChange(workInProgress, context, changedBits, renderLanes);
    }
  }

  const newChildren = newProps.children;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
}
```

### Consumer
`render` 阶段对 `Consumer` 的处理在 `updateContextConsumer` 中：

```ts
function updateContextConsumer(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  let context: ReactContext<any> = workInProgress.type;

  const newProps = workInProgress.pendingProps;
  // render props
  const render = newProps.children;

  prepareToReadContext(workInProgress, renderLanes);
  const newValue = readContext(context, newProps.unstable_observedBits);
  let newChildren = render(newValue);

  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
}

function prepareToReadContext(
  workInProgress: Fiber,
  renderLanes: Lanes,
): void {
  currentlyRenderingFiber = workInProgress;
  lastContextDependency = null;
  lastContextWithAllBitsObserved = null;

  const dependencies = workInProgress.dependencies;
  if (dependencies !== null) {
    const firstContext = dependencies.firstContext;
    if (firstContext !== null) {
      if (includesSomeLane(dependencies.lanes, renderLanes)) {
        // Context list has a pending update. Mark that this fiber performed work.
        markWorkInProgressReceivedUpdate();
      }
      // Reset the work-in-progress list
      dependencies.firstContext = null;
    }
  }
}

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
