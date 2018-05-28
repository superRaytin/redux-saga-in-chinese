# 新手教程

## 本教程的目标

本教程尝试用一种容易上手的方式（希望如此）来介绍 redux-saga。

我们将使用 Redux 仓库那个很小的计数器例子作为我们的入门教程。
这个应用程序比较简单，但是非常适合用来演示说明 redux-saga 的基本概念，不至于迷失在过多的细节里。

### 初始步骤

在我们开始前，clone 这个 [教程仓库](https://github.com/redux-saga/redux-saga-beginner-tutorial)。

> 教程的最终代码放在 `sagas` 分支上

然后在命令行输入：

```sh
$ cd redux-saga-beginner-tutorial
$ npm install
```

接着启动应用：

```sh
$ npm start
```

我们先从最简单的用例开始：2 个按钮 `增加（Increment）` 和 `减少（Decrement）` 计数。之后我们将介绍异步调用。

不出意外的话，你应该能看到 2 个按钮 `Increment` 和 `Decrement`，以及按钮下方 `Counter : 0` 的文字。

> 如果你在运行这个应用程序的时候遇到问题，可随时在这个[教程仓库](https://github.com/redux-saga/redux-saga-beginner-tutorial/issues)上创建 issue。


## Hello，Sagas！

接下来将创建我们的第一个 Saga。按照传统，我们将编写一个 Sagas 版本的 'Hello, world'。

创建一个 `sagas.js` 的文件，然后添加以下代码片段：

```javascript
export function* helloSaga() {
  console.log('Hello Sagas!');
}
```

所以并没有什么吓人的东西，只是一个很普通的功能（好吧，除了 `*`）。这段代码的唯一的作用就是打印一句问候的消息到控制台。

为了运行我们的 Saga，我们需要：

- 创建一个 Saga middleware 和要运行的 Sagas（目前我们只有一个 `helloSaga`）
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

运行 `helloSaga` 之前，我们必须使用 `applyMiddleware` 将 middleware 连接至 Store。然后使用 `sagaMiddleware.run(helloSaga)` 运行 Saga。

到目前为止，我们的 Saga 并没做什么特别的事情。它只是打印了一条消息，然后退出。


## 发起异步调用

现在我们来添加一些更接近原始计数器例子的东西。为了演示异步调用，我们将添加另外一个按钮，用于在点击 1 秒后增加计数。

首先，我们需要在 UI 组件上添加一个额外的按钮和一个回调 `onIncrementAsync`。

```javascript
const Counter = ({ value, onIncrement, onDecrement, onIncrementAsync }) =>
  <div>
    <button onClick={onIncrementAsync}>
      Increment after 1 second
    </button>
    {' '}
    <button onClick={onIncrement}>
      Increment
    </button>
    {' '}
    <button onClick={onDecrement}>
      Decrement
    </button>
    <hr />
    <div>
      Clicked: {value} times
    </div>
  </div>
```

接下来我们需要将组件的 `onIncrementAsync` 与 Store action 连接起来。

修改 `main.js` 模块：

```javascript
function render() {
  ReactDOM.render(
    <Counter
      value={store.getState()}
      onIncrement={() => action('INCREMENT')}
      onDecrement={() => action('DECREMENT')}
      onIncrementAsync={() => action('INCREMENT_ASYNC')} />,
    document.getElementById('root')
  )
}
```

注意，与 redux-thunk 不同，上面组件 dispatch 的是一个 plain Object 的 action。

现在我们将介绍另一种执行异步调用的 Saga。我们的用例如下：

> 我们需要在每个 `INCREMENT_ASYNC` action 启动一个做以下事情的任务：

>- 等待 1 秒，然后增加计数

添加以下代码到 `sagas.js` 模块：

```javascript
import { delay } from 'redux-saga'
import { put, takeEvery } from 'redux-saga/effects'

// ...

// Our worker Saga: 将执行异步的 increment 任务
export function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

// Our watcher Saga: 在每个 INCREMENT_ASYNC action spawn 一个新的 incrementAsync 任务
export function* watchIncrementAsync() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

好吧，该解释一下了。

我们引入了一个工具函数 `delay`，这个函数返回一个延迟 1 秒再 resolve 的 [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
我们将使用这个函数去 *block(阻塞)* Generator。

Sagas 被实现为 [Generator functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*)，它会 yield 对象到 redux-saga middleware。
被 yield 的对象都是一类指令，指令可被 middleware 解释执行。当 middleware 取得一个 yield 后的 Promise，middleware 会暂停 Saga，直到 Promise 完成。
在上面的例子中，`incrementAsync` 这个 Saga 会暂停直到 `delay` 返回的 Promise 被 resolve，这个 Promise 将在 1 秒后 resolve。

一旦 Promise 被 resolve，middleware 会恢复 Saga 接着执行，直到遇到下一个 yield。
在这个例子中，下一个语句是另一个被 yield 的对象：调用 `put({type: 'INCREMENT'})` 的结果，意思是告诉 middleware 发起一个 `INCREMENT` 的 action。

`put` 就是我们称作 *Effect* 的一个例子。Effects 是一些简单 Javascript 对象，包含了要被 middleware 执行的指令。
当 middleware 拿到一个被 Saga yield 的 Effect，它会暂停 Saga，直到 Effect 执行完成，然后 Saga 会再次被恢复。

总结一下，`incrementAsync` Saga 通过 `delay(1000)` 延迟了 1 秒钟，然后 dispatch 一个叫 `INCREMENT` 的 action。

接下来，我们创建了另一个 Saga `watchIncrementAsync`。我们用了一个 `redux-saga` 提供的辅助函数 `takeEvery`，用于监听所有的 `INCREMENT_ASYNC` action，并在 action 被匹配时执行 `incrementAsync` 任务。

现在我们有了 2 个 Sagas，我们需要同时启动它们。为了做到这一点，我们将添加一个 `rootSaga`，负责启动其他的 Sagas。在同样的 `sagas.js` 文件中，重构文件如下：

```javascript
import { delay } from 'redux-saga'
import { put, takeEvery, all } from 'redux-saga/effects'


function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}


function* watchIncrementAsync() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
}


// notice how we now only export the rootSaga
// single entry point to start all Sagas at once
export default function* rootSaga() {
  yield all([
    helloSaga(),
    watchIncrementAsync()
  ])
}
```

这个 Saga yield 了一个数组，值是调用 `helloSaga` 和 `watchIncrementAsync` 两个 Saga 的结果。意思是说这两个 Generators 将会同时启动。
现在我们只有在 `main.js` 的 root Saga 中调用 `sagaMiddleware.run`。

```javascript
// ...
import rootSaga from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = ...
sagaMiddleware.run(rootSaga)

// ...
```

## 让我们的代码可测试

我们希望测试 `incrementAsync` Saga，以确保它执行期望的任务。

创建另一个文件 `saga.spec.js`：

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  // now what ?
});
```

`incrementAsync` 是一个 Generator 函数。执行的时候返回一个 iterator object，这个 iterator 的 `next` 方法返回一个如下格式的对象：

```javascript
gen.next() // => { done: boolean, value: any }
```

`value` 字段包含被 yield 后的表达式，也就是 `yield` 后面那个表达式的结果。`done` 字段指示 generator 是结束了，还是有更多的 `yield` 表达式。

在 `incrementAsync` 的例子中，generator 连续 yield 了两个值：

1. `yield delay(1000)`
2. `yield put({type: 'INCREMENT'})`


所以，如果我们连续 3 次调用 generator 的 next 方法，我们会得到以下结果：

```javascript
gen.next() // => { done: false, value: <result of calling delay(1000)> }
gen.next() // => { done: false, value: <result of calling put({type: 'INCREMENT'})> }
gen.next() // => { done: true, value: undefined }
```

前两次调用返回了 yield 表达式的结果。第三次调用由于没有更多的 yield 了，所以 `done` 字段被设置为 true。
并且由于 `incrementAsync` Generator 未返回任何东西（没有 `return` 语句），所以 `value` 字段被设置为 `undefined`。

所以现在，为了测试 `incrementAsync` 里面的逻辑，我们需要对返回的 Generator 进行简单地迭代并检查 Generator yield 后的值。

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next(),
    { done: false, value: ??? },
    'incrementAsync should return a Promise that will resolve after 1 second'
  )
});
```

上面的问题是我们如何测试 `delay` 的返回值？我们不能在 Promise 之间做简单的相等测试。如果 `delay` 返回的是一个 *普通（normal）* 的值，
事情将会变得很简单。

好吧，`redux-saga` 提供了一种方式，让上面的语句变得可能。与在 `incrementAsync` 中直接（directly）调用 `delay(1000)` 不同，我们叫它 *indirectly*：

```javascript
import { delay } from 'redux-saga'
import { put, takeEvery, all, call } from 'redux-saga/effects'

// ...

export function* incrementAsync() {
  // use the call Effect
  yield call(delay, 1000)
  yield put({ type: 'INCREMENT' })
}
```

我们现在做的是 `yield call(delay, 1000)` 而不是 `yield delay(1000)`，所以有何不同？

在 `yield delay(1000)` 的情况下，yield 后的表达式 `delay(1000)` 在被传递给 `next` 的调用者之前就被执行了（当运行我们的代码时，调用者可能是 middleware。
也有可能是运行 Generator 函数并对返回的 Generator 进行迭代的测试代码）。所以调用者得到的是一个 Promise，像在以上的测试代码里一样。

而在 `yield call(delay, 1000)` 的情况下，yield 后的表达式 `call(delay, 1000)` 被传递给 `next` 的调用者。`call` 就像 `put`，
返回一个 Effect，告诉 middleware 使用给定的参数调用给定的函数。实际上，无论是 `put` 还是 `call` 都不执行任何 dispatch 或异步调用，它们只是简单地返回 plain Javascript 对象。

```javascript
put({type: 'INCREMENT'}) // => { PUT: {type: 'INCREMENT'} }
call(delay, 1000)        // => { CALL: {fn: delay, args: [1000]}}
```

这里发生的事情是：middleware 检查每个被 yield 的 Effect 的类型，然后决定如何实现哪个 Effect。如果 Effect 类型是 `PUT` 那 middleware 会 dispatch 一个 action 到 Store。
如果 Effect 类型是 `CALL` 那么它会调用给定的函数。

这种把 Effect 创建和 Effect 执行之间分开的做法，使得我们能够以一种非常简单的方法去测试 Generator。

```javascript
import test from 'tape';

import { put, call } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { incrementAsync } from './sagas'

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

由于 `put` 和 `call` 返回 plain Object，所以我们可以在测试代码中重复使用同样的函数。为了测试 `incrementAsync` 的逻辑，
我们可以简单地遍历 generator 并对它的值做 `deepEqual` 测试。

为了运行上面的测试代码，我们需要运行：

```sh
$ npm test
```

测试结果会显示在控制面板上。
