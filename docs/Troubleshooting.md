# 问题解答

### 添加 saga 后应用程序被卡住了

确保你在 Generator 函数里 `yield` effect。

参考下面这个示例：

```js
import { take } from 'redux-saga/effects'

function* logActions() {
  while (true) {
    const action = take() // 错误
    console.log(action)
  }
}
```

这将使应用程序进入一个无限循环，因为 `take()` 只创建了一条 effect 描述信息。
除非将它 `yield` 给 middleware 去执行，否则上面的 `while` 循环将会和普通的 `while` 循环行为一样，卡住你的应用程序。

增加 `yield` 会暂停 Generator 并将控制返回给执行这个 effect 的 Redux Saga middleware。
在 `take()` 的情况中，Redux Saga 将等待下一个与表达式匹配的 action，并恢复 Generator。

为了修复上面的示例，直接 `yield` `take()` 返回的 effect：

```js
import { take } from 'redux-saga/effects'

function* logActions() {
  while (true) {
    const action = yield take() // 正确
    console.log(action)
  }
}
```

### Saga 忽略了一些 dispatch 的 action

确保 Saga 没有被某些 effect 阻塞。当 Saga 正在等待一个 effect resolve 时，它不能 take 被 dispatch 的 action，直到 effect 被 resolve。

举个例子：

```javascript
function* watchRequestActions() {
  while (true) {
    const {url, params} = yield take('REQUEST')
    yield call(handleRequestAction, url, params) // Saga 将在这阻塞
  }
}

function* handleRequestAction(url, params) {
  const response = yield call(someRemoteApi, url, params)
  yield put(someAction(response))
}
```

当 `watchRequestActions` 执行 `yield call(handleRequestAction, url, params)` 时，它会在继续下一个 `yield take` 之前，等待 `handleRequestAction` 直到它结束并返回。
举个例子，假设我们有这样一个事件序列：

```
UI                     watchRequestActions             handleRequestAction
-----------------------------------------------------------------------------
.......................take('REQUEST').......................................
dispatch(REQUEST)......call(handleRequestAction).......call(someRemoteApi)... 等待服务器返回.
.............................................................................
.............................................................................
dispatch(REQUEST)............................................................ Action 被忽略了!!
.............................................................................
.............................................................................
.......................................................put(someAction).......
.......................take('REQUEST')....................................... saga 恢复
```

如上所示，当 Saga 被一个 **阻塞调用（blocking call）** 阻塞了，在阻塞解除（译注：即例子中的 put(someAction)）之前，它将忽略所有 dispatch 的 action。

为避免阻塞 Saga，你可以使用一个 **无阻塞调用（non-blocking call）**，即：用 `fork` 来代替 `call`。

```javascript
function* watchRequestActions() {
  while (true) {
    const {url, params} = yield take('REQUEST')
    yield fork(handleRequestAction, url, params) // Saga 将会立即执行
  }
}
```
