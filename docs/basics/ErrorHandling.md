# 错误处理

在这一节中，我们将看到如何在前面的例子中处理故障案例。我们假设远程读取因为某些原因失败了，API 函数 `Api.fetch` 返回一个被拒绝（rejected）的 Promise。

我们希望通过在 Saga 中发起 `PRODUCTS_REQUEST_FAILED` action 到 Store 来处理那些错误。

我们可以使用熟悉的 `try/catch` 语法在 Saga 中捕获错误。

```javascript
import Api from './path/to/api'
import { call, put } from 'redux-saga/effects'

//...

function* fetchProducts() {
  try {
    const products = yield call(Api.fetch, '/products')
    yield put({ type: 'PRODUCTS_RECEIVED', products })
  }
  catch(error) {
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
  }
}
```

为了测试故障案例，我们将使用 Generator 的 `throw` 方法。

```javascript
import { call, put } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// 期望一个 call 指令
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)

// 创建一个模拟的 error 对象
const error = {}

// 期望一个 dispatch 指令
assert.deepEqual(
  iterator.throw(error).value,
  put({ type: 'PRODUCTS_REQUEST_FAILED', error }),
  "fetchProducts should yield an Effect put({ type: 'PRODUCTS_REQUEST_FAILED', error })"
)
```

在这个案例中，我们传递一个模拟的 error 对象给 `throw`，这会引发 Generator 中断当前的执行流并执行捕获区块（catch block）。

当然了，你并不一定得在 `try`/`catch` 区块中处理错误，你也可以让你的 API 服务返回一个正常的含有错误标识的值。例如，
你可以捕捉 Promise 的拒绝操作，并将它们映射到一个错误字段对象。

```javascript
import Api from './path/to/api'
import { take, put } from 'redux-saga/effects'

function fetchProductsApi() {
  return Api.fetch('/products')
    .then(response => {response})
    .catch(error => {error})
}

function* fetchProducts() {
  const { response, error } = yield call(fetchProductsApi)
  if(response)
    yield put({ type: 'PRODUCTS_RECEIVED', products: response })
  else
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
}
```
