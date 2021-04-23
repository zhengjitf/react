[官方相关 issue](https://github.com/facebook/react/pull/18796)


#### 更新优先级标识
- [ReactPriorityLevel](../packages/react-reconciler/src/ReactInternalTypes.js)
- [SchedulerPriorityLevel](../packages/scheduler/src/SchedulerPriorities.js)
- [LanePriority](../packages/react-reconciler/src/ReactFiberLane.new.js)

**互相转换**：
1. [`ReactPriorityLevel` -> `SchedulerPriorityLevel`](../packages/react-reconciler/src/SchedulerWithReactIntegration.new.js)

```ts
function reactPriorityToSchedulerPriority(reactPriorityLevel) {
  switch (reactPriorityLevel) {
    case ImmediatePriority:
      return Scheduler_ImmediatePriority;
    case UserBlockingPriority:
      return Scheduler_UserBlockingPriority;
    case NormalPriority:
      return Scheduler_NormalPriority;
    case LowPriority:
      return Scheduler_LowPriority;
    case IdlePriority:
      return Scheduler_IdlePriority;
    default:
      invariant(false, 'Unknown priority level.');
  }
}
```

2. [`ReactPriorityLevel` -> `LanePriority`]()

```ts
function schedulerPriorityToLanePriority(
  schedulerPriorityLevel: ReactPriorityLevel,
): LanePriority {
  switch (schedulerPriorityLevel) {
    case ImmediateSchedulerPriority:
      return SyncLanePriority;
    case UserBlockingSchedulerPriority:
      return InputContinuousLanePriority;
    case NormalSchedulerPriority:
    case LowSchedulerPriority:
      // TODO: Handle LowSchedulerPriority, somehow. Maybe the same lane as hydration.
      return DefaultLanePriority;
    case IdleSchedulerPriority:
      return IdleLanePriority;
    default:
      return NoLanePriority;
  }
}
```

3. [`LanePriority` -> `ReactPriorityLevel`]()
```ts
function lanePriorityToSchedulerPriority(
  lanePriority: LanePriority,
): ReactPriorityLevel {
  switch (lanePriority) {
    case SyncLanePriority:
    case SyncBatchedLanePriority:
      return ImmediateSchedulerPriority;
    case InputDiscreteHydrationLanePriority:
    case InputDiscreteLanePriority:
    case InputContinuousHydrationLanePriority:
    case InputContinuousLanePriority:
      return UserBlockingSchedulerPriority;
    case DefaultHydrationLanePriority:
    case DefaultLanePriority:
    case TransitionHydrationPriority:
    case TransitionPriority:
    case SelectiveHydrationLanePriority:
    case RetryLanePriority:
      return NormalSchedulerPriority;
    case IdleHydrationLanePriority:
    case IdleLanePriority:
    case OffscreenLanePriority:
      return IdleSchedulerPriority;
    case NoLanePriority:
      return NoSchedulerPriority;
    default:
      invariant(
        false,
        'Invalid update priority: %s. This is a bug in React.',
        lanePriority,
      );
  }
}
```

#### 优先级分类
- 事件优先级


##### 事件优先级：
不同事件对应不同的 `LanePriority`

```ts
function getEventPriority(domEventName: DOMEventName): LanePriority {
  // ...
}
```

#### 计算优先级
```ts
export function findUpdateLane(lanePriority: LanePriority): Lane {
  switch (lanePriority) {
    case NoLanePriority:
      break;
    case SyncLanePriority:
      return SyncLane;
    case SyncBatchedLanePriority:
      return SyncBatchedLane;
    case InputDiscreteLanePriority:
      return SyncLane;
    case InputContinuousLanePriority:
      return InputContinuousLane;
    case DefaultLanePriority:
      return DefaultLane;
    case TransitionPriority: // Should be handled by findTransitionLane instead
    case RetryLanePriority: // Should be handled by findRetryLane instead
      break;
    case IdleLanePriority:
      return IdleLane;
    default:
      // The remaining priorities are not valid for updates
      break;
  }
}
```

```ts
function requestUpdateLane(fiber: Fiber): Lane {
  // Special cases
  const mode = fiber.mode;
  if ((mode & BlockingMode) === NoMode) {
    return (SyncLane: Lane);
  } else if ((mode & ConcurrentMode) === NoMode) {
    return getCurrentUpdateLanePriority() === SyncLanePriority
      ? (SyncLane: Lane)
      : (SyncBatchedLane: Lane);
  } else if (
    !deferRenderPhaseUpdateToNextBatch &&
    (executionContext & RenderContext) !== NoContext &&
    workInProgressRootRenderLanes !== NoLanes
  ) {
    // This is a render phase update. These are not officially supported. The
    // old behavior is to give this the same "thread" (expiration time) as
    // whatever is currently rendering. So if you call `setState` on a component
    // that happens later in the same render, it will flush. Ideally, we want to
    // remove the special case and treat them as if they came from an
    // interleaved event. Regardless, this pattern is not officially supported.
    // This behavior is only a fallback. The flag only exists until we can roll
    // out the setState warning, since existing code might accidentally rely on
    // the current behavior.
    return pickArbitraryLane(workInProgressRootRenderLanes);
  }

  // The algorithm for assigning an update to a lane should be stable for all
  // updates at the same priority within the same event. To do this, the inputs
  // to the algorithm must be the same. For example, we use the `renderLanes`
  // to avoid choosing a lane that is already in the middle of rendering.
  //
  // However, the "included" lanes could be mutated in between updates in the
  // same event, like if you perform an update inside `flushSync`. Or any other
  // code path that might call `prepareFreshStack`.
  //
  // The trick we use is to cache the first of each of these inputs within an
  // event. Then reset the cached values once we can be sure the event is over.
  // Our heuristic for that is whenever we enter a concurrent work loop.
  //
  // We'll do the same for `currentEventTransitionLane` below.
  if (currentEventWipLanes === NoLanes) {
    currentEventWipLanes = workInProgressRootIncludedLanes;
  }

  const isTransition = requestCurrentTransition() !== NoTransition;
  if (isTransition) {
    if (currentEventTransitionLane === NoLane) {
      currentEventTransitionLane = claimNextTransitionLane();
    }
    return currentEventTransitionLane;
  }

  // TODO: Remove this dependency on the Scheduler priority.
  // To do that, we're replacing it with an update lane priority.
  const schedulerPriority = getCurrentPriorityLevel();

  // Find the correct lane based on priorities. Ideally, this would just be
  // the update lane priority, but for now we're also checking for discrete
  // updates and falling back to the scheduler priority.
  let lane;
  if (
    // TODO: Temporary. We're removing the concept of discrete updates.
    (executionContext & DiscreteEventContext) !== NoContext &&
    schedulerPriority === UserBlockingSchedulerPriority
  ) {
    lane = findUpdateLane(InputDiscreteLanePriority);
  } else if (getCurrentUpdateLanePriority() !== NoLanePriority) {
    const currentLanePriority = getCurrentUpdateLanePriority();
    lane = findUpdateLane(currentLanePriority);
  } else {
    const eventLanePriority = getCurrentEventPriority();
    lane = findUpdateLane(eventLanePriority);
  }

  return lane;
}
```


#### 使用优先级
1. `scheduleCallback`

以一个优先级注册 `callback`，在适当的时机执行

```ts
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();
 
  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }
 
  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }
 
  var expirationTime = startTime + timeout;
 
  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    // 用于小顶堆排序时对元素进行比较的标识
    sortIndex: -1,
  };
 
  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);

    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }
 
  return newTask;
}
```

2. reconciler `runWithPriority`
```ts
function runWithPriority<T>(priority: LanePriority, fn: () => T) {
  const previousPriority = getCurrentUpdateLanePriority();
  try {
    setCurrentUpdateLanePriority(priority);
    return fn();
  } finally {
    setCurrentUpdateLanePriority(previousPriority);
  }
}
```

3. schedule `runWithPriority`

以一个优先级执行 `callback`，如果是同步的任务，优先级就是 `ImmediateSchedulerPriority`

```ts
function unstable_runWithPriority(priorityLevel, eventHandler) {
  switch (priorityLevel) {
    case ImmediatePriority:
    case UserBlockingPriority:
    case NormalPriority:
    case LowPriority:
    case IdlePriority:
      break;
    default:
      priorityLevel = NormalPriority;
  }

  var previousPriorityLevel = currentPriorityLevel;
  currentPriorityLevel = priorityLevel;

  try {
    return eventHandler();
  } finally {
    currentPriorityLevel = previousPriorityLevel;
  }
}
```


```
ReactDOM.createRoot(xx) 创建 tag 为 ConcurrentRoot 的 fiberRoot
    |
    |
    v
.render(xx) 调用 updateContainer:
    |
    |
    v 
用 tag 对应生成的 lane，调用 scheduleUpdateOnFiber
```

```
performConcurrentWorkOnRoot >> (renderRootConcurrent -> ensureRootIsScheduled)
```

#### 和同步任务相关的方法

**flushSync**：
`ReactDOM` 模块暴露的方法，用于 `concurrent` 模式下，提高渲染优先级？

```ts
export function flushSync<A, R>(fn: A => R, a: A): R {
  const prevExecutionContext = executionContext;
  if ((prevExecutionContext & (RenderContext | CommitContext)) !== NoContext) {
    return fn(a);
  }
  executionContext |= BatchedContext;

  const previousLanePriority = getCurrentUpdateLanePriority();
  try {
    setCurrentUpdateLanePriority(SyncLanePriority);
    if (fn) {
      return fn(a);
    } else {
      return (undefined: $FlowFixMe);
    }
  } finally {
    setCurrentUpdateLanePriority(previousLanePriority);
    executionContext = prevExecutionContext;
    // Flush the immediate callbacks that were scheduled during this batch.
    // Note that this will happen even if batchedUpdates is higher up
    // the stack.
    flushSyncCallbackQueue();
  }
}
```

**flushSyncCallbackQueue**：
立即调用同步任务（`syncQueue`）

```ts
function flushSyncCallbackQueueImpl() {
  if (!isFlushingSyncQueue && syncQueue !== null) {
    // Prevent re-entrancy.
    isFlushingSyncQueue = true;
    let i = 0;
    const previousLanePriority = getCurrentUpdateLanePriority();
    try {
      const isSync = true;
      const queue = syncQueue;
      setCurrentUpdateLanePriority(SyncLanePriority);
      for (; i < queue.length; i++) {
        let callback = queue[i];
        do {
          callback = callback(isSync);
        } while (callback !== null);
      }
      syncQueue = null;
    } catch (error) {
      // If something throws, leave the remaining callbacks on the queue.
      if (syncQueue !== null) {
        syncQueue = syncQueue.slice(i + 1);
      }
      // Resume flushing in the next tick
      Scheduler_scheduleCallback(
        Scheduler_ImmediatePriority,
        flushSyncCallbackQueue,
      );
      throw error;
    } finally {
      setCurrentUpdateLanePriority(previousLanePriority);
      isFlushingSyncQueue = false;
    }
  }
}
```

#### 和异步任务相关的方法
**unstable_scheduleCallback**：

```ts
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }

  var expirationTime = startTime + timeout;

  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }

  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      /** requestHostCallback 负责调度 */
      /** flushWork 负责处理任务：
       *    1. 将 timerQueue 中到期的 task 放入 taskQueue 
       *    2. 循环 taskQueue ，如果取出的 task 未过期且当前渲染周期没有剩余时间时，跳出循环，否则执行该 task 并从队列推出 
       */
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

```bash
scheduleUpdateOnFiber
  |
  |
  v
# 标记 root 有待处理的更新：root.pendingLanes |= lane
markRootUpdated(root, lane, eventTime);
```

一些关键方法：
- `getNextLanes`
- `markStarvedLanesAsExpired`


#### lane 的使用路径（以 setState 更新函数组件为例）
```
调用 setState (触发 dispatchAction)
  |
  |
  v
创建 update 对象 （update 包含的 lane 信息通过 requestUpdateLane 计算得出）
```
