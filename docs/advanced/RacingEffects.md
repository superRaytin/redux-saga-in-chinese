## 在多个 Effects 之间启动 race

有时候我们同时启动多个任务，但又不想等待所有任务完成，我们只希望拿到 *胜利者*：即第一个被 resolve（或 reject）的任务。
`race` Effect 提供了一个方法，在多个 Effects 之间触发一个竞赛（race）。

下面的示例演示了触发一个远程的获取请求，并且限制了 1 秒内响应，否则作超时处理。

```javascript
import { race, call, put } from 'redux-saga/effects'
import { delay } from 'redux-saga'

function* fetchPostsWithTimeout() {
  const {posts, timeout} = yield race({
    posts: call(fetchApi, '/posts'),
    timeout: call(delay, 1000)
  })

  if (posts)
    put({type: 'POSTS_RECEIVED', posts})
  else
    put({type: 'TIMEOUT_ERROR'})
}
```

`race` 的另一个有用的功能是，它会自动取消那些失败的 Effects。例如，假设我们有 2 个 UI 按钮：

- 第一个用于在后台启动一个任务，这个任务运行在一个无限循环的 `while(true)` 中（例如：每 x 秒钟从服务器上同步一些数据）

- 一旦该后台任务启动了，我们启用第二个按钮，这个按钮用于取消该任务。


```javascript
import { race, take, call } from 'redux-saga/effects'

function* backgroundTask() {
  while (true) { ... }
}

function* watchStartBackgroundTask() {
  while (true) {
    yield take('START_BACKGROUND_TASK')
    yield race({
      task: call(backgroundTask),
      cancel: take('CANCEL_TASK')
    })
  }
}
```

在 `CANCEL_TASK` action 被发起的情况下，`race` Effect 将自动取消 `backgroundTask`，并在 `backgroundTask` 中抛出一个取消错误。
