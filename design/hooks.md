### 以 useState 为例
1. 函数组件第一次调用 (`mount`) 时，`useState` 对应的是 `mountState` 返回 `[初始值, dispatchAction]`
2. 调用 `dispatchAction` 会创建一个 `update` 对象放入 `hook.queue.pending` 末尾，然后调用 `scheduleUpdateOnFiber` 开始 `render` 过程，
3. render 过程中再次调用函数组件，再次调用 `useState`，此时对应的是 `updateState`，内部调用 `updateReducer`，根据 `hook.baseQueue` 和 
`hook.queue.pending` 更新状态
