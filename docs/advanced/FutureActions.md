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

The `take` is just like `call` and `put` we saw earlier. It creates another command object
that tells the middleware to wait for a specific action. Just as the middleware suspends
the Generator in case of `call` Effects until the returned Promise resolve. In the `take`
case it'll suspend the Generator until a matching action is dispatched. In the above example
`watchAndLog` is suspended until any action is dispatched.

Note how we're running an endless loop `while(true)`. Remember this is a Generator function,
which doesn't have a run-to-completion behavior. Our Generator will block on each iteration
waiting for an action to happen.

Using `take` has a subtle impact on how we write our code. In the case of `takeEvery` the invoked
tasks has no control on when to be called, they will be invoked again and again on each matching
action. They also have no control on when to stop the observation.

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
