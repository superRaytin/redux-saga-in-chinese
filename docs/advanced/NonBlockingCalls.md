# 无阻塞调用

在上一节中，我们看到了 `take` Effect 让我们可以在一个集中的地方更好地去描述一个非常规的流程。

重温一下登录流程示例：


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

让我们来完成这个例子，并实现真实的登录/登出逻辑。假设有这样一个 Api，它允许我们在一个远程服务器上验证用户的权限。
如果验证成功，服务器将会返回一个授权令牌，我们的应用程序将会通过 DOM storage 存储这个令牌（假设我们的 Api 为 DOM storage 提供了另外一个服务）。

当用户登出，我们将直接删除以前存储的授权令牌。

### 初次尝试

到目前为止，我们拥有所有需要的 Effects 用来实现上述流程。我们可以使用 `take` Effect 等待 store 中指定的 action。
我们也可以使用 `call` Effect 进行同步调用，最后使用 `put` Effect 来发起 action 到 store。

让我们试试吧：

> 注意，以下代码有一个微妙的问题，请务必将这一节全部阅读完。

```javascript
import { take, call, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    return token
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  }
}

function* loginFlow() {
  while(true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    const token = yield call(authorize, user, password)
    if(token) {
      yield call(Api.storeItem({token}))
      yield take('LOGOUT')
      yield call(Api.clearItem('token'))
    }
  }
}
```

首先我们创建了一个独立的 Generator `authorize`，它将执行真实的 Api 调用并在成功后通知 Store。

`loginFlow` 在一个 `while(true)` 循环中实现它所有的流程，这样做的意思是：一旦到达流程最后一步（`LOGOUT`），通过等待一个新的 `LOGIN_REQUEST` action 来启动一个新的迭代。

`loginFlow` 首先等待一个 `LOGIN_REQUEST` action。
然后在 action 的 payload 中获取有效凭据（即 `user` 和 `password`）并调用一个 `call` 到 `authorize` 任务。

正如你注意到的，`call` 不仅可以用来调用返回 Promise 的函数。我们也可以用它来调用其他 Generator 函数。
在上面的例子中，**`loginFlow` 将等待 `authorize` 直到它终止或返回**（即执行 api 调用后，发起 action 然后返回 token 至 `loginFlow`）。

如果 Api 调用成功了，`authorize` 将发起一个 `LOGIN_SUCCESS` action 然后返回获取到的 token。
如果调用导致了错误，将会发起一个 `LOGIN_ERROR` action。

如果调用 `authorize` 成功，`loginFlow` 将在 DOM storage 中存储返回的 token，并等待 `LOGOUT` action。
当用户登出，我们删除存储的 token 并等待一个新的用户登录。

在 `authorize` 失败的情况下，它将返回一个 undefined 值，这将导致 `loginFlow` 跳过当前处理进程并等待一个新的 `LOGIN_REQUEST` action。

观察整个逻辑是如何存储在一个地方的。一个新的开发者阅读我们的代码时，不必再为了理解控制流而在各个地方来回切换。
这就像是在阅读同步代码：它们的自然顺序确定了执行步骤。并且我们有很多 Effects 可以调用其他函数并等待它们的结果。

### 但上面的方法还是有一个微妙的问题

假如 `loginFlow` 正在等待如下的调用被 resolve：

```javascript
function* loginFlow() {
  while(true) {
    ...
    try {
      const token = yield call(authorize, user, password)
      ...
    }
    ...
  }
}
```

但用户点击了 `Logout` 按钮使得 `LOGOUT` action 被发起。

下面的例子演示了假想的一系列事件：

```
UI                              loginFlow
--------------------------------------------------------
LOGIN_REQUEST...................call authorize.......... waiting to resolve
........................................................
........................................................                     
LOGOUT.................................................. missed!
........................................................
................................authorize returned...... dispatch a `LOGIN_SUCCESS`!!
........................................................
```

当 `loginFlow` 在 `authorize` 中被阻塞了，最终发生在开始调用和收到响应之间的 `LOGOUT` 将会被错过，
因为那时 `loginFlow` 还没有执行 `yield take('LOGOUT')`。

上面代码的问题是 `call` 是一个会阻塞的 Effect。即 Generator 在调用结束之前不能执行或处理任何其他事情。
但在我们的情况中，我们不仅想要 `loginFlow` 执行授权调用，也想监听可能发生在调用未完成之前的 `LOGOUT` action。
因为 `LOGOUT` 与调用 `authorize` 是 *并发的*。

所以我们需要的是一些非阻塞调用 `authorize` 的方法。这样 `loginFlow` 就可以继续执行，并且监听并发的或响应未完成之前发出的 `LOGOUT` action。

为了表示无阻塞调用，redux-saga 提供了另一个 Effect：[`fork`](http://superRaytin.github.io/redux-saga/docs/api/index.html#forkfn-args)。
当我们 fork 一个 *任务*，任务会在后台启动，调用者也可以继续它自己的流程，而不用等待被 fork 的任务结束。

所以为了让 `loginFlow` 不错过一个并发的 `LOGOUT`，我们不要使用 `call` 调用 `authorize` 任务，而应该使用 `fork`。

```javascript
import { fork, call, take, put } from 'redux-saga/effects'

function* loginFlow() {
  while(true) {
    ...
    try {
      // non blocking call, what's the returned value here ?
      const ?? = yield fork(authorize, user, password)
      ...
    }
    ...
  }
}
```

The issue now is since our `authorize` action is started in the background, we can't get
the `token` result (because we'd have to wait for it). So we need to move the token storage
operation into the `authorize` task.

```javascript
import { fork, call, take, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  }
}

function* loginFlow() {
  while(true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    yield fork(authorize, user, password)
    yield take(['LOGOUT', 'LOGIN_ERROR'])
    yield call(Api.clearItem('token'))
  }
}
```

We're also doing `yield take(['LOGOUT', 'LOGIN_ERROR'])`. It means we are watching for
2 concurrent actions :

- If the `authorize` task succeeds before the user logouts, it'll dispatch a `LOGIN_SUCCESS`
action then terminates. Our `loginFlow` saga will then wait only for a future `LOGOUT` action
(because `LOGIN_ERROR` will never happen).

- If the `authorize` fails before the user logouts, it'll dispatch a `LOGIN_ERROR` action then
terminates. So `loginFlow` will take the `LOGIN_ERROR` before the `LOGOUT` then it'll enter in
a another `while` iteration and will wait for the next `LOGIN_REQUEST` action.

- If the user logouts before the `authorize` terminate, then `loginFlow` will take a `LOGOUT`
action and also wait for the next `LOGIN_REQUEST`.

Note the call for `Api.clearItem` is supposed to be idempotent. It'll have no effect if no token was
stored by the `authorize` call. `loginFlow` just make sure no token will be in the storage
before waiting for the next login.

But we've not yet done. If we take a `LOGOUT` in the middle of an Api call, we have
to **cancel** the `authorize` process, otherwise we'll have 2 concurrent tasks evolving in
parallel, otherwise, the `authorize` task will continue running and upon a successful (resp. failed)
result, will dispatch a `LOGIN_SUCCESS` (resp. a `LOGIN_ERROR`) action leading to an inconsistent state.


In order to cancel a forked task, we use a dedicated Effect [`cancel`](http://yelouafi.github.io/redux-saga/docs/api/index.html#canceltask)

```javascript
import { take, put, call, fork, cancel } from 'redux-saga/effects'

// ...

function* loginFlow() {
  while(true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    // fork return a Task object
    const task = yield fork(authorize, user, password)
    const action = yield take(['LOGOUT', 'LOGIN_ERROR'])
    if(action.type === 'LOGOUT')
      yield cancel(task)
    yield call(Api.clearItem('token'))
  }
}
```

`yield fork` results in a [Task Object](http://yelouafi.github.io/redux-saga/docs/api/index.html#task). We assign
the returned object into a local constant `task`. Later if we take a `LOGOUT` action, we pass that task
to the `cancel` Effect. If the task is still running, it'll be aborted. If the task has already completed
then nothing will happen and the cancellation will result in a no-op. And finally, if the task
completed with an error, the we do nothing, because we know that the task has already completed.

Ok, we are *almost* done (yeah, concurrency it's not that easy, you have to take it seriously).

Suppose that when we receive a `LOGIN_REQUEST` action, our reducer sets some `isLoginPending` flag
to true so it can display some message or spinner in the UI. If we get a `LOGOUT` in the middle
of an Api call and abort the task by simply *killing it* (i.e. the task is stopped right away), then
we may endup again with an inconsistent state. Because we'll still have `isLoginPending` set to true
and our reducer waiting for an outcome action (`LOGIN_SUCCESS` or `LOGIN_ERROR`).

Fortunately, the `cancel` Effect won't brutally kill our `authorize` task, it'll instead throw
a special Error inside it to give it a chance to perform its cleanup logic. And the cancelled task
should catch this Error if it has something to do before terminating.

Our `authorize` already have a try/catch block, but it defines a generic handler which dispatches
`LOGIN_ERROR` actions on each error. But login cancellations aren't really errors.
It doesn't make sense to display an error message to the user we he logouts. So our `authorize`
task has to dispatch `LOGIN_ERROR` actions only on case of authorization failures.


```javascript
import { isCancelError } from 'redux-saga'
import { take, call, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    return token
  } catch(error) {
    if(!isCancelError(error))
      yield put({type: 'LOGIN_ERROR', error})
  }
}
```

You may have noted that we still didn't do anyting about clearing our `isLoginPending` state.
For that, there are 2 possible solutions (and maybe others)

- dispatch a dedicated action `RESET_LOGIN_PENDING`

- or more simply, make the reducer clear the `isLoginPending` on a `LOGOUT` action.

## **More to come**
