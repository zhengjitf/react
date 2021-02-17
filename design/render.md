## render 阶段
render 阶段开始于 `performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot` 方法的调用

### **“递”阶段**
首先从rootFiber开始向下深度优先遍历。为遍历到的每个Fiber节点调用 `beginWork` 方法 (opens new window)。

该方法会根据传入的Fiber节点创建子Fiber节点，并将这两个Fiber节点连接起来。

当遍历到叶子节点（即没有子组件的组件）时就会进入“归”阶段

### **“归”阶段**
在“归”阶段会调用 `completeWork` (opens new window)处理Fiber节点。

当某个 Fiber 节点执行完 `completeWork` ，如果其存在兄弟 Fiber 节点（即 `fiber.sibling !== null` ），会进入其兄弟 Fiber 的“递”阶段。

如果不存在兄弟 Fiber，会进入父级 Fiber 的“归”阶段。

“递”和“归”阶段会交错执行直到“归”到 `rootFiber`。至此，`render` 阶段的工作就结束了

## 组件什么时候 render ？
更新组件时，当满足如下条件时，会跳过 `render`，直接复用之前的 `fiber`：（注：这些判断逻辑在 `beginWork` ）

- `oldProps === newProps`
- `context` 值未变
- `workInProgress.type === current.type`
- `!includesSomeLane(renderLanes, updateLanes)`：不包含更新或更新优先级不够

而对于类组件满足以下条件时，也会跳过 `render`：（注：这些判断逻辑在 `updateClassInstance` 和 `finishClassComponent` 中）
- 如果声明了 `shouldComponentUpdate` 则直接根据返回值判定是否需要 `render`
- 如果是 `PureReactComponent` 组件，则判断 `oldProps` 与 `newProps` 浅比较是否相等以及`oldState` 与 `newState` 浅比较相等，并且没有调用 `forceUpdate`
