## 架构
React16架构可以分为三层：

- Scheduler（调度器）—— 调度任务的优先级，高优任务优先进入 Reconciler
- Reconciler（协调器）—— 对应 `render` 阶段，会调用组件的 `render` 方法，负责找出变化的组件
- Renderer（渲染器）—— 如 `react-dom`，对应 `commit` 阶段, 负责将变化的组件渲染到页面上


