# API 参考

* [`Middleware API`](#middleware-api)
  * [`createSagaMiddleware(...sagas)`](#createsagamiddlewaresagas)
  * [`middleware.run(saga, ...args)`](#middlewarerunsaga-args)
* [`Saga Helpers`](#saga-helpers)
  * [`takeEvery(pattern, saga, ...args)`](#takeeverypattern-saga-args)
  * [`takeLatest(pattern, saga, ..args)`](#takelatestpattern-saga-args)
* [`Effect creators`](#effect-creators)
  * [`take(pattern)`](#takepattern)
  * [`put(action)`](#putaction)
  * [`call(fn, ...args)`](#callfn-args)
  * [`call([context, fn], ...args)`](#callcontext-fn-args)
  * [`apply(context, fn, args)`](#applycontext-fn-args)
  * [`cps(fn, ...args)`](#cpsfn-args)
  * [`cps([context, fn], ...args)`](#cpscontext-fn-args)
  * [`fork(fn, ...args)`](#forkfn-args)
  * [`fork([context, fn], ...args)`](#forkcontext-fn-args)
  * [`join(task)`](#jointask)
  * [`cancel(task)`](#canceltask)
  * [`select(selector, ...args)`](#selectselector-args)
* [`Effect combinators`](#effect-combinators)
  * [`race(effects)`](#raceeffects)
  * [`[...effects] (aka parallel effects)`](#effects-parallel-effects)
* [`Interfaces`](#interfaces)
  * [`Task`](#task)
* [`External API`](#external-api)
  * [`runSaga(iterator, {subscribe, dispatch, getState}, [monitor])`](#runsagaiterator-subscribe-dispatch-getstate-monitor)


## Middleware API
------------------------

### `createSagaMiddleware(...sagas)`

创建一个 Redux 中间件，将 Sagas 与 Redux Store 建立连接。

- `sagas: Array<Function>` - Generator 函数列表

#### 例子

```JavaScript
import createSagaMiddleware from 'redux-saga'
import reducer from './path/to/reducer'
import sagas from './path/to/sagas'

export default function configureStore(initialState) {
  // 注意：redux@>=3.1.0 的版本才支持把 middleware 作为 createStore 方法的最后一个参数
  return createStore(
    reducer,
    initialState,
    applyMiddleware(/* other middleware, */createSagaMiddleware(...sagas))
  )
}

```

#### 注意事项

`sagas` 中的每个 Generator 函数被调用时，都会被传入 Redux Store 的 `getState` 方法作为第一个参数。

`sagas` 中的每个函数都必须返回一个 [Generator 对象](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator)。
middleware 会迭代这个 Generator 并执行所有 yield 后的 Effect（译注：Effect 可以看作是 redux-saga 的任务单元，参考 [名词解释](http://leonshi.com/redux-saga-in-chinese/docs/Glossary.html)）。

在第一次迭代里，middleware 会调用 `next()` 方法以取得下一个 Effect。然后 middleware 会通过下面提到的 Effects API 来执行 yield 后的 Effect。
与此同时，Generator 会暂停，直到 Effect 执行结束。当接收到执行的结果，middleware 在 Generator 里接着调用 `next(result)`，并将得到的结果作为参数传入。
这个过程会一直重复，直到 Generator 正常或通过抛出一些错误结束。

如果执行引发了一个错误（像提到的那些 Effect 创建器），就会调用 Generator 的 `throw(error)` 方法来代替。如果定义了一个 `try/catch` 包裹当前的 yield 指令，
那么 `catch` 区块将被底层 Generator runtime 调用。

### `middleware.run(saga, ...args)`

动态执行 `saga`。用于 `applyMiddleware` 阶段之后执行 Sagas。

- `saga: Function`: 一个 Generator 函数
- `args: Array<any>`: 提供给 `saga` 的参数 (除了 Store 的 `getState` 方法)

这个方法返回一个 [Task 描述对象](#task-descriptor)

#### 注意事项

在某些场景中，比如在大型应用中使用 code splitting（模块按需从服务器上加载），又或者处于服务端环境中，
在这些情况下，我们可能希望或者需要在 `applyMiddleware` 阶段完成之后启动 Sagas。

当你创建一个 middleware 实例：

```javascript
import createSagaMiddleware from 'redux-saga'
import startupSagas from './path/to/sagas'

// middleware 实例
const sagaMiddleware = createSagaMiddleware(...startupSagas)
```

middleware 实例暴露了一个 `run` 方法。你可以用这个方法来执行 Sagas，并在稍后的某个时间点连接 Store。

Saga 的第一个参数为 Redux Store 的 `getState` 方法。如果 `run` 方法被提供了非空的 `...args`，那么 `args` 的所有元素将作为 `saga` 的额外参数。

#### 例子

这个例子中，从 `configureStore.js` 输出 Saga middleware。然后在 `someModule` 中引入它，`someModule` 从服务端动态加载一个 Saga，
然后使用引入的 middleware 的 `run` 方法来执行载入的 Saga。

##### `configureStore.js`

```javascript
import createSagaMiddleware from 'redux-saga'
import reducer from './path/to/reducer'
import startupSagas from './path/to/sagas'

export const sagaMiddleware = createSagaMiddleware(...startupSagas)

export default function configureStore(initialState) {
  // 注意：redux@>=3.1.0 的版本才支持把 middleware 作为 createStore 方法的最后一个参数
  return createStore(
    reducer,
    initialState,
    applyMiddleware(/* other middleware, */, sagaMiddleware)
  )
}
```

##### `someModule.js`

```javascript
import { sagaMiddleware } from './configureStore'

require.ensure(["dynamicSaga"], (require) => {
    const dynamicSaga = require("dynamicSaga");
    const task = sagaMiddleware.run(dynamicSaga)
});
```

## Saga 辅助函数
--------------------------

>#### 注意，以下的函数都是辅助函数，是在 Effect 创建器的基础之上构建的（译注：即高阶 API）

### `takeEvery(pattern, saga, ...args)`

在发起的 action 与 `pattern` 匹配时派生指定的 `saga`。

每次发起一个 action 到 Store，并且这个 action 与 `pattern` 相匹配，那么 `takeEvery` 将会在后台启动一个新的 `saga` 任务。

- `pattern: String | Array | Function` - 要查看更多信息可查看文档 [`take(pattern)`](#takepattern)

- `saga: Function` - 一个 Generator 函数

- `args: Array<any>` - 将被传入启动的任务作为参数。`takeEvery` 会把当前的 action 放入参数列表（action 将作为 `saga` 的最后一个参数）

#### 例子

在以下的例子中，我们创建了一个简单的任务 `fetchUser`。在每次 `USER_REQUESTED` action 被发起时，使用 `takeEvery` 来启动一个新的 `fetchUser` 任务。

```javascript
import { takeEvery } from `redux-saga`

function* fetchUser(action) {
  ...
}

function* watchFetchUser() {
  yield* takeEvery('USER_REQUESTED', fetchUser)
}
```

#### 注意事项

`takeEvery` 是一个高阶 API，使用 `take` 和 `fork` 构建。下面演示了这个辅助函数是如何实现的：

```javascript
function* takeEvery(pattern, saga, ...args) {
  while(true) {
    const action = yield take(pattern)
    yield fork(saga, ...args.concat(action))
  }
}
```

`takeEvery` 允许处理并发的 action（译注：即同时触发相同的 action）。在上面的例子里，当发起一个 `USER_REQUESTED` action 时，
将会启动一个新的 `fetchUser` 任务，即使前一个 `fetchUser` 任务还未处理结束
（举个例子，用户以极快的速度连续点击一个 `Load User` 按钮 2 次，第二次点击依然会发起一个 `USER_REQUESTED` action，即使第一个触发的 `fetchUser` 任务还未结束）

`takeEvery` 不会对多个任务的响应返回进行排序，并且也无法保证任务将会按照启动的顺序结束。如果要对响应进行排序，可以关注以下的 `takeLatest`。

### `takeLatest(pattern, saga, ...args)`

在发起的 action 与 `pattern` 匹配时派生指定的 `saga`。并且自动取消之前启动的所有 `saga` 任务（如果在执行中）。

每次发起一个 action 到 Store，并且这个 action 与 `pattern` 相匹配，那么 `takeLatest` 将会在后台启动一个新的 `saga` 任务。
如果之前已经有一个 `saga` 任务启动了（当前 action 之前的最后发起的 action），并且这个任务在执行中，那这个任务将被取消，
并抛出一个 `SagaCancellationException` 错误。

- `pattern: String | Array | Function` - 要查看更多信息可查看文档 [`take(pattern)`](#takepattern)

- `saga: Function` - 一个 Generator 函数

- `args: Array<any>` - 将被传入启动的任务作为参数。`takeLatest` 会把当前的 action 放入参数列表（action 将作为 `saga` 的最后一个参数）

#### 例子

在以下的例子中，我们创建了一个简单的任务 `fetchUser`。在每次 `USER_REQUESTED` action 被发起时，使用 `takeLatest` 来启动一个新的 `fetchUser` 任务。
由于 `takeLatest` 取消了所有之前启动的未完成的任务，这样就可以保证：如果用户以极快的速度连续多次触发 `USER_REQUESTED` action，将会以最后的那个结束。

```javascript
import { takeLatest } from `redux-saga`

function* fetchUser(action) {
  ...
}

function* watchLastFetchUser() {
  yield* takeLatest('USER_REQUESTED', fetchUser)
}
```

#### 注意事项

`takeLatest` 是一个高阶 API，使用 `take` 和 `fork` 构建。下面演示了这个辅助函数是如何实现的：

```javascript
function* takeLatest(pattern, saga, ...args) {
  let lastTask
  while(true) {
    const action = yield take(pattern)
    if(lastTask)
      yield cancel(lastTask) // 如果任务已经终止，取消就是空操作

    lastTask = yield fork(saga, ...args.concat(action))
  }
}
```

## Effect 创建器
-------------------------

>#### 注意
以下每个函数都会返回一个 plain Javascript object (纯文本 Javascript 对象) 并且不会执行任何其它的操作。
执行是由 middleware 在上述迭代过程中进行的。
middleware 检查每个 Effect 的信息，并进行相应的操作。

### `take(pattern)`

创建一条 Effect 描述信息，指示 middleware 等待 Store 上指定的 action。
Generator 会暂停，直到一个与 `pattern` 匹配的 action 被发起。

用以下规则来解释 `pattern`：

- 如果调用 `take` 时参数为空，或者传入 `'*'`，那将会匹配所有发起的 action（例如，`take()` 会匹配所有的 action）。

- 如果是一个函数，action 会在 `pattern(action)` 返回为 true 时被匹配（例如，`take(action => action.entities)` 会匹配那些 `entities` 字段为真的 action）。

- 如果是一个字符串，action 会在 `action.type === pattern` 时被匹配（例如，`take(INCREMENT_ASYNC)`）。

- 如果参数是一个数组，会针对数组所有项，匹配与 `action.type` 相等的 action（例如，`take([INCREMENT, DECREMENT])` 会匹配 `INCREMENT` 或 `DECREMENT` 类型的 action）。


### `put(action)`

创建一条 Effect 描述信息，指示 middleware 发起一个 action 到 Store。

- `action: Object` - [完整信息可查看 Redux 的 `dispatch` 文档](http://redux.js.org/docs/api/Store.html#dispatch)

#### 注意

`put` 执行是异步的。即作为一个单独的 microtask，因此不会立即发生。

### `call(fn, ...args)`

创建一条 Effect 描述信息，指示 middleware 调用 `fn` 函数并以 `args` 为参数。

- `fn: Function` - 一个 Generator 函数, 或者返回 Promise 的普通函数

- `args: Array<any>` - 一个数组，作为 `fn` 的参数

#### 注意

`fn` 既可以是一个*普通*函数，也可以是一个 Generator 函数。

middleware 调用这个函数并检查它的结果。

如果结果是一个 Generator 对象，middleware 会执行它，就像在启动 Generator （startup Generators，启动时被传给 middleware）时做的。
如果有子级 Generator，那么在子级 Generator 正常结束前，父级 Generator 会暂停，这种情况下，父级 Generator 将会在子级 Generator 返回后继续执行，或者直到子级 Generator 被某些错误中止，
如果是这种情况，将在父级 Generator 中抛出一个错误。

如果结果是一个 Promise，middleware 会暂停直到这个 Promise 被 resolve，resolve 后 Generator 会继续执行。
或者直到 Promise 被 reject 了，如果是这种情况，将在 Generator 中抛出一个错误。

当 Generator 中抛出了一个错误，如果有一个 `try/catch` 包裹当前的 yield 指令，控制权将被转交给 `catch`。
否则，Generator 会被错误中止，并且如果这个 Generator 被其他 Generator 调用了，错误将会传到调用的 Generator。

### `call([context, fn], ...args)`

类似 `call(fn, ...args)`，但支持为 `fn` 指定 `this` 上下文。用于调用对象的方法。

### `apply(context, fn, args)`

类似 `call([context, fn], ...args)`

### `cps(fn, ...args)`

创建一条 Effect 描述信息，指示 middleware 以 Node 风格调用 `fn` 函数。

- `fn: Function` - 一个 Node 风格的函数。即不仅接受它自己的参数，`fn` 结束后会调用一个附加的回调函数。回调函数接受两个参数，第一个参数是报错信息，第二个是成功的结果。

- `args: Array<any>` - 一个数组，作为 `fn` 的参数

#### 注意

middleware 将执行 `fn(...arg, cb)`。`cb` 是被 middleware 传给 `fn` 的回调函数。如果 `fn` 正常结束，会调用 `cb(null, result)` 通知 middleware 成功了。
如果 `fn` 遇到了某些错误，会调用 `cb(error)` 通知 middleware 出错了。

`fn` 结束之前 middleware 会保持暂停状态。

### `cps([context, fn], ...args)`

支持为 `fn` 指定 `this` 上下文（调用对象方法）。

### `fork(fn, ...args)`

创建一条 Effect 描述信息，指示 middleware 以 *无阻塞调用* 方式执行 `fn`。

#### 参数

- `fn: Function` - 一个 Generator 函数, 或者返回 Promise 的普通函数

- `args: Array<any>` - 一个数组，作为 `fn` 的参数

#### 注意

`fork` 类似于 `call`，可以用来调用普通函数和 Generator 函数。但 `fork` 的调用是无阻塞的，在等待 `fn` 返回结果时，middleware 不会暂停 Generator。
相反，一旦 `fn` 被调用，Generator 立即恢复执行。

`fork` 与 `race` 类似，是一个中心化的 Effect，管理 Sagas 间的并发。

`yield fork(fn ...args)` 的结果是一个 [Task](#task) 对象 —— 一个具备某些有用的方法和属性的对象。


### `fork([context, fn], ...args)`

支持为 `fn` 指定 `this` 上下文（调用对象方法）。

### `join(task)`

创建一条 Effect 描述信息，指示 middleware 等待之前的 fork 任务返回结果。

- `task: Task` - 之前的 `fork` 指令返回的 [Task](#task) 对象

### `cancel(task)`

创建一条 Effect 描述信息，指示 middleware 取消之前的 fork 任务。

- `task: Task` - 之前的 `fork` 指令返回的 [Task](#task) 对象

#### 注意

取消执行中的 Generator，middleware 将会抛出一个 `SagaCancellationException` 的错误。

取消会向下传播。当取消 Generator 时，middleware 会同时取消当前 Effect（阻塞当前 Generator 的 Effect）。
如果当前 Effect 调用了另一个 Generator，那这个 Generator 也会被取消。

被取消的 Generator 可以捕获 `SagaCancellationException` 错误，以便在它结束前执行某些清理逻辑（例如，如果 Generator 正处于 AJAX 调用，清除 state 中的 `isFetching` 标识）。

注意，未被捕获的 `SagaCancellationException` 不会向上冒泡，如果 Generator 不处理取消异常，异常将不会冒泡至它的父级 Generator。

`cancel` 是一个无阻塞 Effect。也就是说，Generator 将在取消异常被抛出后立即恢复。

对于返回 Promise 结果的函数，你可以通过给 promise 附加一个 `[CANCEL]` 来插入自己的取消逻辑。

以下的例子演示了如何给 Promise 结果附加一个取消逻辑：

```javascript
import { fork, cancel, CANCEL } from 'redux-saga/effects'

function myApi() {
  const promise = myXhr(...)

  promise[CANCEL] = () => myXhr.abort()
  return promise
}

function* mySaga() {

  const task = yield fork(myApi)

  // ... 过一会儿
  // 将会调用 myApi 上的 promise[CANCEL]
  yield cancel(task)
}
```

### `select(selector, ...args)`

创建一条 Effect 描述信息，指示 middleware 调用提供的选择器获取 Store state 上的数据（例如，返回 `selector(getState(), ...args)` 的结果）。

- `selector: Function` - 一个 `(state, ...args) => args` 函数. 通过当前 state 和一些可选参数，返回当前 Store state 上的部分数据。

- `args: Array<any>` - 可选参数，传递给选择器（附加在 `getState` 后）

如果 `select` 调用时参数为空（即 `yield select()`），那 effect 会取得整个的 state（和调用 `getState()` 的结果一样）。

>重要提醒：在发起 action 到 store 时，middleware 首先会转发 action 到 reducers 然后通知 Sagas。这意味着，当你查询 Store 的 state，
你获取的是 action 被处理之后的 state。

#### 注意

Saga 最好是自主独立的，不应依赖 Store 的 state。这使得很容易修改 state 实现部分而不影响 Saga 代码。
Saga 最好只依赖它自己内部的控制 state，并尽可能这样做。但有时，我们可能会发现在 Saga 中查询 state 而不是自行维护所需的数据，会更加方便（比如，当一个 Saga 重复调用 reducer 的逻辑，来计算那些已经被 Store 计算的 state）。

一般来说，*顶级 Sagas* (middleware 启动的 Sagas) 会被传入 Store 的 `getState` 方法，这个方法允许 Sagas 查询当前 Store 的 state。
但是 `getState` 函数必须在 *嵌套 Sagas*（其他 Sagas 启动的子级 Sagas）时被传来传去，以便访问 state。

举个例子，假设我们我们的应用有这样结构的一份 state：

```javascript
state = {
  cart: {...}
}
```

并且有这样的 Sagas：

`./sagas.js`
```javascript
import { take, fork, ... } from 'redux-saga/effects'

function* checkout(getState) {
  // 必须知道 `cart` 存在哪
  const cart = getState().cart

  // ... 调用某些 Api 然后发起一个 success/error action
}

// rootSaga 会在 middleware 自动它时，自动获得 getState 参数
export default function* rootSaga(getState) {
  while(true) {
    yield take('CHECKOUT_REQUEST')

    // 需要将 getState 传递给子级 Saga
    yield fork(checkout, getState)
  }
}
```

以上代码有一个问题，除了 `getState` 需要显式地向下传递给 call/fork 调用链，`checkout` 还必须知道 `cart` 被存在全局 state 的哪个地方（比如，在 `cart` 字段里）。
Saga 与 State 结构藕合了。如果未来 state 结构改变了（比如通过某些正常处理流程），那我们也必须小心更改 `checkout` Saga 里面的代码。
如果我们还有另外的 Sagas 也访问了 `cart`，并且如果我们忘记更新其中一个依赖的 Sagas，那就会引起一些细小的错误。

为了缓解这种情况，我们可心创建一个 *选择器*（selector），即一个知道如何从 State 提取 `cart` 数据的函数。

`./selectors`
```javascript
export const getCart = state => state.cart
```

然后我们可以在 `checkout` Saga 中使用这个选择器：

`./sagas.js`
```javascript
import { take, fork, select } from 'redux-saga/effects'
import { getCart } from './selectors'

function* checkout() {
  // 使用选择器查询 state
  const cart = yield select(getCart)

  // ... 调用某些 Api 然后发起一个 success/error action
}

export default function* rootSaga() {
  while(true) {
    yield take('CHECKOUT_REQUEST')

    // 不再需要传递 getState 参数了
    yield fork(checkout)
  }
}
```

首先要注意的是，`rootSaga` 没有使用 `getState` 参数。没有必要为子级 Saga 传递 `getState`。
`rootSaga` 再也不需要做关于 `checkout` 的假设（即：`checkout` 是否需要访问 state）。

其次，`checkout` 现在可以通过 `select(getCart)` 直接得到需要的信息。Saga 现在只与 `getCart` 选择器藕合。
如果我们有许多需要访问 `cart` 的 Sagas（或 React Components），它们都将藕合到统一的 `getCart` 函数。
并且如果我们改变了 state 结构，我们只需要更新 `getCart` 即可。

## Effect combinators
----------------------------

### `race(effects)`

Creates an Effect description that instructs the middleware to run a *Race* between
multiple Effects (this is similar to how `Promise.race([...])` behaves).

`effects: Object` - a dictionary Object of the form {label: effect, ...}

#### Example

The following example run a race between 2 effects :

1. A call to a function `fetchUsers` which returns a Promise
2. A `CANCEL_FETCH` action which may be eventually dispatched on the Store

```javascript
import { take, call } from `redux-saga/effects`
import fetchUsers from './path/to/fetchUsers'

function* fetchUsersSaga {
  const { response, cancel } = yield race({
    response: call(fetchUsers),
    cancel: take(CANCEL_FETCH)
  })
}
```

If `call(fetchUsers)` resolves (or rejects) first, the result of `race` will be an object
with a single keyed object `{response: result}` where `result` is the resolved result of `fetchUsers`.

If an action of type `CANCEL_FETCH` is dispatched on the Store before `fetchUsers` completes, the result
will be a single keyed object `{cancel: action}`, where action is the dispatched action.

#### Notes

When resolving a `race`, the middleware automatically cancels all the losing Effects.


### `[...effects] (parallel effects)`

Creates an Effect description that instructs the middleware to run multiple Effects
in parallel and wait for all of them to complete.

#### Example

The following example run 2 blocking calls in parallel :

```javascript
import { fetchCustomers, fetchProducts } from './path/to/api'

function* mySaga() {
  const [customers, products] = yield [
    call(fetchCustomers),
    call(fetchProducts)
  ]
}
```

#### Notes

When running Effects in parallel, the middleware suspends the Generator until one
of the followings :  

- All the Effects completed with success: resumes the Generator with an array containing
the results of all Effects.

- One of the Effects was rejected before all the effects complete: throw the rejection
error inside the Generator.

## Interfaces
---------------------

### Task

The Task interface specifies the result of running a Saga using `fork`, `middleware.run` or `runSaga`


<table id="task-descriptor">
  <tr>
    <th>method</th>
    <th>return value</th>
  </tr>
  <tr>
    <td>task.isRunning()</td>
    <td>true if the task hasn't yet returned or thrown an error</td>
  </tr>
  <tr>
    <td>task.result()</td>
    <td>task return value. `undefined` if task is still running</td>
  </tr>
  <tr>
    <td>task.error()</td>
    <td>task thrown error. `undefined` if task is still running</td>
  </tr>
  <tr>
    <td>task.done</td>
    <td>
      a Promise which is either:
        <ul>
          <li>resolved with task's return value</li>
          <li>rejected with task's thrown error</li>
        </ul>
      </td>
  </tr>
  <tr>
    <td>task.cancel()</td>
    <td>Cancels the task (If it is still running)</td>
  </tr>
</table>




## External API
------------------------

### `runSaga(iterator, {subscribe, dispatch, getState}, [monitor])`

Allows starting sagas outside the Redux middleware environment. Useful if you want to
connect a Saga to external input/output, other than store actions.

`runSaga` returns a Task object. Just like the one returned from a `fork` effect.


- `iterator: {next, throw}` - an Iterator object, Typically created by invoking a Generator function

- `{subscribe, dispatch, getState}: Object` - an Object which exposes `subscribe`, `dispatch` and `getState` methods

  - `subscribe(callback): Function` - A function which accepts a callback and returns an `unsubscribe` function

    - `callback(input): Function` - callback(provided by runSaga) used to subscribe to input events. `subscribe` must support registering multiple subscriptions.
      - `input: any` - argument passed by `subscribe` to `callback` (see Notes below)

    - `dispatch(output): Function` - used to fulfill `put` effects.
      - `output : any` -  argument provided by the Saga to the `put` Effect (see Notes below).

    - `getState() : Function` - used to fulfill `select` and `getState` effects

- `monitor(sagaAction): Function` (optional): a callback which is used to dispatch all Saga related events. In the middleware version, all actions are dispatched to the Redux store. See the [sagaMonitor example](https://github.com/yelouafi/redux-saga/tree/master/examples/sagaMonitor) for usage.
  - `sagaAction: Object` - action dispatched by Sagas to notify `monitor` of Saga related events.

#### Notes

The `{subscribe, dispatch}` is used to fulfill `take` and `put` Effects. This defines the Input/Output
interface of the Saga.

`subscribe` is used to fulfill `take(PATTERN)` effects. It must call `callback` every time it
has an input to dispatch (e.g. on every mouse click if the Saga is connected to DOM click events).
Each time `subscribe` emits an input to its callbacks, if the Saga is blocked on a `take` effect, and
if the take pattern matches the currently incoming input, the Saga is resumed with that input.

`dispatch` is used to fulfill `put` effects. Each time the Saga emits a `yield put(output)`, `dispatch`
is invoked with output.
