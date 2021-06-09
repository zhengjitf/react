## Diff

`diff` 核心逻辑在 `reconcileChildFibers` 函数中，根据新的 `children` 的类型进行对应处理：
1. 没有指定 `key` 的 `Fragment`：取其 `props.children` 继续判断
2. 对象类型：可能是 `Element`、`Portal` 或者 `Lazy Element`，分别进行处理
3. 数组类型（最复杂的部分）
4. 迭代类型：类似数组类型的处理
