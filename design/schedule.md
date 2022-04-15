## schedule
Concurrent 模式下更新任务会采用调度的模式，调度方法即 `packages/scheduler/src/forks/Scheduler.js` 导出的 `unstable_scheduleCallback`

```js
function unstable_scheduleCallback(priorityLevel, callback, options) {}
```

`callback` 参数即调用的任务，为 `performConcurrentWorkOnRoot`，调度实现依赖特定环境的特定 api：
- IE and Node.js + jsdom： `setImmediate`
- DOM and Worker environments: `MessageChannel`
- fallback: `setTimeout`

在调度期间的更新会被合并成一次 render，对于现代浏览器，依赖 `MessageChannel` 实现调度，则在 `MessageChannel` 回调前触发的异步更新也会被合并，如 `Promise`。

```js
// 合并成一次 render

setState(1)
setState(2)
```

```js
// 合并成一次 render

setState(1)
Promise.resolve().then(() => {
  setState(2)
})
```

```js
// 会触发两次 render，事件回调使用的是 batchedUpdates

const handleClick = () => {
  setState(1)
  Promise.resolve().then(() => {
    setState(2)
  })
}

<App onClick={handleClick} />
```

