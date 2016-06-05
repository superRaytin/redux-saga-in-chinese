# 问题解答

### 添加 saga 后应用程序被卡住了

确保你在 Generator 函数里 `yield` 了 effect。

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

增加 `yield` 会暂停 Generator 并将控制返回给 Redux Saga middleware，然后 middleware 会执行这个 effect。
在 `take()` 的情况中，Redux Saga 将等待下一个与表达式匹配的 action，然后才会恢复 Generator 执行。

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

### 使用多个 `yield* takeEvery/yield* takeLatest` 时，Saga 错过了发起的 action

你可能在同一个 Saga 里运行了多个 `yield*` 语句。

```javascript
function* mySaga() {
  yield* takeEvery(ACTION_1, doSomeWork)
  yield* takeEvery(ACTION_2, doSomeWork)
}
```

要么在不同的 Saga 里分别运行它们。要么使用 `yield [...]` (没有 `*`) 来并行地运行它们。

```javascript
function* mySaga() {
  yield [
    takeEvery(ACTION_1, doSomeWork),
    takeEvery(ACTION_2, doSomeWork)
  ]
}
```

### 解释

`yield*` 用于 *代理（delegate）* 控制权给其他的迭代器。在上面的示例中，第一个 `takeEvery(ACTION_1, doSomeWork)` 返回了一个迭代器对象。
由于结合了 `yield*`，`mySaga` Generator 将代理它所有的 `next()` 调用给返回的迭代器。这意味着 `mySaga` 的任何 `next()` 调用将会转发到 `takeEvery(...)` 迭代器的 `next()`。
并且只有在第一个 `takeEvery(...)` 迭代器完成之后，第二个迭代器 `yield* takeEvery(ACTION_2, doSomeWork)` 的调用才会执行
（由于 `takeEvery(...)` 执行的是一个 `while(true) {...}`，所以第一个迭代器永远不会结束，第二个迭代器的调用永远不会被执行）。

在 `yield [takeEvery(...), ...]` 这样并行的形式中，middleware 将并行运行所有的迭代器。

### Saga 错过了发起的 action

确保 Saga 没有被一些 Effect 阻塞。当 Saga 正在等待 Effect 解决（resolve），这将导致 Saga 无法接收发起的 action，直到 Effect 被解决。

例如，考虑这个例子：

```javascript
function watchRequestActions() {
  while(true) {
    const {url, params} = yield take('REQUEST')
    yield call(handleRequestAction, url, params) // Saga 在此处阻塞
  }
}

function handleRequestAction(url, params) {
  const response = yield call(someRemoteApi, url, params)
  yield put(someAction(response))
}
```

当 `watchRequestActions` 执行 `yield call(handleRequestAction, url, params)` 时，它会在继续下一个 `yield take` 之前一直等待 `handleRequestAction` 直到其结束并返回。
例如，假设我们有这样一个事件队列：

```
UI                     watchRequestActions             handleRequestAction  
-----------------------------------------------------------------------------
.......................take('REQUEST').......................................
dispatch(REQUEST)......call(handleRequestAction).......call(someRemoteApi)... 等待服务器返回
.............................................................................   
.............................................................................
dispatch(REQUEST)............................................................ 错过 action！！
.............................................................................   
.............................................................................
.......................................................put(someAction).......
.......................take('REQUEST')....................................... saga 恢复了
```

如上所示，当 Saga 被一个 **阻塞调用（blocking call）** 阻塞了，在阻塞解除（译注：即例子中的 put(someAction)）之前，它将错过所有发起的 action。

为了避免阻塞 Saga，你可以使用 **无阻塞调用** Effect `fork` 来代替 `call`。

```javascript
function watchRequestActions() {
  while(true) {
    const {url, params} = yield take('REQUEST')
    yield fork(handleRequestAction, url, params) //  Saga 将立即恢复
  }
}
```
