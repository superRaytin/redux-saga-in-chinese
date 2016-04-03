# 初级教程

## 本教程的目标

本教程尝试用一种易于接受的方式（希望如此）来介绍 redux-saga。

我们将使用 Redux 仓库那个很小的计数器例子作为我们的入门教程。
这个应用比较简单，但是非常适合用来演示说明 redux-saga 的基本概念，不至于在迷失在过多的细节里。

### 初始步骤

在我们开始前，需要先 clone 这个仓库：

https://github.com/yelouafi/redux-saga-beginner-tutorial

> 此教程最终的代码位于 sagas 分支

然后在命令行输入：

```
cd redux-saga-beginner-tutorial
npm install
```

接着启动应用：

```
npm start
```

我们先从最简单的用例开始：2 个按钮 `增加（Increment）` 和 `减少（Decrement）` 计数。之后我们将介绍异步调用。

不出意外的话，你应该能看到 2 个按钮 `Increment` 和 `Decrement`，以及按钮下方 `Counter : 0` 的文字。

> 如果你在运行这个应用的时候遇到问题，可随时在这个教程的仓库上创建 issue

>https://github.com/yelouafi/redux-saga-beginner-tutorial/issues


## 雷猴，Sagas！

接下来将创建我们的第一个 Saga。按照传统，我们将编写一个 Sagas 版本的 'Hello, world'。

创建一个 `sagas.js` 的文件，然后添加以下代码片段：

```javascript
export function* helloSaga() {
  console.log('Hello Sagas!');
}
```

所以并没有什么吓人的东西，只是一个很普通的功能（好吧，除了 `*`）。这段代码的作用是打印一句问候消息到控制台。

为了运行我们的 Saga，我们需要：

- 以 Sagas 列表创建一个 Saga middleware（目前我们只有一个 `helloSaga`）
- 将这个 Saga middleware 连接至 Redux store.

我们修改一下 `main.js`：

```javascript
// ...
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

//...
import { helloSaga } from './sagas'

const store = createStore(
  reducer,
  applyMiddleware(createSagaMiddleware(helloSaga))
)

// rest unchanged
```

首先我们引入 `./sagas` 模块中的 Saga。然后使用 `redux-saga` 模块的 `createSagaMiddleware` 工厂函数来创建一个 Saga middleware。
`createSagaMiddleware` 接受 Sagas 列表，这些 Sagas 将会通过创建的 middleware 被立即执行。


到目前为止，我们的 Saga 并没做什么特别的事情。它只是打印了一条消息，然后退出。


## 发起异步调用

现在我们来添加一些更接近原始计数器例子的东西。为了演示异步调用，我们将添加另外一个按钮，用于点击后 1 秒增加计数。

首先，我们需要提供一个额外的回调 `onIncrementAsync`。

```javascript
const Counter = ({ value, onIncrement, onDecrement, onIncrementAsync }) =>
  <div>
    ...
    {' '}
    <button onClick={onIncrementAsync}>Increment after 1 second</button>
    <hr />
    <div>Clicked: {value} times</div>
  </div>
```

接下来我们需要使 `onIncrementAsync` 与 Store action 联系起来。

修改 `main.js` 模块：

```javascript
function render() {
  ReactDOM.render(
    <Counter
      ...
      onIncrementAsync={() => action('INCREMENT_ASYNC')}
    />,
    document.getElementById('root')
  )
}
```

注意，与 redux-thunk 不同，上面组件发起的 action 是普通对象格式的。

现在我们将介绍另一种执行异步调用的 Saga。我们的用例如下：

> 在每个 `INCREMENT_ASYNC` action 发起后，我们想要启动一个做以下事情的任务：

>- 等待 1 秒，然后增加计数


添加以下代码到 `sagas.js` 模块：

```javascript
import { takeEvery } from 'redux-saga'
import { put } from 'redux-saga/effects'

// 一个工具函数：返回一个 Promise，这个 Promise 将在 1 秒后 resolve
export const delay = ms => new Promise(resolve => setTimeout(resolve, ms))

// Our worker Saga: 将异步执行 increment 任务
export function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

// Our watcher Saga: 在每个 INCREMENT_ASYNC action 调用后，派发一个新的 incrementAsync 任务
export function* watchIncrementAsync() {
  yield* takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

好吧，该解释一下了。首先我们创建一个工具函数 `delay`，用于返回一个延迟 1 秒 resolve 的 Promise。
我们将使用这个函数去 *阻塞* Generator。

Sagas, which are implemented as Generator functions, yield objects to the
redux-saga middleware. The yielded objects are a kind of instructions to be interpreted by
the middleware. When the middleware retrieves a yielded Promise, it'll suspend the Saga until
the Promise completes. In the above example, the `incrementAsync` Saga will be suspended until
the Promise returned by `delay` resolves, which will happen after 1 second.

Once the Promise is resolved, the middleware will resume the Saga to execute the next statement
(more accurately to execute all the following statements until the next yield). In our case, the
next statement is another yielded object: which is the result of calling `put({type: 'INCREMENT'})`.
It means the Saga instructs the middleware to dispatch an `INCREMENT` action.

`put` is one example of what we call an *Effect*. Effects are simple JavaScript Objects which
contain instructions to be fulfilled by the middleware. When a middleware retreives an Effect
yielded by a Saga, it pauses the Saga until the Effect is fullfilled, then the Saga is resumed
again.

So to summarize, the `incrementAsync` Saga sleeps for 1 second via the call to `delay(1000)`, then
dispatches an `INCREMENT` action.

Next, we created another Saga `watchIncrementAsync`. The Saga will watch the dispatched `INCREMENT_ASYNC`
actions and spawn a new `incrementAsync` task on each action. For this purpose, we use a helper function
provided by the library `takeEvery` which will perform the process above.

Before we start the application, we need to connect the `watchIncrementAsync` Saga to the Store

```javascript

//...
import { helloSaga, watchIncrementAsync } from './sagas'

const store = createStore(
  reducer,
  applyMiddleware(createSagaMiddleware(helloSaga, watchIncrementAsync))
)

//...
```

Note we don't need to connect the `incrementAsync` Saga, because it'll be started dynamically
by `watchIncrementAsync` on each `INCREMENT_ASYNC` action.


## Making our code testable

We want to test our `incrementAsync` Saga to make sure it performs the desired task.

Create another file `saga.spec.js`

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  // now what ?
});
```

Since `incrementAsync` is a Generator function, when we run it outside the middleware,
Each time you invoke `next` on the generator, you get an object of the following shape

```javascript
gen.next() // => { done: boolean, value: any }
```

The `value` field contains the yielded expression, i.e. the result of the expression after
the `yield`. The `done` field indicates if the generator has terminated or if there are still
more 'yield' expressions.

In the case of `incrementAsync`, the generator yields 2 values consecutively:

1. `yield delay(1000)`
2. `yield put({type: 'INCREMENT'})`


So if we invoke the next method of the generator 3 times consecutively we get the following
results:

```javascript
gen.next() // => { done: false, value: <result of calling delay(1000)> }
gen.next() // => { done: false, value: <result of calling put({type: 'INCREMENT'})> }
gen.next() // => { done: true, value: undefined }
```

The first 2 invocations return the results of the yield expressions. On the 3rd invocation
since there is no more yield the `done` field is set to true. And since the `incrementAsync`
Generator doesn't return anything (no `return` statement), the `value` field is set to
`undefined`.

So now, in order to test the logic inside `incrementAsync`, we'll simply have to iterate
over the returned Generator and check the values yielded by the generator.

```javascript
import test from 'tape';

import { incrementAsync } from '../src/sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next().value,
    { done: false, value: ??? },
    'incrementAsync should return a Promise that will resolve after 1 second'
  )
});
```

The issue is how do we test the return value of `delay`? We can't do a simple equality test
on Promises. If `delay` returned a *normal* value, things would've been be easier to test.

Well, `redux-saga` provides a way which makes the above statement possible. Instead of calling
`delay(1000)` directly inside `incrementAsync`, we'll call it *indirectly*


```javascript
//...
import { put, call } from 'redux-saga/effects'

export const delay = ms => new Promise(resolve => setTimeout(resolve, ms))

export function* incrementAsync() {
  // use the call Effect
  yield call(delay, 1000)
  yield put({ type: 'INCREMENT' })
}
```

Instead of doing `yield delay(1000)`, we're now doing `yield call(delay, 1000)` so what's the difference ?

In the first case, the yield expression `delay(1000)` is evaluated before it gets passed to the caller of `next`
(the caller could be the middleware when running our code. It could also be our test code which runs the Generator
function and iterates over the returned Generator). So what the caller gets is a Promise, like in the test code
above.

In the second case, the yield expression `call(delay, 1000)` is what gets passed to the caller of `next`. `call`
just like `put`, returns an Effect which instructs the middleware to call a given function with the given arguments.
In fact, neither `put` nor `call` performs any dispatch or asynchronous call by themselves, they simply return
plain JavaScript objects.

```javascript
put({type: 'INCREMENT'}) // => { PUT: {type: 'INCREMENT'} }
call(delay, 1000)        // => { CALL: {fn: delay, args: [1000]}}
```

What happens is that the middleware examines the type of each yielded Effect then decides how
to fulfill that Effect. If the Effect type is a `PUT` then it'll dispatch an action to the Store.
If the Effect is a `CALL` then it'll call the given function.

This separation between Effect creation and Effect execution makes possible to test our Generator
in a surprisingly easy way

```javascript
import test from 'tape';

import { put, call } from 'redux-saga/effects'
import { incrementAsync, delay } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next().value,
    call(delay, 1000),
    'incrementAsync Saga must call delay(1000)'
  )

  assert.deepEqual(
    gen.next().value,
    put({type: 'INCREMENT'}),
    'incrementAsync Saga must dispatch an INCREMENT action'
  )

  assert.deepEqual(
    gen.next(),
    { done: true, value: undefined },
    'incrementAsync Saga must be done'
  )

  assert.end()
});
```

Since `put` and `call` return plain objects, we can reuse the same functions in our test
code. And to test the logic of `incrementAsync`, we simply iterate over the generator
and do `deepEqual` tests on its values.

In order to run the above test, type
```
npm test
```

which should report the results on the console
