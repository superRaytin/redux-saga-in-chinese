# 监听未来的 action

到现在为止，我们已经使用了辅助函数 `takeEvery` 在每个 action 来到时派生一个新的任务。
这多少有些模仿 redux-thunk 的行为：举个例子，每次一个组件调用 `fetchProducts` Action 创建器（Action Creator），Action 创建器就会发起一个 thunk 来执行控制流。

在现实情况中，`takeEvery` 只是一个在强大的低阶 API 之上构建的辅助函数。
在这一节中我们将看到一个新的 Effect，即 `take`。`take` 让我们通过全面控制 action 观察进程来构建复杂的控制流成为可能。

## 一个简单的日志记录器

让我们开始一个简单的 Saga 例子，这个 Saga 将监听所有发起到 store 的 action，然后将它们记录到控制台。

使用 `takeEvery('*')`（`*` 代表通配符模式），我们就能捕获发起的所有类型的 action。

```javascript
import { select, takeEvery } from 'redux-saga'

function* watchAndLog(getState) {
  yield* takeEvery('*', function* logger(action) {
    const state =  yield select()
    
    console.log('action', action)
    console.log('state after', state)
  })
}
```

现在我们知道如何使用 `take` Effect 来实现和上面相同的功能：

```javascript
import { select, take } from 'redux-saga/effects'

function* watchAndLog(getState) {
  while(true) {
    const action = yield take('*')
    const state = yield select()
    
    console.log('action', action)
    console.log('state after', state)
  }
}
```

`take` 就像我们更早之前看到的 `call` 和 `put`。它创建另一个命令对象，告诉 middleware 等待一个特定的 action。
正如在 `call` Effect 的情况中，middleware 会暂停 Generator，直到返回的 Promise 被 resolve。
在 `take` 的情况中，它将会暂停 Generator 直到一个匹配的 action 被发起了。
在以上的例子中，`watchAndLog` 处于暂停状态，直到任意的一个 action 被发起。

注意，我们运行了一个无限循环的 `while(true)`。记住这是一个 Generator 函数，它不具备 `从运行至完成` 的行为（run-to-completion behavior）。
Generator 将在每次迭代上阻塞以等待 action 发起。

使用 `take` 组织代码有一个小问题。在 `takeEvery` 的情况中，被调用的任务无法控制何时被调用，
它们将在每次 action 被匹配时一遍又一遍地被调用。并且它们也无法控制何时停止监听。

而在 `take` 的情况中，控制恰恰相反。与 action 被 *推向（pushed）* 任务处理函数不同，Saga 是自己主动 *拉取（pulling）* action 的。
看起来就像是 Saga 在执行一个普通的函数调用 `action = getNextAction()`，这个函数将在 action 被发起时 resolve。

这样的反向控制让我们能够实现一些使用传统的 *push* 方法做非常规事情的控制流。

一个简单的例子，假设在我们的 Todo 应用中，我们希望监听用户的操作，并在用户初次创建完三条 Todo 信息时显示祝贺信息。

```javascript
import { take, put } from 'redux-saga/effects'

function* watchFirstThreeTodosCreation() {
  for(let i = 0; i < 3; i++) {
    const action = yield take('TODO_CREATED')
  }
  yield put({type: 'SHOW_CONGRATULATION'})
}
```

与 `while(true)` 不同，我们运行一个只迭代三次的 `for` 循环。在 `take` 初次的 3 个 `TODO_CREATED` action 之后，
`watchFirstThreeTodosCreation` Saga 将会使应用显示一条祝贺信息然后中止。这意味着 Generator 会被回收并且相应的监听不会再发生。

主动拉取 action 的另一个好处是我们可以使用熟悉的同步风格来描述我们的控制流。举个例子，假设我们希望实现一个这样的登录控制流，有两个 action 分别是 `LOGIN` 和 `LOGOUT`。
使用 `takeEvery`（或 redux-thunk）我们必须要写两个分别的任务（或 thunks）：一个用于 `LOGIN`，另一个用于 `LOGOUT`。

结果就是我们的逻辑现在分开在两个地方了。别人为了阅读我们的代码搞明白这是怎么回事，他必须阅读两个处理函数的源代码并且要在两处逻辑之间建立连接。
这意味着他必须通过在心中重新排列放在几个不同地方的代码逻辑获得正确的排序，从而在脑中重建控制流模型。

使用拉取（pull）模式，我们可以在同一个地方写控制流。而不是重复处理同样的acton

```javascript
function* loginFlow() {
  while(true) {
    yield take('LOGIN')
    // ... perform the login logic
    yield take('LOGOUT')
    // ... perform the logout logic
  }
}
```

与重复处理相同的 action 不同，`loginFlow` Saga 更好理解，因为序列中的 actions 是期望之中的。它知道 `LOGIN` action
后面应该始终跟着一个 `LOGOUT` action。`LOGOUT` 也应该始终跟在一个 `LOGIN` 后面（一个好的 UI 程序应该始终强制执行顺序一致的 actions，通过隐藏或禁用意料之外的 action）。
