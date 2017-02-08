# 初级教程

## 本教程的目标

本教程尝试用一种易于接受的方式（希望如此）来介绍 redux-saga。

我们将使用 Redux 仓库那个很小的计数器例子作为我们的入门教程。
这个应用比较简单，但是非常适合用来演示说明 redux-saga 的基本概念，不至于迷失在过多的细节里。

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


## 你好，Sagas！

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

const sagaMiddleware = createSagaMiddleware()
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)
sagaMiddleware.run(helloSaga)

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

注意，与 redux-thunk 不同，上面组件发起的是一个普通对象格式的 action。

现在我们将介绍另一种执行异步调用的 Saga。我们的用例如下：

> 在每个 `INCREMENT_ASYNC` action 发起后，我们需要启动一个做以下事情的任务：

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

// Our watcher Saga: 在每个 INCREMENT_ASYNC action 调用后，派生一个新的 incrementAsync 任务
export function* watchIncrementAsync() {
  yield* takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

好吧，该解释一下了。首先我们创建一个工具函数 `delay`，用于返回一个延迟 1 秒再 resolve 的 Promise。
我们将使用这个函数去 *阻塞* Generator。

Sagas 被实现为 Generator 函数，它 yield 对象到 redux-saga middleware。
被 yield 的对象都是一类指令，指令可被 middleware 解释执行。当 middleware 取得一个 yield 后的 Promise，middleware 会暂停 Saga，直到 Promise 完成。
在上面的例子中，`incrementAsync` 这个 Saga 会暂停直到 `delay` 返回的 Promise 被 resolve，这个 Promise 将在 1 秒后 resolve。

一旦 Promise 被 resolve，middleware 会恢复 Saga 去执行下一个语句（更准确地说是执行下面所有的语句，直到下一个 yield）。
在我们的情况里，下一个语句是另一个 yield 后的对象：调用 `put({type: 'INCREMENT'})` 的结果。
意思是 Saga 指示 middleware 发起一个 `INCREMENT` 的 action。

`put` 就是我们所说的一个调用 *Effect* 的例子。Effect 是一些简单 Javascript 对象，对象包含了要被 middleware 执行的指令。
当 middleware 拿到一个被 Saga yield 后的 Effect，它会暂停 Saga，直到 Effect 执行完成，然后 Saga 会再次被恢复。

总结一下，`incrementAsync` Saga 通过 `delay(1000)` 延迟了 1 秒钟，然后发起了一个 `INCREMENT` 的 action。

接下来，我们创建了另一个 Saga `watchIncrementAsync`。这个 Saga 将监听所有发起的 `INCREMENT_ASYNC` action，并在每次 action 被匹配时派生一个新的 `incrementAsync` 任务。
为了实现这个目的，我们使用一个辅助函数 `takeEvery` 来执行以上的处理过程。

在我们开始这个应用之前，我们需要将 `watchIncrementAsync` 这个 Saga 连接至 Store：

```javascript

//...
import { helloSaga, watchIncrementAsync } from './sagas'

const store = createStore(
  reducer,
  applyMiddleware(createSagaMiddleware(helloSaga, watchIncrementAsync))
)

//...
```

注意我们不需要连接 `incrementAsync` 这个 Saga，因为它会在每次 `INCREMENT_ASYNC` action 发起时被 `watchIncrementAsync` 动态启动。


## 让我们的代码可测试

我们希望测试 `incrementAsync` Saga，以此保证它执行期望的任务。

创建另一个文件 `saga.spec.js`：

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  // now what ?
});
```

由于 `incrementAsync` 是一个 Generator 函数，当我们在 middleware 之外运行它，每次调用 generator 的 `next`，你将得到一个以下结构的对象：

```javascript
gen.next() // => { done: boolean, value: any }
```

`value` 字段包含 yield 后的表达式，即 `yield` 后面那个表达式的结果。`done` 字段指示 generator 是结束了，还是有更多的 `yield` 表达式。

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

问题是我们如何测试 `delay` 的返回值？我们不能在 Promise 之间做简单的相等测试。如果 `delay` 返回的是一个 *普通（normal）* 的值，
事情将会变得很简单。

好吧，`redux-saga` 提供了一种方式，让上面的语句变得可能。与在 `incrementAsync` 中直接调用 `delay(1000)` 不同，我们叫它 *间接（indirectly*


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

我们现在做的是 `yield call(delay, 1000)` 而不是 `yield delay(1000)`，所以有何不同？

在 `yield delay(1000)` 的情况下，yield 后的表达式 `delay(1000)` 在被传递给 `next` 的调用者之前就被执行了（当运行我们的代码时，调用者可能是 middleware。
也有可能是运行 Generator 函数并对返回的 Generator 进行迭代的测试代码）。所以调用者得到的是一个 Promise，像在以上的测试代码里一样。

在 `yield call(delay, 1000)` 的情况下，yield 后的表达式 `call(delay, 1000)` 被传递给 `next` 的调用者。`call` 就像 `put`，
返回一个指示 middleware 以给定参数调用给定的函数的 Effect。

```javascript
put({type: 'INCREMENT'}) // => { PUT: {type: 'INCREMENT'} }
call(delay, 1000)        // => { CALL: {fn: delay, args: [1000]}}
```

这里发生的情况是：middleware 检查每个 yield Effect 的类型，然后决定如何实现那个 Effect。如果 Effect 类型是 `PUT` 那 middleware 会发起一个 action 到 Store。
如果 Effect 类型是 `CALL` 那么它会调用给定的函数。

这种把 Effect 创建和 Effect 执行之间分开的做法，使得我们以一种令人惊讶的简单方法去测试 Generator 成为可能。

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

由于 `put` 和 `call` 返回文本对象，所以我们可以在测试代码中重复使用同样的函数。为了测试 `incrementAsync` 的逻辑，
我们可以简单地遍历 generator 并对它的值做 `deepEqual` 测试。

为了运行上面的测试代码，我们需要输入：

```
npm test
```

测试结果会显示在控制面板上。
