# API 参考

* [`Middleware API`](#middleware-api)
  * [`createSagaMiddleware(options)`](#createsagamiddlewareoptions)
  * [`middleware.run(saga, ...args)`](#middlewarerunsaga-args)
* [`Saga 辅助函数`](#saga-辅助函数)
  * [`takeEvery(pattern, saga, ...args)`](#takeeverypattern-saga-args)
  * [`takeEvery(channel, saga, ...args)`](#takeeverychannel-saga-args)
  * [`takeLatest(pattern, saga, ..args)`](#takelatestpattern-saga-args)
  * [`takeLatest(channel, saga, ..args)`](#takelatestchannel-saga-args)
  * [`takeLeading(pattern, saga, ..args)`](#takeleadingpattern-saga-args)
  * [`takeLeading(channel, saga, ..args)`](#takeleadingchannel-saga-args)
  * [`throttle(ms, pattern, saga, ..args)`](#throttlems-pattern-saga-args)
* [`Effect 创建器`](#effect-创建器)
  * [`take(pattern)`](#takepattern)
  * [`take.maybe(pattern)`](#takemaybepattern)
  * [`take(channel)`](#takechannel)
  * [`take.maybe(channel)`](#takemaybechannel)
  * [`put(action)`](#putaction)
  * [`put.resolve(action)`](#putresolveaction)
  * [`put(channel, action)`](#putchannel-action)
  * [`call(fn, ...args)`](#callfn-args)
  * [`call([context, fn], ...args)`](#callcontext-fn-args)
  * [`call([context, fnName], ...args)`](#callcontext-fnname-args)
  * [`apply(context, fn, args)`](#applycontext-fn-args)
  * [`cps(fn, ...args)`](#cpsfn-args)
  * [`cps([context, fn], ...args)`](#cpscontext-fn-args)
  * [`fork(fn, ...args)`](#forkfn-args)
  * [`fork([context, fn], ...args)`](#forkcontext-fn-args)
  * [`spawn(fn, ...args)`](#spawnfn-args)
  * [`spawn([context, fn], ...args)`](#spawncontext-fn-args)
  * [`join(task)`](#jointask)
  * [`join(...tasks)`](#jointasks)
  * [`cancel(task)`](#canceltask)
  * [`cancel(...tasks)`](#canceltasks)
  * [`cancel()`](#cancel)
  * [`select(selector, ...args)`](#selectselector-args)
  * [`actionChannel(pattern, [buffer])`](#actionchannelpattern-buffer)
  * [`flush(channel)`](#flushchannel)
  * [`cancelled()`](#cancelled)
  * [`setContext(props)`](#setcontextprops)
  * [`getContext(prop)`](#getcontextprop)
* [`Effect 组合器（combinators）`](#effect-组合器combinators)
  * [`race(effects)`](#raceeffects)
  * [`race([...effects])`](#raceeffects-with-array)
  * [`all([...effects]) (aka parallel effects)`](#alleffects---parallel-effects)
  * [`all(effects)`](#alleffects)
* [`接口`](#接口)
  * [`Task`](#task)
  * [`Channel`](#channel)
  * [`Buffer`](#buffer)
  * [`SagaMonitor`](#sagamonitor)
* [`外部 API`](#外部-api)
  * [`runSaga(options, saga, ...args)`](#runsagaoptions-saga-args)
* [`工具`](#工具)
  * [`channel([buffer])`](#channelbuffer)
  * [`eventChannel(subscribe, [buffer], matcher)`](#eventchannelsubscribe-buffer-matcher)
  * [`buffers`](#buffers)
  * [`delay(ms, [val])`](#delayms-val)
  * [`cloneableGenerator(generatorFunc)`](#cloneablegeneratorgeneratorfunc)
  * [`createMockTask()`](#createmocktask)


# 速查表

* [阻塞 / 非阻塞](#阻塞--非阻塞)

## Middleware API

### `createSagaMiddleware(options)`

创建一个 Redux middleware，并将 Sagas 连接到 Redux Store。

- `options: Object` - 传递给 middleware 的选项列表。目前支持的选项有:

  - `sagaMonitor` : [SagaMonitor](#sagamonitor) - 如果提供了 Saga Monitor, middleware 将向 monitor 传送监视事件。

 - `emitter` : 用于从 redux 向 redux-saga 进给 actions。Emitter 是一个高阶函数（high order function），它接受一个内置 emitter 并返回另一个 emitter。

      **例子**

      在下面的示例中，我们创建了一个 emitter，它将拆开 actions 列表，并发送从中提取的每个 action。

     ```javascript
     createSagaMiddleware({
       emitter: emit => action => {
        if (Array.isArray(action)) {
          action.forEach(emit);
          return
        }
        emit(action);
       }
     });
     ```

  - `logger` : Function -  为 middleware 定义一个自定义的日志方法。默认情况下，middleware 会把所有的错误和警告记录到控制台中。此选项告诉 middleware 把错误/警告发送到我们所提供的替代日志方法中。调用该日志方法的参数为 `(level, ...args)`。第一个参数表示日志的级别（``info``、``warning`` 或 ``error``）。其余的对应后面的参数（你可以使用  `args.join(' ')` 将所有的参数拼接成单个字符串）。

  - `onError` : Function - 当提供该方法时，middleware 将带着 Sagas 中未被捕获的错误调用它。这个参数在向错误跟踪服务发送未被捕获的异常时非常有用。

#### 例子

下面，我们将创建一个函数 `configureStore`，它将使用一个新的方法 `runSaga` 来增强 Store。然后我们将在主模块中使用该函数来启动应用的顶级 Saga（root Saga）。

**configureStore.js**
```javascript
import createSagaMiddleware from 'redux-saga'
import reducer from './path/to/reducer'

export default function configureStore(initialState) {
  // 注意：必须满足 redux@>=3.1.0 才可以将 middleware 作为 createStore 的最后一个参数传递
  const sagaMiddleware = createSagaMiddleware()
  return {
    ...createStore(reducer, initialState, applyMiddleware(/* 其它 middleware, */sagaMiddleware)),
    runSaga: sagaMiddleware.run
  }
}
```

**main.js**
```javascript
import configureStore from './configureStore'
import rootSaga from './sagas'
// ... 其它 imports

const store = configureStore()
store.runSaga(rootSaga)
```

#### 注意事项

请阅读下面关于 `sagaMiddleware.run` 方法的更多信息。

### `middleware.run(saga, ...args)`

动态地运行 `saga`。**只能** 用于在 `applyMiddleware` 阶段 **之后** 执行 Saga。

- `saga: Function`: 一个 Generator 函数
- `args: Array<any>`: 提供给 `saga` 的参数

该方法返回一个 [Task 描述对象](#task-descriptor).

#### 注意事项

`saga` 必须是一个返回 [Generator 对象](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator) 的函数。middleware 会迭代这个 Generator 并执行所有 yield 后的 Effect。（译注：Effect 可以看作是 redux-saga 的任务单元，参考 [名词解释](http://superRaytin.github.io/redux-saga-in-chinese/docs/Glossary.html)）。

`saga` 也可以使用库中提供的各种 Effect 来启动其他 saga。下述的迭代过程也适用于所有的子级 saga。

在第一次迭代里，middleware 会调用 `next()` 方法来获取下一个 Effect。然后 middleware 按照后续 Effects API 所指定的方式来执行 yield 后的 Effect。
与此同时，Generator 将被暂停，直到 effect 执行结束。在接收到执行的结果时，middleware 在 Generator 里接着调用 `next(result)`，并将得到的结果作为参数传入。
这个过程会一直重复，直到 Generator 正常终止或抛出错误。

如果执行导致了错误（由各个 Effect 创建器定义），则会调用 Generator 的 `throw(error)` 方法来代替。如果 Generator 函数定义了一个 `try/catch` 包裹当前的 yield 指令，那么 `catch` 区块将被底层 Generator 运行时（runtime）调用。运行时还将调用所有相应的 `finally` 区块。

在 Saga 被取消（手动或使用所提供的 Effect）的情况下，middleware 将调用 Generator 的 `return()` 方法。这将导致 Generator 直接跳到 `finally` 区块。

## Saga 辅助函数

> 注意: 下列函数都是构建在以下 Effect 创建器之上的辅助函数。（译注：即高级 API）

### `takeEvery(pattern, saga, ...args)`

在发起（dispatch）到 Store 并且匹配 `pattern` 的每一个 action 上派生一个 `saga`。

- `pattern: String | Array | Function` - 有关更多信息，请参见 [`take(pattern)`](#takepattern) 的文档

- `saga: Function` - 一个 Generator 函数

- `args: Array<any>` - 传递给启动任务的参数。`takeEvery` 会把当前的 action 追加到参数列表中。（即 action 将是 `saga` 的最后一个参数）

#### 例子

在下面的示例中，我们创建了一个简单的任务 `fetchUser`。我们在每次 `USER_REQUESTED` action 被发起时，使用 `takeEvery` 来启动一个新的 `fetchUser` 任务。

```javascript
import { takeEvery } from `redux-saga/effects`

function* fetchUser(action) {
  ...
}

function* watchFetchUser() {
  yield takeEvery('USER_REQUESTED', fetchUser)
}
```

#### 注意事项

`takeEvery` 是一个使用 `take` 和 `fork` 构建的高级 API。下面演示了这个辅助函数是如何由低级 Effect 实现的：

```javascript
const takeEvery = (patternOrChannel, saga, ...args) => fork(function*() {
  while (true) {
    const action = yield take(patternOrChannel)
    yield fork(saga, ...args.concat(action))
  }
})
```

`takeEvery` 允许处理并发的 action（译注：即同时触发相同的 action）。在上面的例子里，当发起一个 `USER_REQUESTED` action 时，即使前一个 `fetchUser` 任务还未处理结束，也将会启动一个新的 `fetchUser` 任务。
（举个例子，用户以极快的速度连续点击一个 `Load User` 按钮 2 次，即使第一个触发的 `fetchUser` 任务还未结束，第二次点击依然会发起一个 `USER_REQUESTED` action。）

`takeEvery` 不会对多个任务的响应进行排序，并且不保证任务将会以它们启动的顺序结束。如果要对响应进行排序，可以关注以下的 `takeLatest`。

### `takeEvery(channel, saga, ...args)`

你还可以将 channel 作为参数传入，其行为与 [takeEvery(pattern, saga, ...args)](#takeeverypattern-saga-args) 一致。

### `takeLatest(pattern, saga, ...args)`

在发起到 Store 并且匹配 `pattern` 的每一个 action 上派生一个 `saga`。并自动取消之前所有已经启动但仍在执行中的 `saga` 任务。 

每当一个 action 被发起到 Store，并且匹配 `pattern` 时，则 `takeLatest` 将会在后台启动一个新的 `saga` 任务。
如果此前已经有一个 `saga` 任务启动了（在当前 action 之前发起的最后一个 action），并且仍在执行中，那么这个任务将被取消。

- `pattern: String | Array | Function` - 有关更多信息，请参见 [`take(pattern)`](#takepattern) 的文档

- `saga: Function` - 一个 Generator 函数

- `args: Array<any>` - 传递给启动任务的参数。`takeLatest` 会把当前的 action 追加到参数列表中。（即 action 将是 `saga` 的最后一个参数）

#### 例子

在下面的示例中，我们创建了一个简单的任务 `fetchUser`。我们在每次 `USER_REQUESTED` action 被发起时，使用 `takeLatest` 来启动一个新的 `fetchUser` 任务。
由于 `takeLatest` 取消了所有之前启动且未完成的任务，这样便可以保证：即使用户以极快的速度连续多次触发 `USER_REQUESTED` action，我们都只会以最后的一个结束。

```javascript
import { takeLatest } from `redux-saga/effects`

function* fetchUser(action) {
  ...
}

function* watchLastFetchUser() {
  yield takeLatest('USER_REQUESTED', fetchUser)
}
```

#### 注意事项

`takeLatest` 是一个使用 `take` 和 `fork` 构建的高级 API。下面演示了这个辅助函数是如何由低级 Effect 实现的：

```javascript
const takeLatest = (patternOrChannel, saga, ...args) => fork(function*() {
  let lastTask
  while (true) {
    const action = yield take(patternOrChannel)
    if (lastTask) {
      yield cancel(lastTask) // 如果任务已经结束，cancel 则是空操作
    }
    lastTask = yield fork(saga, ...args.concat(action))
  }
})
```

### `takeLatest(channel, saga, ...args)`

你还可以将 channel 作为参数传入，其行为与 [takeLatest(pattern, saga, ...args)](#takelatestpattern-saga-args) 一致。

### `takeLeading(pattern, saga, ...args)`

在发起到 Store 并且匹配 `pattern` 的每一个 action 上派生一个 `saga`。
它将在派生一次任务之后阻塞，直到派生的 saga 完成，然后又再次开始监听指定的 `pattern`。

简而言之，`takeLeading` 只在没有 saga 运行的时候才监听 action。

- `pattern: String | Array | Function` - 有关更多信息，请参见 [`take(pattern)`](#takepattern) 的文档

- `saga: Function` - 一个 Generator 函数

- `args: Array<any>` - 传递给启动任务的参数。`takeLeading` 会把当前的 action 追加到参数列表中。（即 action 将是 `saga` 的最后一个参数）

#### 例子

在下面的示例中，我们创建了一个简单的任务 `fetchUser`。我们在每次 `USER_REQUESTED` action 被发起时，使用 `takeLeading` 来启动一个新的 `fetchUser` 任务。
由于 `takeLeading` 在其开始之后便无视所有新传入的任务，我们便可以保证：如果用户以极快的速度连续多次触发 `USER_REQUESTED` action，我们都只会保持以第一个 action 运行。

```javascript
import { takeLeading } from `redux-saga/effects`

function* fetchUser(action) {
  ...
}

function* watchLastFetchUser() {
  yield takeLeading('USER_REQUESTED', fetchUser)
}
```

#### 注意事项

`takeLeading` 是一个使用 `take` 和 `call` 构建的高级 API。下面演示了这个辅助函数是如何由低级 Effect 实现的：

```javascript
const takeLeading = (patternOrChannel, saga, ...args) => fork(function*() {
  while (true) {
    const action = yield take(patternOrChannel);
    yield call(saga, ...args.concat(action));
  }
})
```

### `takeLeading(channel, saga, ...args)`

你还可以将 channel 作为参数传入，其行为与 [takeLeading(pattern, saga, ...args)](#takeleadingpattern-saga-args) 一致。

### `throttle(ms, pattern, saga, ...args)`

在发起到 Store 并且匹配 `pattern` 的一个 action 上派生一个 `saga`。
它在派生一次任务之后，仍然将新传入的 action 接收到底层的 `buffer` 中，至多保留（最近的）一个。但与此同时，它在 `ms` 毫秒内将暂停派生新的任务 —— 这也就是它被命名为节流阀（`throttle`）的原因。其用途，是在处理任务时，无视给定的时间内新传入的 action。

- `ms: Number` - 在 action 开始处理后，无视新 action 的时长；以毫秒为单位。

- `pattern: String | Array | Function` - 有关更多信息，请参见 [`take(pattern)`](#takepattern) 的文档

- `saga: Function` - 一个 Generator 函数

- `args: Array<any>` - 传递给启动任务的参数。`throttle` 会把当前的 action 追加到参数列表中。（即 action 将是 `saga` 的最后一个参数）

#### 例子

在下面的示例中，我们创建了一个简单的任务 `fetchAutocomplete`。我们在 `FETCH_AUTOCOMPLETE` action 被发起时，使用 `throttle` 来启动一个新的 `fetchAutocomplete` 任务。
不过由于 `throttle` 无视了一段时间内连续的 `FETCH_AUTOCOMPLETE`，我们便可以确保用户不会因此向我们的服务器发起大量请求。

```javascript
import { call, put, throttle } from `redux-saga/effects`

function* fetchAutocomplete(action) {
  const autocompleteProposals = yield call(Api.fetchAutocomplete, action.text)
  yield put({type: 'FETCHED_AUTOCOMPLETE_PROPOSALS', proposals: autocompleteProposals})
}

function* throttleAutocomplete() {
  yield throttle(1000, 'FETCH_AUTOCOMPLETE', fetchAutocomplete)
}
```

#### 注意事项

`throttle` 是一个使用 `take`、`fork` 和 `actionChannel` 构建的高级 API。下面演示了这个辅助函数是如何由低级 Effect 实现的：

```javascript
const throttle = (ms, pattern, task, ...args) => fork(function*() {
  const throttleChannel = yield actionChannel(pattern)

  while (true) {
    const action = yield take(throttleChannel)
    yield fork(task, ...args, action)
    yield delay(ms)
  }
})

```

## Effect 创建器

> 注意:

> - 以下每个函数都会返回一个普通 Javascript 对象（plain JavaScript object），并且不会执行任何其它操作。
> - 执行是由 middleware 在上述迭代过程中进行的。
> - middleware 会检查每个 Effect 的描述信息，并进行相应的操作

### `take(pattern)`

创建一个 Effect 描述信息，用来命令 middleware 在 Store 上等待指定的 action。
在发起与 `pattern` 匹配的 action 之前，Generator 将暂停。

我们用以下规则来解释 `pattern`：

- 如果以空参数或 `'*'` 调用 `take`，那么将匹配所有发起的 action。（例如，`take()` 将匹配所有 action）

- 如果它是一个函数，那么将匹配 `pattern(action)` 为 true 的 action。（例如，`take(action => action.entities)` 将匹配哪些 `entities` 字段为真的 action）
> 注意: 如果 pattern 函数上定义了 `toString`，`action.type` 将改用 `pattern.toString` 来测试。这个设定在你使用 action 创建函数库（如 redux-act 或 redux-actions）时非常有用。

- 如果它是一个字符串，那么将匹配 `action.type === pattern` 的 action。（例如，`take(INCREMENT_ASYNC)`）

- 如果它是一个数组，那么数组中的每一项都适用于上述规则 —— 因此它是支持字符串与函数混用的。不过，最常见的用例还属纯字符串数组，其结果是用 `action.type` 与数组中的每一项相对比。（例如，`take([INCREMENT, DECREMENT])` 将匹配 `INCREMENT` 或 `DECREMENT` 类型的 action）

middleware 提供了一个特殊的 action —— `END`。如果你发起 END action，则无论哪种 pattern，只要是被 take Effect 阻塞的 Sage 都会被终止。假如被终止的 Saga 下仍有分叉（forked）任务还在运行，那么它在终止任务前，会先等待其所有子任务均被终止。

### `take.maybe(pattern)`

与 `take(pattern)` 相同，但在 `END` action 时不自动地终止 Saga。与所有在 take Effect 上阻塞的 Saga 都将获得 `END` 对象的规则相反。

#### 注意事项

`take.maybe` 的命名来自于函数式编程 —— 就好比我们可以使用 `Maybe(ACTION)` 类型代替（自动处理的）`ACTION`，使我们得以处理以下两种情况：

- 存在 `Just(ACTION)` 的情况（我们有一个 action）
- `NOTHING` 的情况（channel 已经被关闭）。例如，我们需要一些途径来映射 `END`。

* 在内部，所有 `dispath` 过的 action 都会经过 `stdChannel`；而当 `dispatch(END)` 发生时，`stdChannel` 则会被关闭。

### `take(channel)`

创建一个 Effect 描述信息，用来命令 middleware 从指定的 Channel 中等待一条特定消息。
如果 channel 已经被关闭，那么 Generator 将以与上面 `take(pattern)` 所描述一致的步骤马上终止。

### `take.maybe(channel)`

与 `take(channel)` 相同，但在 `END` action 时不自动地终止 Saga。与所有在 take Effect 上阻塞的 Saga 都将获得 `END` 对象的规则相反。有关更多信息，请参见 [这里](#takemaybepattern)。

### `put(action)`

创建一个 Effect 描述信息，用来命令 middleware 向 Store 发起一个 action。
这个 effect 是非阻塞型的，并且所有向下游抛出的错误（例如在 reducer 中），都不会冒泡回到 saga 当中。

- `action: Object` - [有关完整信息，请参见 Redux `dispatch` 的文档](http://redux.js.org/docs/api/Store.html#dispatch)

### `put.resolve(action)`

类似 [`put`](#putaction)，但 effect 是阻塞型的（如果从 `dispatch` 返回了 promise，它将会等待其结果），并且会从下游冒泡错误。

- `action: Object` - [有关完整信息，请参见 Redux `dispatch` 的文档](http://redux.js.org/docs/api/Store.html#dispatch)

### `put(channel, action)`

创建一个 Effect 描述信息，用来命令 middleware 向指定的 channel 中放入一条 action。

- `channel: Channel` - 一个 [`Channel`](#channel) 对象.
- `action: Object` - [有关完整信息，请参见 Redux `dispatch` 的文档](http://redux.js.org/docs/api/Store.html#dispatch)

当 put **没有** 被缓存而是被 taker 立即消费掉的时候，这个 effect 是阻塞型的。假如有错误被抛到了这些 taker 当中，那这个错误将会冒泡回到 saga 里面。

### `call(fn, ...args)`

创建一个 Effect 描述信息，用来命令 middleware 以参数 `args` 调用函数 `fn` 。

- `fn: Function` - 一个 Generator 函数, 也可以是一个返回 Promise 或任意其它值的普通函数。
- `args: Array<any>` - 传递给 `fn` 的参数数组。

#### 注意事项

`fn` 即可以是一个 **普通** 函数，也可以是一个 Generator 函数。

middleware 会调用该函数，并检查其结果。

如果其结果是一个迭代器对象（Iterator object），那么 middleware 将会执行这个 Generator 函数 —— 正如它对待 startup Generator（该 Generator 会在启动时被传递给 middleware）那样。在子级 Generator 正常结束或遭遇某些错误而中断之前，父级 Generator 将被一直暂停 —— 在前者的情况下，父级 Generator 会在子级 Generator 返回值后带着该值恢复执行；而在后者的情况下，将在父级 Generator 中抛出一个错误。

如果其结果是一个 Promise，那么在该 Promise 被 resolve 或 reject 之前，middleware 都将一直暂停 Generator —— 在前者的情况下，Generator 会在 resolve 之后带着其返回值恢复执行；而在后者的情况下，将在 Generator 中抛出一个错误。

如果其结果既不是迭代器对象也不是 Promise，那么 middleware 会立即把该值返回给 saga，从而让它可以以同步的形式地恢复执行。

当 Generator 中抛出了一个错误时，假如有使用 `try/catch` 区块包裹当前的 `yield` 指令，那么控制权将会被转交给 `catch` 区块。否则，Generator 会因错误而中断，并且假如这个 Generator 是由其它 Generator 调用的话，那么错误还会被传递给该调用方。

### `call([context, fn], ...args)`

类似 `call(fn, ...args)`，但支持传递 `this` 上下文给 `fn`。在调用对象方法时很有用。

### `call([context, fnName], ...args)`

类似 `call([context, fn], ...args)`，但支持用字符串传递 `fn`。在调用对象的方法时很有用。例如 `yield call([localStorage, 'getItem'], 'redux-saga')`。

### `apply(context, fn, [args])`

`call([context, fn], ...args)` 的另一种写法。

### `cps(fn, ...args)`

创建一个 Effect 描述信息，用来命令 middleware 以 Node 风格的函数（Node style function）的方式调用 `fn`。

- `fn: Function` - 一个 Node 风格的函数。即除了接受其自身参数之外，还接受一个在 `fn` 执行结束后会被调用的附加回调函数。该回调函数接受两个参数，第一个参数用于报告错误，而第二个用于报告成功的结果。

- `args: Array<any>` - 传递给 `fn` 的参数数组。

#### 注意事项

middleware 将执行 `fn(...arg, cb)`。其中 `cb` 是由 middleware 传递给 `fn` 的回调函数。如果 `fn` 正常结束，则必定会调用 `cb(null, result)`，从而告知 middleware 成功的结果。如果 `fn` 遇到了错误，则必定会调用 `cb(error)`，从而告知 middleware 出错了。

在 `fn` 终止之前，middleware 会保持暂停状态。

### `cps([context, fn], ...args)`

支持传递 `this` 上下文给 `fn`。（对象方法调用）

### `fork(fn, ...args)`

创建一个 Effect 描述信息，用来命令 middleware 以 **非阻塞调用** 的形式执行 `fn`。

#### 参数

- `fn: Function` - 一个 Generator 函数，或返回 Promise 的普通函数
- `args: Array<any>` - 传递给 `fn` 的参数数组。

返回一个 [Task](#task) 对象。

#### 注意事项

`fork` 类似于 `call`，可以用来调用普通函数和 Generator 函数。不过，`fork` 的调用是非阻塞的，Generator 不会在等待 `fn` 返回结果的时候被 middleware 暂停；恰恰相反地，它在 `fn` 被调用时便会立即恢复执行。

`fork`，以及 `race`，都是用于管理 Saga 间并发的中心化 Effect。

`yield fork(fn ...args)` 的结果是一个 [Task](#task) 对象 —— 一个具备着某些实用方法及属性的对象。

所有分叉任务（forked tasks）都会被附加（attach）到它们的父级任务身上。当父级任务终止其自身命令的执行，它会在返回之前等待所有分叉任务终止。

来自于子级任务的错误会自动地冒泡到它们的父级任务。如果有任何分叉任务引发了一个未被捕获的错误，那么父级任务会因该子级错误而中断，并且父级的整个执行树（即分叉任务 + 如果还在运行的由父体扮演的 **主任务**）也会被取消。

一个分叉任务的取消，将自动地取消所有还在执行的分叉任务。另外，这个被取消的任务在被阻塞时所处的当前 Effect（若有的话）也会被取消。

如果一个分叉任务以同步的形式失败（即在它执行完成后且未开始做异步操作前立即失败），则不会返回 Task，并且父级任务将被尽快地中断（由于父子任务均并行地运行，父级任务将在收到子级任务失败的通知时中断）。

若要创建 **被分离的（detached）** 分叉，请使用 `spawn` 代替。

### `fork([context, fn], ...args)`

支持使用 `this` 上下文调用分叉函数。

### `spawn(fn, ...args)`

与 `fork(fn, ...args)` 相同，但创建的是 **被分离的** 任务。被分离的任务与其父级任务保持独立，并像顶级任务般工作。父级任务不会在返回之前等待被分离的任务终止，并且所有可能影响父级或被分离的任务的事件都是完全独立的（错误、取消）。

### `spawn([context, fn], ...args)`

支持使用 `this` 上下文衍生（spawn）函数。

### `join(task)`

创建一个 Effect 描述信息，用来命令 middleware 等待之前的一个分叉任务的结果。

- `task: Task` - 由之前的 `fork` 指令返回的 [Task](#task) 对象

#### 注意事项

`join` 解出的结果（成功或错误）与被连接（join）的任务的相同。如果被连接的任务被取消，那么该取消信息还将冒泡到执行 join effect 的 Saga。同样地，这些连接者的任何潜在调用方也将一律被取消。

### `join(...tasks)`

创建一个 Effect 描述信息，用来命令 middleware 等待之前的多个分叉任务的结果。

- `tasks: Array<Task>` - [Task](#task) 是由之前 `fork` 指令返回的对象

#### 注意事项

它只是把任务数组包裹在 [join effects](#jointask) 中，大致相当于 `yield tasks.map(t => join(t))`。

### `cancel(task)`

创建一个 Effect 描述信息，用来命令 middleware 取消之前的一个分叉任务。

- `task: Task` - 由之前 `fork` 指令返回的 [Task](#task) 对象

#### 注意事项

若要取消正在运行的任务，middleware 将调用底层 Generator 对象上的 `return`。这将取消任务中的当前 Effect，并跳转至 finally 区块（若有定义的话）。

在 finally 区块中，你可以执行任何清理逻辑，或发起某些 action 来保持 store 处于一致状态(例如，当 ajax 请求被取消时，将 spinner 的状态重置为 false)。你可以在 finally 区块中检查 Saga 是不是通过 `yield cancelled()` 取消的。

取消信息会向下传播到子 saga。当取消任务时，middleware 还会取消当前 Effect（当前阻塞 task 的 Effect）。如果当前 Effect 调用了另一个 saga，那么该 saga 也会被取消。当取消 saga 时，所有 **附加分叉（attached forks）**（用 `yield fork()` 分叉出的 saga）将被取消。这意味着取消动作会有效地影响属于取消任务的整个执行树。

`cancel` 是一个非阻塞的 Effect。也就是说，执行 `cancel` 的 Saga 会在发起取消动作后立即恢复执行。

对于返回 Promise 结果的函数，你可以通过给 promise 附加一个 `[CANCEL]` 来插入自己的取消逻辑。

下述例子演示了如何将取消逻辑附加到 Promise 结果上:

```javascript
import { CANCEL } from 'redux-saga'
import { fork, cancel } from 'redux-saga/effects'

function myApi() {
  const promise = myXhr(...)

  promise[CANCEL] = () => myXhr.abort()
  return promise
}

function* mySaga() {

  const task = yield fork(myApi)

  // ... 过一会儿儿
  // 将会调用 myApi 上的 promise[CANCEL]
  yield cancel(task)
}
```

对于取消 jqXHR 对象，redux-saga 将自动地使用其 `abort` 方法。

### `cancel(...tasks)`

创建一个 Effect 描述信息，用来命令 middleware 取消之前的多个分叉任务。

- `tasks: Array<Task>` - [Task](#task) 是由之前 `fork` 指令返回的对象

#### 注意事项

它只是把任务数组包裹在 [cancel effects](#canceltask) 中，大致相当于 `yield tasks.map(t => cancel(t))`。

### `cancel()`

创建一个 Effect 描述信息，用来命令 middleware 取消 yield 它的任务（自取消）。

允许在 `finally` 区块中为外部取消（`cancel(task)`）和自取消（`cancel()`）复用析构类逻辑。

#### Example

```javascript
function* deleteRecord({ payload }) {
  try {
    const { confirm, deny } = yield call(prompt);
    if (confirm) {
      yield put(actions.deleteRecord.confirmed())
    }
    if (deny) {
      yield cancel()
    }
  } catch(e) {
    // 处理失败的情况
  } finally {
    if (yield cancelled()) {
      // 共享的取消逻辑
      yield put(actions.deleteRecord.cancel(payload))
    }
  }
}
```

### `select(selector, ...args)`

创建一个 Effect，用来命令 middleware 在当前 Store 的 state 上调用指定的选择器（即返回 `selector(getState(), ...args)` 的结果）。

- `selector: Function` - 一个 `(state, ...args) => args` 的函数。它接受当前 state 和一些可选参数，并返回当前 Store state 上的一部分数据。

- `args: Array<any>` - 传递给选择器的可选参数，将追加在 `getState` 后。

如果调用 `select` 的参数为空（即 `yield select()`），那么 effect 会取得完整的 state（与调用 `getState()` 的结果相同）。 

> 重要提醒：在向 store 发起 action 时，middleware 首先会把 action 转发给 reducers，然后通知 Sagas。这意味着，当你查询 Store 的 state 时，你获得的是 action 被应用 **后** 的 state。
> 但是，只有当所有后续中间件都以同步的形式调用 `next(action)` 时，才能保证此行为。如果有任何后续 middleware 异步地调用 `next(action)`（虽然不常见，但存在这种可能），那么 saga 会在 action 被应用 **前** 获得 state。因此，建议检查每一个后续的 middleware 的来源，以确保是通过同步的形式调用 `next(action)`；或者确保 redux-saga 是调用链中的最后一个中间件。

#### 注意事项

最好的话，Saga 应是自主独立的，并且不应依赖 Store 的 state。这使得我们在不影响 Saga 代码的情况下便可以轻松地修改 state 的实现。如果可能，saga 最好只依赖其自身内部控制的状态。但有的时候我们可能会发现在 Saga 中查询 state 比单独维护所属数据更方便（例如，当一个 Saga 重复调用某个 reducer，来计算那些已经被 Store 计算过的 state）。

例如，假设我们在应用程序中有这样结构的一份 state：

```javascript
state = {
  cart: {...}
}
```

我们创建一个 **选择器（selector）**，即一个知道如果从 State 中提取 `cart` 数据的函数： 

`./selectors`
```javascript
export const getCart = state => state.cart
```

然后，我们可以使用 `select` Effect 从 Saga 的内部使用该选择器：

`./sagas.js`
```javascript
import { take, fork, select } from 'redux-saga/effects'
import { getCart } from './selectors'

function* checkout() {
  // 使用被导出的选择器查询 state
  const cart = yield select(getCart)

  // ... 调用某些 API，然后发起一个 success/error action
}

export default function* rootSaga() {
  while (true) {
    yield take('CHECKOUT_REQUEST')
    yield fork(checkout)
  }
}
```

`checkout` 可以通过 `select(getCart)` 直接地获得所需的信息。Saga 仅与 `getCart` 选择器相耦合。如果我们有许多个需要访问 `cart` 数据的 Saga（或 React Component），那么它们将被耦合到统一的 `getCart` 函数上。并且如果我们改变了 state 的结构，我们只需要更新 `getCart` 即可。

### `actionChannel(pattern, [buffer])`

创建一个 Effect，用来命令 middleware 通过一个事件 channel 对匹配 `pattern` 的 action 进行排序。
作为可选项，你也可以提供一个 buffer 来控制如何缓存排序的 actions。

- `pattern:` - 请查看 `take(pattern)` 的 API
- `buffer: Buffer` - 一个 [Buffer](#buffer) 对象

#### 例子

以下代码创建了一个 channel，并用来缓存所有 `USER_REQUEST` action。请注意，即使是 Saga 也可能被 `call` effect 阻塞。所有在它被阻塞时进来的 action 都会被自动地缓存。这使得 Saga 每次只执行一次 API 调用。

```javascript
import { actionChannel, call } from 'redux-saga/effects'
import api from '...'

function* takeOneAtMost() {
  const chan = yield actionChannel('USER_REQUEST')
  while (true) {
    const {payload} = yield take(chan)
    yield call(api.getUser, payload)
  }
}
```

### `flush(channel)`

创建一个 Effect，用来命令 middleware 从 channel 中冲除所有被缓存的数据。被冲除的数据会返回至 saga，这样便可以在需要的时候再次被利用。

- `channel: Channel` - 一个 [`Channel`](#channel) 对象.

#### 例子

```javascript

function* saga() {
  const chan = yield actionChannel('ACTION')

  try {
    while (true) {
      const action = yield take(chan)
      // ...
    }
  } finally {
    const actions = yield flush(chan)
    // ...
  }

}
```

### `cancelled()`

创建一个 Effect，用来命令 middleware 返回该 generator 是否已经被取消。通常你会在 finally 区块中使用这个 Effect 来运行取消时专用的代码。

#### 例子

```javascript

function* saga() {
  try {
    // ...
  } finally {
    if (yield cancelled()) {
      // 只应在取消时执行的逻辑
    }
    // 应在所有情况下都执行的逻辑（例如关闭一个 channel）
  }
}
```

### `setContext(props)`

创建一个 effect，用来命令 middleware 更新其自身的上下文。这个 effect 扩展了 saga 的上下文，而不是代替。

### `getContext(prop)`

创建一个 effect，用来命令 middleware 返回 saga 的上下文中的一个特定属性。

## Effect 组合器（combinators）

### `race(effects)`

创建一个 Effect 描述信息，用来命令 middleware 在多个 Effect 间运行 *竞赛（Race）*（与 [`Promise.race([...])`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise/race) 的行为类似）。

`effects: Object` - 一个 {label: effect, ...} 形式的字典对象 

#### 例子

下面的例子在两个 effect 间运行了一次竞赛：

1. 对函数 `fetchUsers` 的一次 call，该函数将返回一个 Promise
2. 一个 `CANCEL_FETCH` action，该 action 可能最终在 Store 上被发起

```javascript
import { take, call, race } from `redux-saga/effects`
import fetchUsers from './path/to/fetchUsers'

function* fetchUsersSaga {
  const { response, cancel } = yield race({
    response: call(fetchUsers),
    cancel: take(CANCEL_FETCH)
  })
}
```

如果 `call(fetchUsers)` 先 resolve（或 reject），那么 `race` 的结果将是一个对象，该对象包含一个单键对象 `{response: result}`，其中 `result` 是 `fetchUsers` resolve 的结果。 

如果在 `fetchUsers` 完成之前，Store 上先发起了一个 `CANCEL_FETCH` 类型的 action，那么结果将是一个单键对象 `{cancel: action}`，其中 `action` 是被发起的 action。

#### 注意事项

当 resolve `race` 的时候，middleware 会自动地取消所有输掉的 Effect。

### `race([...effects]) (with Array)`

与 [`race(effects)`](#raceeffects) 相同，但传入的是 effect 的数组。

#### 例子

下面的例子在两个 effect 间运行了一次竞赛：


1. 对函数 `fetchUsers` 的一次 call，该函数将返回一个 Promise
2. 一个 `CANCEL_FETCH` action，该 action 可能最终在 Store 上被发起

```javascript
import { take, call, race } from `redux-saga/effects`
import fetchUsers from './path/to/fetchUsers'

function* fetchUsersSaga {
  const [response, cancel] = yield race([
    call(fetchUsers),
    take(CANCEL_FETCH)
  ])
}
```

如果 `call(fetchUsers)` 先 resolve（或 reject），那么 `response` 将是 `fetchUsers` 的结果，并且 `cancel` 将是 `undefined`。

如果在 `fetchUsers` 完成之前，Store 上先发起了一个 `CANCEL_FETCH` 类型的 action，那么 `response` 将是 `undefined`，并且 `cancel` 将是被发起的 action。

### `all([...effects]) - parallel effects`

创建一个 Effect 描述信息，用来命令 middleware 并行地运行多个 Effect，并等待它们全部完成。这是与标准的 [`Promise#all`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) 相当对应的 API。

#### 例子

以下的例子并行地运行了两个阻塞型调用：

```javascript
import { fetchCustomers, fetchProducts } from './path/to/api'
import { all, call } from `redux-saga/effects`

function* mySaga() {
  const [customers, products] = yield all([
    call(fetchCustomers),
    call(fetchProducts)
  ])
}
```

### `all(effects)`

与 [`all([...effects])`](#alleffects-parallel-effects) 相同，但就像 [`race(effects)`](#alleffects) 那样，传入的是一个带有 label 的 effect 的字典对象。

- `effects: Object` - 一个 {label: effect, ...} 形式的字典对象 

#### 例子

以下的例子并行地运行了两个阻塞型调用：

```javascript
import { fetchCustomers, fetchProducts } from './path/to/api'
import { all, call } from `redux-saga/effects`

function* mySaga() {
  const { customers, products } = yield all({
    customers: call(fetchCustomers),
    products: call(fetchProducts)
  })
}
```

#### 注意事项

当并发运行 Effect 时，middleware 将暂停 Generator，直到以下任一情况发生：

- 所有 Effect 都成功完成：返回一个包含所有 Effect 结果的数组，并恢复 Generator。

- 在所有 Effect 完成之前，有一个 Effect 被 reject：在 Generator 中抛出 reject 错误。

## 接口

### Task

Task 接口指定了通过 `fork`，`middleare.run` 或 `runSaga` 运行 Saga 的结果。

<table id="task-descriptor">
  <tr>
    <th>方法</th>
    <th>返回值</th>
  </tr>
  <tr>
    <td>task.isRunning()</td>
    <td>若任务还未返回或抛出了一个错误则为 true</td>
  </tr>
  <tr>
    <td>task.isCancelled()</td>
    <td>若任务已被取消则为 true</td>
  </tr>
  <tr>
    <td>task.result()</td>
    <td>任务的返回值。若任务仍在运行中则为 `undefined`</td>
  </tr>
  <tr>
    <td>task.error()</td>
    <td>任务抛出的错误。若任务仍在执行中则为 `undefined`</td>
  </tr>
  <tr>
    <td>task.done</td>
    <td>
      一个 Promise，以下二者之一：
        <ul>
          <li>以任务的返回值 resolve</li>
          <li>以任务抛出的错误 reject</li>
        </ul>
      </td>
  </tr>
  <tr>
    <td>task.cancel()</td>
    <td>取消任务（如果任务仍在执行中）</td>
  </tr>
</table>

### Channel

channel 是用于在任务间发送和接收消息的对象。在被感兴趣的接收者请求之前，来自发送者的消息将被放入（put）队列；在信息可用之前，已注册的接收者将被放入队列。

每个 channel 都有一个底层 buffer，这个 buffer 定义了缓存策略（fixed size、dropping、sliding）。

Channel 接口定义了 3 个方法：`take`，`put` 和 `close`

`Channel.take(callback):` 用于注册一个 taker。take 会根据以下规则解析

- 如果 channel 有被缓存的消息，那么将会从底层 buffer 用下一条消息调用 `callback`。
- 如果 channel 已关闭，并且没有被缓存的消息，那么将以 `END` 为参数调用 `callback`
- 否则，直到有消息被放入 channel 之前，`callback` 将被放入队列。

`Channel.put(message):` 用于在 buffer 上放入消息。将根据以下规则处理 put

- 如果 channel 已关闭，那么 put 将没有效果
- 如果还有未被处理的 taker，那么将用该 message 调用最老的 taker。
- 否则将 message 放入底层 buffer。

`Channel.flush(callback):` 用于从 channel 中提取所有被缓存的消息。flush 会根据以下规则解析

- 如果 channel 已关闭，并且没有被缓存的消息，那么将以 `END` 为参数调用 `callback`
- 否则，将以所有被缓存的消息为参数调用 `callback`

`Channel.close():` 关闭 channel，意味着不再允许做放入操作。所有未被处理的 taker 都将被以 `END` 为参数调用。

### Buffer

用于为 channel 实现缓存策略。Buffer 接口定义了 3 个方法：`isEmpty`，`put` 和 `take`

- `isEmpty()`: 如果缓存中没有消息则返回。每当注册了新的 taker 时，channel 都会调用该方法。
- `put(message)`: 用于往缓存中放入新的消息。请注意，缓存可以选择不存储消息。（例如，一个 dropping buffer 可以丢弃超过给定限制的任何新消息）
- `take()`：用于检索任何被缓存的消息。请注意，此方法的行为必须与 `isEmpty` 一致。

### SagaMonitor

用于由 middleware 发起监视（monitor）事件。实际上，middleware 发起 5 个事件：

- 当一个 effect 被触发时（通过 `yield someEffect`），middleware 调用 `sagaMonitor.effectTriggered`

- 如果该 effect 成功地被 resolve，则 middleware 调用 `sagaMonitor.effectResolved` 

- 如果该 effect 因一个错误被 reject，则 middleware 调用 `sagaMonitor.effectRejected`

- 如果该 effect 被取消，则 middleware 调用 `sagaMonitor.effectCancelled`

- 最后，当 Redux action 被发起时，middleware 调用 `sagaMonitor.actionDispatched` 

以下是每个方法的特点：

- `effectTriggered(options)` : options 是一个包含以下字段的对象

  - `effectId` : Number - 分配给 yielded effect 的唯一 ID

  - `parentEffectId` : Number - 父级 Effect 的 ID。在 `race` 或 `parallel` effect 的情况下，所有在内部 yield 的 effect 都将有一个直接 race/parallel 的父级 effect。在最顶级的 effect 的情况下，父级是包裹它的 Saga。

  - `label` : String - 在 `race` effect 的情况下，所有子 effect 都将被指定为传递给 `race` 的对象中对应键的标签。

  - `effect` : Object - yielded effect 其自身

- `effectResolved(effectId, result)`

    - `effectId` : Number - yielded effect 的 ID

    - `result` : any - 该 effect 成功 resolve 的结果。在 `fork` 或 `spawn` 的情况下，结果将是一个 `Task` 对象。

- `effectRejected(effectId, error)`

    - `effectId` : Number - yielded effect 的 ID

    - `error` : any - 该 effect reject 的错误

- `effectCancelled(effectId)`

    - `effectId` : Number - yielded effect 的 ID

- `actionDispatched(action)`

    - `action` : Object - 被发起的 Redux action。如果该 action 是由一个 Saga 发起的，那么该 action 将拥有一个属性 `SAGA_ACTION` 并被设为 true（你可以从 `redux-saga/utils` 中导入 `SAGA_ACTION`）。

## 外部 API
------------------------

### `runSaga(options, saga, ...args)`

允许在 Redux middleware 环境外部启动 saga。如果你想把 Saga 连接至外部的输入和输出（译注：即在外部执行 Saga），而不是至 store 的 action，这个 API 则会很有用。

`runSaga` 返回一个 Task 对象。与 `fork` effect 返回的对象一样。

- `options: Object` - 目前支持的选项是：

  - `subscribe(callback): Function` - 一个函数。它接受一个回调函数，并返回一个 `unsubscribe` 函数。

    - `callback(input): Function` - 用于订阅输入事件的回调函数（由 runSaga 提供）。`subscribe` 必须支持多个订阅。
      - `input: any` - 由 `subscribe` 传递给 `callback` 的参数（参考下方的注意事项）。

  - `dispatch(output): Function` - 用于实现 `put` effects。
    - `output: any` - 由 Saga 传递给 `put` effect 的参数（参考下方注意事项）。

  - `getState(): Function` - 用于实现 `select` 和 `getState` effects

  - `sagaMonitor` : [SagaMonitor](#sagamonitor) - 请查看 [`createSagaMiddleware(options)`](#createsagamiddlewareoptions) 的文档

  - `logger: Function` - 请查看 [`createSagaMiddleware(options)`](#createsagamiddlewareoptions) 的文档

  - `onError: Function` - 请查看 [`createSagaMiddleware(options)`](#createsagamiddlewareoptions) 的文档

- `saga: Function` - 一个 Generator 函数

- `args: Array<any>` - 传递给 `saga` 的参数

#### 注意事项

`{subscribe, dispatch}` 用于实现 `take` 和 `put` Effects。它定义了 Saga 的输入和输出接口。

`subscribe` 用于实现 `take(PATTERN)` effects。它会在每次有输入要发起时调用 `callback`（例如每次鼠标的点击，如果 Saga 连接到了 DOM 的点击事件的话）。
每次 `subscribe` 发射一个输入到它的 callback 时，如果 Saga 被 `take` Effect 阻塞了，并且 take pattern 和当前输入相匹配，那么 Saga 将以那个输入恢复。

`dispatch` 用于履行 `put` Effect。每次 Saga 发射一个 `yield put(output)` 时，将以 `output` 调用 `dispatch`。

## 工具

### `channel([buffer])`

一个可以用于创建 Channel 的工厂方法。你可以选择给它传递一个 buffer 参数，从而控制该 channel 如何缓存消息。

默认情况下，如果没有提供 buffer 参数，那么直到感兴趣的 taker 被注册之前，channel 都将会把将传入的消息放入队列中，最多存放十条。默认的缓存会用一个 FIFO 的策略派发消息：一个新的 taker 将收到缓存中最老的消息。

### `eventChannel(subscribe, [buffer], [matcher])`

使用 `subscribe` 方法创建 channel，该 channel 将订阅一个事件源。直到感兴趣的 taker 被注册之前，从事件源传入的事件都将在 channel 中排队。

- `subscribe: Function` 用于订阅底层事件源。这个函数必定返回一个用于结束订阅的 unsubscribe 函数。

- `buffer: Buffer` 可选的 Buffer 对象，用于在该 channel 上缓存消息。如果不传该参数，消息将不会被缓存在该 channel 上。

- `matcher: Function` 可选的断言函数（`any => Boolean`），用于过滤传入的消息。只有被该 matcher 接受的消息才会被放到 channel 上。

若要通知 channel 事件源已结束，你可以使用 `END` 通知传入的 subscriber 。

#### 例子

在下面的例子中，我们创建了一个 event channel，它将订阅一个 `setInterval`。
In the following example we create an event channel that will subscribe to a `setInterval`

```javascript
const countdown = (secs) => {
  return eventChannel(emitter => {
      const iv = setInterval(() => {
        console.log('countdown', secs)
        secs -= 1
        if (secs > 0) {
          emitter(secs)
        } else {
          emitter(END)
          clearInterval(iv)
          console.log('countdown terminated')
        }
      }, 1000);
      return () => {
        clearInterval(iv)
        console.log('countdown cancelled')
      }
    }
  )
}
```

### `buffers`

提供一些通用的缓存

- `buffers.none()`: 不缓存。如果没有尚未处理的 taker，那么新消息将被丢失。

- `buffers.fixed(limit)`: 新消息将被缓存，最多缓存 `limit` 条。溢出时将会报错。如果不填 `limit` 的值，那么 `limit` 将为 `10`。

- `buffers.expanding(initialSize)`: 与 `fixed` 类似，但溢出时将会使缓存动态扩展。

- `buffers.dropping(limit)`: 与 `fixed` 类似，但溢出时将会静默地丢弃消息。

- `buffers.sliding(limit)`: 与 `fixed` 类似，但溢出时将会把新消息插到缓存的最尾处，并丢弃缓存中最老的消息。

### `delay(ms, [val])`

返回一个 effect 描述符，用于阻塞执行 `ms` 毫秒，并返回 `val` 值。

### `cloneableGenerator(generatorFunc)`

接受一个 generator 函数（function*），并返回一个 generator 函数。
从该函数实例化的所有 generator 都是可克隆的。
仅用于测试。

#### 例子

当你想要测试一个 saga 中的不同分支，而又不需要重放引向它的 actions 时，这会很有用：

```javascript

function* oddOrEven() {
  // some stuff are done here
  yield 1;
  yield 2;
  yield 3;

  const userInput = yield 'enter a number';
  if (userInput % 2 === 0) {
    yield 'even';
  } else {
    yield 'odd'
  }
}

test('my oddOrEven saga', assert => {
  const data = {};
  data.gen = cloneableGenerator(oddOrEven)();

  assert.equal(
    data.gen.next().value,
    1,
    'it should yield 1'
  );

  assert.equal(
    data.gen.next().value,
    2,
    'it should yield 2'
  );

  assert.equal(
    data.gen.next().value,
    3,
    'it should yield 3'
  );

  assert.equal(
    data.gen.next().value,
    'enter a number',
    'it should ask for a number'
  );

  assert.test('even number is given', a => {
    // 我们在给出 number 前生成该 generator 的一个克隆。
    data.clone = data.gen.clone();

    a.equal(
      data.gen.next(2).value,
      'even',
      'it should yield "event"'
    );

    a.equal(
      data.gen.next().done,
      true,
      'it should be done'
    );

    a.end();
  });

  assert.test('odd number is given', a => {

    a.equal(
      data.clone.next(1).value,
      'odd',
      'it should yield "odd"'
    );

    a.equal(
      data.clone.next().done,
      true,
      'it should be done'
    );

    a.end();
  });

  assert.end();
});

```
### `createMockTask()`

返回一个对象，用于 mock 一个 task。
仅用于测试。
[有关更多信息，请查看 Task Cancellation 的文档。](/docs/advanced/TaskCancellation.md#testing-generators-with-fork-effect)

## 速查表

### 阻塞 / 非阻塞

|         名称         |                       阻塞                        |
| -------------------- | ------------------------------------------------- |
| takeEvery            | 否                                                |
| takeLatest           | 否                                                |
| takeLeading          | 否                                                |
| throttle             | 否                                                |
| take                 | 是                                                |
| take(channel)        | 有时 (请查看 API 参考)                            |
| take.maybe           | 是                                                |
| put                  | 否                                                |
| put.resolve          | 是                                                |
| put(channel, action) | 否                                                |
| call                 | 是                                                |
| apply                | 是                                                |
| cps                  | 是                                                |
| fork                 | 否                                                |
| spawn                | 否                                                |
| join                 | 是                                                |
| cancel               | 否                                                |
| select               | 否                                                |
| actionChannel        | 否                                                |
| flush                | 是                                                |
| cancelled            | 是                                                |
| race                 | 是                                                |
| delay                | 是                                                |
| all                  | 当 array 或 object 中有阻塞型 effect 的时候阻塞。 |
