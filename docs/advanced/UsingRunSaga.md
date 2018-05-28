# 连接 Sagas 至外部输入和输出

我们已经看到了，`take` Effect 是通过等待 action 被发起到 Store 来解决（resolved）的。
也看到了 `put` Effect 是通过发起一个 action 来解决的，action 被作为参数传给 `put` Effect。

当 Saga 启动了（不管是初始启动或是稍后动态启动），middleware 会自动将它的 `take` / `put` 连接至 store。
这 2 个 Effect 可以被看作是一种 Saga 的输入/输出（Input/Output）。

`redux-saga` 提供了一种方式在 redux middleware 环境外部运行 Saga，并可以连接至自定义的输入输出（Input/Output）。

```javascript
import { runSaga } from 'redux-saga'

function* saga() { ... }

const myIO = {
  subscribe: ..., // this will be used to resolve take Effects
  dispatch: ...,  // this will be used to resolve put Effects
  getState: ...,  // this will be used to resolve select Effects
}

runSaga(
  myIO,
  saga,
)
```

欲了解更多信息请参见 [API 文档](http://leonshi.com/redux-saga-in-chinese/docs/api/index.html)。
