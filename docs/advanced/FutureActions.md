# 监听未来的 action

到现在为止，我们已经使用了辅助函数 `takeEvery` 在每个 action 来到时派生一个新的任务。
这多少有些模仿 redux-thunk 的行为：举个例子，每次一个组件调用 `fetchProducts` Action 创建器（Action Creator），Action 创建器就会发起一个 thunk 来执行控制流。

在现实情况中，`takeEvery` 只是一个在强大的低阶 API 之上构建的辅助函数。
在这一节中我们将看到一个新的 Effect，即 `take`。`take` 使得通过允许全面控制 action 观察进程来构建复杂的控制流成为可能。

## 一个简单的日志记录器

让我们开始一个简单的 Saga 例子，这个 Saga 将监听所有发起到 store 的 action，然后将它们记录到控制台。

使用 `takeEvery('*')`（`*` 通配符模式），我们就能捕获发起的所有类型的 action。

```javascript
import { takeEvery } from 'redux-saga'

function* watchAndLog(getState) {
  yield* takeEvery('*', function* logger(action) {
    console.log('action', action)
    console.log('state after', getState())
  })
}
```

现在我们知道如何使用 `take` Effect 来实现与上面相同的功能：

```javascript
import { take } from 'redux-saga/effects'

function* watchAndLog(getState) {
  while(true) {
    const action = yield take('*')
    console.log('action', action)
    console.log('state after', getState())
  })
}
```

`take` 就像我们更早之前看到的 `call` 和 `put`。它创建另一个命令对象，告诉 middleware 等待一个特定的 action。
正如在 `call` Effect 的情况中，middleware 暂停 Generator，直到返回的 Promise 被 resolve。
在 `take` 的情况中，它将会暂停 Generator 直到一个匹配的 action 被发起了。
在以上的例子中，`watchAndLog` 被暂停了，直到任意的一个 action 被发起。

注意，我们运行了一个无限循环的 `while(true)`。记住这是一个 Generator 函数，它不具备 `从运行至完成` 的行为（run-to-completion behavior）。
我们的 Generator 将在每次迭代上阻塞以等待 action 发起。

使用 `take` 对于我们如何写代码有一个微妙的影响。在 `takeEvery` 的情况中，被调用的任务无法控制何时被调用，
它们将在每次 action 被匹配时一遍又一遍地被调用。并且它们也无法控制何时停止监听。

In the case of `take` the control is inversed. Instead of the actions being *pushed* to the
handler tasks, the Saga is *pulling* the action by itself. It looks as if the Saga is performing
a normal function call `action = getNextAction()` which will resolve when the action is
dispatched.

This inversion of control allows us to implement control flows that are non trivial to do with
the traditional *push* approach.

As a simple example, suppose that in our Todo application, we want to watch for the user actions
and show a congratulation message when the User has created his three first Todos.

```javascript
import { take, put } from 'redux-saga/effects'

function* watchFirstThreeTodosCreation() {
  for(let i = 0; i < 3; i++) {
    const action = yield take('TODO_CREATED')
  }
  yield put({type: 'SHOW_CONGRATULATION'})
}
```

Instead of a `while(true)` we're running a `for` loop which will iterate only for 3 times. After
taking the first three `TODO_CREATED` actions, `watchFirstThreeTodosCreation` will cause the
application to display a congratulation message then terminates. It means the Generator will be
garbage collected and no longer observation will take place.

Another benefit of the pull approach is that we can describe our control flow using a familiar
synchronous style. For example suppose we want to implement a login flow with 2 actions `LOGIN`
and `LOGOUT`. Using `takeEvery` (or redux-thunk) we'll have to write 2 separate tasks (or thunks) : one for
`LOGIN` and the other for `LOGOUT`.

The result is that our logic is now spread in 2 places. In order for someone reading our code to
understand what's going on, he has to read the source of the 2 handlers and make the link
between the logic in both. It means he has to rebuild the model of the flow in his head
by rearranging mentally the logic placed in various places of the code in the correct
order.

Using the pull model we can write our flow in the same place

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

Instead of handling the same action repeatedly. The `loginFlow` Saga has a better understanding
of the expected sequence of actions. It knows that `LOGIN` action should always be followed by
a `LOGOUT` action. And that `LOGOUT` is always followed by a `LOGIN` (a good UI should always enforce
a consistent order of the actions, by hiding or disabling unexpected action).
