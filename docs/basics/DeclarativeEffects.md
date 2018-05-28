# 声明式 Effects

在 `redux-saga` 的世界里，Sagas 都用 Generator 函数实现。我们从 Generator 里 yield 纯 JavaScript 对象以表达 Saga 逻辑。
我们称呼那些对象为 *Effect*。Effect 是一个简单的对象，这个对象包含了一些给 middleware 解释执行的信息。
你可以把 Effect 看作是发送给 middleware 的指令以执行某些操作（调用某些异步函数，发起一个 action 到 store，等等）。

你可以使用 `redux-saga/effects` 包里提供的函数来创建 Effect。

这一部分和接下来的部分，我们将介绍一些基础的 Effect。并见识到这些 Effect 概念是如何让 Sagas 很容易地被测试的。

Sagas 可以多种形式 yield Effect。最简单的方式是 yield 一个 Promise。

举个例子，假设我们有一个监听 `PRODUCTS_REQUESTED` action 的 Saga。每次匹配到 action，它会启动一个从服务器上获取产品列表的任务。

```javascript
import { takeEvery } from 'redux-saga/effects'
import Api from './path/to/api'

function* watchFetchProducts() {
  yield takeEvery('PRODUCTS_REQUESTED', fetchProducts)
}

function* fetchProducts() {
  const products = yield Api.fetch('/products')
  console.log(products)
}
```

在上面的例子中，我们在 Generator 中直接调用了 `Api.fetch`（在 Generator 函数中，`yield` 右边的任何表达式都会被求值，结果会被 yield 给调用者）。

`Api.fetch('/products')` 触发了一个 AJAX 请求并返回一个 Promise，Promise 会 resolve 请求的响应，
这个 AJAX 请求将立即执行。看起来简单又地道，但...

假设我们想测试上面的 generator：

```javascript
const iterator = fetchProducts()
assert.deepEqual(iterator.next().value, ??) // 我们期望得到什么？
```

我们想要检查 generator yield 的结果的第一个值。在我们的情况里，这个值是执行 `Api.fetch('/products')` 这个 Promise 的结果。
在测试过程中，执行真正的服务（real service）是一个既不可行也不实用的方法，所以我们必须 *模拟（mock）* `Api.fetch` 函数。
也就是说，我们需要将真实的函数替换为一个假的，这个假的函数并不会真的发送 AJAX 请求而只会检查是否用正确的参数调用了 `Api.fetch`（在我们的情况里，正确的参数是 `'/products'`）。

模拟使测试更加困难和不可靠。另一方面，那些只简单地返回值的函数更加容易测试，因此我们可以使用简单的 `equal()` 来检查结果。
这是编写最可靠测试用例的方法。

不相信？我建议你阅读 [Eric Elliott's article](https://medium.com/javascript-scene/what-every-unit-test-needs-f6cd34d9836d#.4ttnnzpgc):

> (...)`equal()`, by nature answers the two most important questions every unit test must answer,
but most don’t:
- What is the actual output?
- What is the expected output?
>
> If you finish a test without answering those two questions, you don’t have a real unit test. You have a sloppy, half-baked test.

实际上我们需要的只是保证 `fetchProducts` 任务 yield 一个调用正确的函数，并且函数有着正确的参数。

相比于在 Generator 中直接调用异步函数，**我们可以仅仅 yield 一条描述函数调用的信息**。也就是说，我们将简单地 yield 一个看起来像下面这样的对象：

```javascript
// Effect -> 调用 Api.fetch 函数并传递 `./products` 作为参数
{
  CALL: {
    fn: Api.fetch,
    args: ['./products']  
  }
}
```

换句话说，Generator 将会 yield 包含 *指令* 的文本对象（plain Objects），`redux-saga` middleware 将确保执行这些指令并将指令的结果回馈给 Generator。
这样的话，在测试 Generator 时，所有我们需要做的就是，将 yield 后的对象作一个简单的 `deepEqual` 来检查它是否 yield 了我们期望的指令。

出于这样的原因，`redux-saga` 提供了一个不一样的方式来执行异步调用。

```javascript
import { call } from 'redux-saga/effects'

function* fetchProducts() {
  const products = yield call(Api.fetch, '/products')
  // ...
}
```

我们使用了 `call(fn, ...args)` 这个函数。**与前面的例子不同的是，现在我们不立即执行异步调用，相反，`call` 创建了一条描述结果的信息**。
就像在 Redux 里你使用 action 创建器，创建一个将被 Store 执行的、描述 action 的纯文本对象。
`call` 创建一个纯文本对象描述函数调用。`redux-saga` middleware 确保执行函数调用并在响应被 resolve 时恢复 generator。

这让你能容易地测试 Generator，就算它在 Redux 环境之外。因为 `call` 只是一个返回纯文本对象的函数而已。

```javascript
import { call } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)
```

现在我们不需要模拟任何东西了，一个简单的相等测试就足够了。

这些 *声明式调用（declarative calls）* 的优势是，我们可以通过简单地遍历 Generator 并在 yield 后的成功的值上面做一个 `deepEqual` 测试，
就能测试 Saga 中所有的逻辑。这是一个真正的好处，因为复杂的异步操作都不再是黑盒，你可以详细地测试操作逻辑，不管它有多么复杂。

`call` 同样支持调用对象方法，你可以使用以下形式，为调用的函数提供一个 `this` 上下文：

```javascript
yield call([obj, obj.method], arg1, arg2, ...) // 如同 obj.method(arg1, arg2 ...)
```

`apply` 提供了另外一种调用的方式：

```javascript
yield apply(obj, obj.method, [arg1, arg2, ...])
```

`call` 和 `apply` 非常适合返回 Promise 结果的函数。另外一个函数 `cps` 可以用来处理 Node 风格的函数
（例如，`fn(...args, callback)` 中的 `callback` 是 `(error, result) => ()` 这样的形式，`cps` 表示的是延续传递风格（Continuation Passing Style））。

举个例子：

```javascript
import { cps } from 'redux-saga'

const content = yield cps(readFile, '/path/to/file')
```

当然你也可以像测试 call 一样测试它：

```javascript
import { cps } from 'redux-saga/effects'

const iterator = fetchSaga()
assert.deepEqual(iterator.next().value, cps(readFile, '/path/to/file') )
```

`cps` 同 `call` 的方法调用形式是一样的。

完整列表的 declarative effects 可在这里找到： [API reference](https://redux-saga.js.org/docs/api/#effect-creators)