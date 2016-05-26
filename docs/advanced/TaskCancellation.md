# 任务的取消

我们已经在 [无阻塞调用](http://superRaytin.github.io/redux-saga-in-chinese/docs/advanced/NonBlockingCalls.html) 一节中看到了取消任务的示例。
在这节，我们将回顾一下，在一些更加详细的情况下取消的语义。

一旦任务被 fork，可以使用 `yield cancel(task)` 来中止任务执行。取消正在运行的任务，将抛出 `SagaCancellationException` 错误。

来看看它是如何工作的，让我们先考虑一个简单的例子：一个可通过某些 UI 命令启动或停止的后台同步任务。
在接收到 `START_BACKGROUND_SYNC` action 后，我们 fork 一个后台任务，周期性地从远程服务器同步一些数据。

这个任务将会一直执行直到一个 `STOP_BACKGROUND_SYNC` action 被触发。
然后我们取消后台任务，等待下一个 `STOP_BACKGROUND_SYNC` action。

```javascript
import { SagaCancellationException } from 'redux-saga'
import {  take, put, call, fork, cancel } from 'redux-saga/effects'
import actions from 'somewhere'
import { someApi, delay } from 'somewhere'

function* bgSync() {
  try {
    while(true) {
      yield put(actions.requestStart())
      const result = yield call(someApi)
      yield put(actions.requestSuccess(result))
      yield call(delay, 5000)
    }
  } catch(error) {
    // 或直接使用 `isCancelError(error)`
    if(error instanceof SagaCancellationException)
      yield put(actions.requestFailure('Sync cancelled!'))
  }
}

function* main() {
  while( yield take(START_BACKGROUND_SYNC) ) {
    // 启动后台任务
    const bgSyncTask = yield fork(bgSync)

    // 等待用户的停止操作
    yield take(STOP_BACKGROUND_SYNC)
    // 用户点击了停止，取消后台任务
    // 将抛出一个 SagaCancellationException 错误至被 fork 的 bgSync 任务
    yield cancel(bgSyncTask)
  }
}
```

`yield cancel(bgSyncTask)` 将在当前执行的任务中抛出一个 `SagaCancellationException` 类型的异常。
在上面的示例中，异常是由 `bgSync` 引发的。**注意，未被捕获的 `SagaCancellationException` 不会向上冒泡**。
在上面的示例中，如果 `bgSync` 没有捕获取消错误，错误将不会被传播到 `main`（因为 `main` 已经往前进了）。

取消正在执行的任务，也将同时取消被阻塞在当前 Effect 中的任务。

举个例子，假设在应用程序生命周期的某个时刻，还有挂起的（未完成的）调用链：

```javascript
function* main() {
  const task = yield fork(subtask)
  ...
  // later
  yield cancel(task)
}

function* subtask() {
  ...
  yield call(subtask2) // currently blocked on this call
  ...
}

function* subtask2() {
  ...
  yield call(someApi) // currently blocked on this all
  ...
}
```

`yield cancel(task)` 将触发 `subtask` 任务的取消，反过来它将触发 `subtask2` 的取消。
`subtask2` 中将抛出一个 `SagaCancellationException` 错误，然后另一个 `SagaCancellationException` 错误将会在 `subtask` 中抛出。
 如果 `subtask` 没有处理取消异常，一条警告信息将在控制台中打印出来，以警告开发者（如果 `process.env.NODE_ENV` 变量存在，并且它被设置为 `development`，就仅仅会打印日志信息而不是警告信息）。

取消异常的主要目的是让已取消的任务执行任何自定义的清理逻辑，因此，我们不会让应用程序状态不一致。
在上面后台同步的示例中，通过捕获取消异常，`bgSync` 能够发起一个 `requestFailure` action 至 store。
否则，store 可能会变得状态不一致（例如，等待一个被挂起的请求的结果）。

### 注意

记住重要的一点，`yield cancel(task)` 不会等待被取消的任务完成（即执行其 catch 区块）。`cancel` Effect 的行为和 `fork` 有点类似。
一旦取消发起，它就会尽快返回。一旦取消，任务通常应尽快完成它的清理逻辑然后返回。
在某些情况下，清理逻辑可能涉及一些异步操作，但被取消的任务变成了独立的进程，并且没有办法让它重新加入主控制流程（除了通过 Redux store 为其他任务发起 action。
然而，这将导致控制流变的复杂并且难以理解。更好的做法是尽快结束一个已取消的任务）。

## 自动取消

除了手动取消任务，还有一些情况的取消是自动触发的。

1. 在 `race` Effect 中。所有参与竞赛的任务，除了优胜者（译注：最先完成的任务），其他任务都会被取消。

2. 并行的 Effect (`yield [...]`)。一旦其中任何一个任务被拒绝，并行的 Effect 将会被拒绝（受 `Promise.all` 启发）。在这种情况中，所有其他的 Effect 将被自动取消。
