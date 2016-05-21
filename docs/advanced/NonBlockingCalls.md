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

> 注意，以下代码有一个微妙的问题，请务必将这一节全部阅读完

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

If the Api call succeeds, `authorize` will dispatch a `LOGIN_SUCCESS` action then returns the
fetched token. If it results in an error,  it'll dispatch a `LOGIN_ERROR` action.

If the call to `authorize` is successful, `loginFlow` will store the returned token
in the DOM storage and wait for a `LOGOUT` action. When the user logouts, we remove the
stored token and wait for a new user login.

In the case of `authorize` failed, it'll return an undefined value, which will cause `loginFlow`
to skip the pervious process and wait for a new `LOGIN_REQUEST` action.

Observe how the entire logic is stored in one place. A new developer reading our code
doesn't have to travel between various places in order to understand the
control flow. It's like reading a synchronous algorithm: steps are layed out in their
natural order. And we have functions which call other functions and wait for their results.

### But there is still a subtle issue with the above approach

Suppose that when the `loginFlow` is waiting for the following call to resolve

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

The user clicks on the `Logout` button causing a `LOGOUT` action to be dispatched

The following example illustrates the hypothetical sequence of the events :

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

When `loginFlow` is blocked on the `authorize` call, an eventual `LOGOUT` occurring in
between the call and the response will be missed, because `loginFlow` hasn't yet performed
the `yield take('LOGOUT')`.

The problem with the above code is that `call` is a blocking Effect. i.e. the Generator
can't perform/handle anything else until the call terminates. But in our case we do not only
want `loginFlow` to execute the authorization call, but also watch for an eventual `LOGOUT`
action that may occur in the middle of this call. That's because `LOGOUT` is *concurrent*
to the `authorize` call.

So what's needed is some way to start `authorize` without blocking. So `loginFlow` can continue
and watch for an eventual/concurrent `LOGOUT` action.


To express non-blocking calls, the library provides another Effect: [`fork`](http://yelouafi.github.io/redux-saga/docs/api/index.html#forkfn-args). When we fork
a *task*, the task is started in the background and the caller can continue its flow without
waiting for the forked task to terminate.

So in order for `loginFlow` to not miss a concurrent `LOGOUT`, we must not `call` the `authorize`
task, instead we have to `fork` it.

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
