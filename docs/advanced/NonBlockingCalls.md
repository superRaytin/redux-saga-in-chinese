# 无阻塞调用

在之前的章节中，我们看到了 `take` Effect 让我们可以在一个集中的地方更好地去描述一个非常规的流程。

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

让我们来完成这个例子，并实现真实的登录/登出逻辑。假设有这样一个 API，它允许我们在一个远程服务器上验证用户的权限。
如果验证成功，服务器将会返回一个授权令牌，我们的应用程序将会通过 DOM storage 存储这个令牌（假设我们的 API 为 DOM storage 提供了另外一个服务）。

当用户登出，我们将直接删除以前存储的授权令牌。

### 初次尝试

到目前为止，我们拥有所有需要的 Effects 用来实现上述流程。我们可以使用 `take` Effect 等待 store 中指定的 action。
我们也可以使用 `call` Effect 进行同步调用，最后使用 `put` Effect 来发起 action 到 store。

让我们试试吧：

> 注意，以下代码有一个小问题，请务必将这一节全部阅读完。

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
    yield call(Api.clearItem, 'token')
    }
  }
}
```

首先我们创建了一个独立的 Generator `authorize`，它将执行真实的 API 调用并在成功后通知 Store。

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

请观察整个逻辑是如何存储在一个地方的。一个新的开发者阅读我们的代码时，不必再为了理解控制流而在各个地方来回切换。
这就像是在阅读同步代码：它们的自然顺序确定了执行步骤。并且我们有很多 Effects 可以调用其他函数并等待它们的结果。

### 但上面的方法还是有一个小问题

假设 `loginFlow` 正在等待如下的调用被 resolve：

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
但在我们的情况中，我们不仅希望 `loginFlow` 执行授权调用，也想监听可能发生在调用未完成之前的 `LOGOUT` action。
因为 `LOGOUT` 与调用 `authorize` 是 *并发的*。

所以我们需要的是一些非阻塞调用 `authorize` 的方法。这样 `loginFlow` 就可以继续执行，并且监听并发的或响应未完成之前发出的 `LOGOUT` action。

为了表示无阻塞调用，redux-saga 提供了另一个 Effect：[`fork`](http://leonshi.com/redux-saga-in-chinese/docs/api/index.html#forkfn-args)。
当我们 fork 一个 *任务*，任务会在后台启动，调用者也可以继续它自己的流程，而不用等待被 fork 的任务结束。

所以为了让 `loginFlow` 不错过一个并发的 `LOGOUT`，我们不应该使用 `call` 调用 `authorize` 任务，而应该使用 `fork`。

```javascript
import { fork, call, take, put } from 'redux-saga/effects'

function* loginFlow() {
  while(true) {
    ...
    try {
      // 无阻塞调用，这里返回的值是什么？
      const ?? = yield fork(authorize, user, password)
      ...
    }
    ...
  }
}
```

现在的问题是，自从 `authorize` 的 action 在后台启动之后，我们获取不到 `token` 的结果（因为我们不应该等待它）。
所以我们需要将 token 存储操作移到 `authorize` 任务内部。

```javascript
import { fork, call, take, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    yield call(Api.storeItem, {token})
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  }
}

function* loginFlow() {
  while(true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    yield fork(authorize, user, password)
    yield take(['LOGOUT', 'LOGIN_ERROR'])
    yield call(Api.clearItem, 'token')
  }
}
```

我们使用了 `yield take(['LOGOUT', 'LOGIN_ERROR'])`。意思是监听 2 个并发的 action：

- 如果 `authorize` 任务在用户登出之前成功了，它将会发起一个 `LOGIN_SUCCESS` action 然后结束。
然后 `loginFlow` Saga 只会等待一个未来的 `LOGOUT` action 被发起（因为 `LOGIN_ERROR` 永远不会发生）。

- 如果 `authorize` 在用户登出之前失败了，它将会发起一个 `LOGIN_ERROR` action 然后结束。
那么 `loginFlow` 将在 `LOGOUT` 之前收到 `LOGIN_ERROR`，然后它会进入另外一个 `while` 迭代并等待下一个 `LOGIN_REQUEST` action。

- 如果在 `authorize` 结束之前，用户就登出了，那么 `loginFlow` 将收到一个 `LOGOUT` action 并且也会等待下一个 `LOGIN_REQUEST`。

注意 `Api.clearItem` 应该是幂等调用。如果 `authorize` 调用时没有存储 token 也不会有任何影响。
`loginFlow` 仅仅是保证在等待下一次登录之前，storage 中没有 token。

但是还没完。如果我们在 API 调用期间收到一个 `LOGOUT` action，我们必须要 **取消** `authorize` 处理进程，否则将有 2 个并发的任务，
并且 `authorize` 任务将会继续运行，并在成功的响应（或失败的响应）返回后发起一个 `LOGIN_SUCCESS` action（或一个 `LOGIN_ERROR` action），而这将导致状态不一致。

为了取消 fork 任务，我们可以使用一个指定的 Effect [`cancel`](http://superRaytin.github.io/redux-saga-in-chinese/docs/api/index.html#canceltask)。

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
    yield call(Api.clearItem, 'token')
  }
}
```

`yield fork` 的返回结果是一个 [Task Object](http://superRaytin.github.io/redux-saga-in-chinese/docs/api/index.html#task)。
我们将它们返回的对象赋给一个本地常量 `task`。如果我们收到一个 `LOGOUT` action，我们将那个 task 传入给 `cancel` Effect。
如果任务仍在运行，它会被中止。如果任务已完成，那什么也不会发生，取消操作将会是一个空操作（no-op）。最后，如果该任务完成了但是有错误，
那我们什么也没做，因为我们知道，任务已经完成了。

OK，我们 *几乎* 要完成了（是的，它的并发性并不容易，你必须认真对待）。

假设在我们收到一个 `LOGIN_REQUEST` action 时，我们在 reducer 中设置了一些 `isLoginPending` 标识为 true，以便可以在界面上显示一些消息或者旋转 loading。
如果此时我们在 Api 调用期间收到一个 `LOGOUT` action，并通过 *杀死它*（即任务被立即停止）简单粗暴地中止任务。
那我们可能又以不一致的状态结束了。因为 `isLoginPending` 仍然是 true，而 reducer 还在等待一个结果 action（`LOGIN_SUCCESS` 或 `LOGIN_ERROR`）。

幸运的是，`cancel` Effect 不会粗暴地结束我们的 `authorize` 任务，相反它会给予一个机会执行清理的逻辑。
在 `finally` 区块可以处理任何的取消逻辑（以及其他类型的完成逻辑）。由于 finally 区块执行在任何类型的完成上（正常的 return, 错误, 或强制取消），如果你想要为取消作特殊处理，有一个 `cancelled` Effect：

```javascript
import { take, call, put, cancelled } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    yield call(Api.storeItem, {token})
    return token
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  } finally {
    if (yield cancelled()) {
      // ... put special cancellation handling code here
    }
  }
}
```

你可能已经注意到了，我们仍然没有做任何与清除 `isLoginPending` 状态相关的事情。
对于这一点，有 2 个可能的解决方案（或其他的）。

- 发起一个指定的 action `RESET_LOGIN_PENDING`。

- 或者更简单，让 reducer 收到 `LOGOUT` action 时清除 `isLoginPending`。
