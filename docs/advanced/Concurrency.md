## 并发

在基础知识部分，我们看到了如何使用辅助函数 `takeEvery` 和 `takeLatest` effect 来管理 Effects 之间的并发。

在本节中，我们将看到如何使用低阶 Effects 来实现那些辅助函数。

## `takeEvery`

```javascript
import {fork, take} from "redux-saga/effects"

const takeEvery = (pattern, saga, ...args) => fork(function*() {
  while (true) {
    const action = yield take(pattern)
    yield fork(saga, ...args.concat(action))
  }
})
```

`takeEvery` 可以让多个 `saga` 任务并行被 fork 执行。

## `takeLatest`

```javascript
import {cancel, fork, take} from "redux-saga/effects"

const takeLatest = (pattern, saga, ...args) => fork(function*() {
  let lastTask
  while (true) {
    const action = yield take(pattern)
    if (lastTask) {
      yield cancel(lastTask) // 如果任务已经结束，则 cancel 为空操作
    }
    lastTask = yield fork(saga, ...args.concat(action))
  }
})
```

`takeLatest` 不允许多个 `saga` 任务并行地执行。一旦接收到新的发起的 action，它就会取消前面所有 fork 过的任务（如果这些任务还在执行的话）。

在处理 AJAX 请求的时候，如果我们只希望获取最后那个请求的响应，`takeLatest` 就会非常有用。
