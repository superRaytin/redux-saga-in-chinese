# 使用 Saga 辅助函数

redux-saga 提供了一些辅助函数，用来在一些特定的 action 被发起到 Store 时派生任务。

这些辅助函数构建在低阶 API 之上。我们将会在高级概念一节看到这些函数是如何实现的。

第一个函数，`takeEvery` 是最常见的，它提供了类似 redux-thunk 的行为。

让我们演示一下常见的 AJAX 例子。每次点击 Fetch 按钮时，我们发起一个 `FETCH_REQUESTED` 的 action。
我们想通过启动一个任务从服务器获取一些数据，来处理这个 action。

首先我们创建一个将执行异步 action 的任务：

```javascript
import { call, put } from 'redux-saga/effects'

export function* fetchData(action) {
   try {
      const data = yield call(Api.fetchUser, action.payload.url);
      yield put({type: "FETCH_SUCCEEDED", data});
   } catch (error) {
      yield put({type: "FETCH_FAILED", error});
   }
}
```

然后在每次 `FETCH_REQUESTED` action 被发起时启动上面的任务。

```javascript
import { takeEvery } from 'redux-saga'

function* watchFetchData() {
  yield* takeEvery('FETCH_REQUESTED', fetchData)
}
```

在上面的例子中，`takeEvery` 允许多个 `fetchData` 实例同时启动。在某个特定时刻，我们可以启动一个新的 `fetchData` 任务，
尽管之前还有一个或多个 `fetchData` 尚未结束。

如果我们只想得到最新那个请求的响应（例如，始终显示最新版本的数据）。我们可以使用 `takeLatest` 辅助函数。


```javascript
import { takeLatest } from 'redux-saga'

function* watchFetchData() {
  yield* takeLatest('FETCH_REQUESTED', fetchData)
}
```

和 `takeEvery` 不同，在任何时刻 `takeLatest` 只允许执行一个 `fetchData` 任务。并且这个任务是最后被启动的那个。
如果之前已经有一个任务在执行，那之前的这个任务会自动被取消。

如果你有多重的 Sagas 监听不同的actions，你可以通过这些内建的辅助函数来创建多重的监视器（watcher），这里其实是通过 `fork` 来处理的（我们将会在之后 `fork` 部分讨论，到目前为止，就把它当做是一个 Effect, 它能够让我们在后台启动多重的sagas）

```javascript
import { takeEvery } from 'redux-saga/effects'

// FETCH_USERS
function* fetchUsers(action) { ... }

// CREATE_USER
function* createUser(action) { ... }

// use them in parallel
export default function* rootSaga() {
  yield takeEvery('FETCH_USERS', fetchUsers)
  yield takeEvery('CREATE_USER', createUser)
}
```
