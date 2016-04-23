# 发起 action 到 store

在前面的例子上更进一步，假设每次保存之后，我们想发起一些 action 通知 Store 数据获取成功了（目前我们先忽略失败的情况）。

我们可以找出一些方法来传递 Store 的 `dispatch` 函数到 Generator。然后 Generator 可以在接收到获取的响应之后调用它。

```javascript
//...

function* fetchProducts(dispatch)
  const products = yield call(Api.fetch, '/products')
  dispatch({ type: 'PRODUCTS_RECEIVED', products })
}
```

然而，该解决方案与我们在上一节中看到的从 Generator 内部直接调用函数，有着相同的缺点。如果我们想要测试 `fetchProducts` 接收到 AJAX 响应之后执行 dispatch，
我们还需要模拟 `dispatch` 函数。

相反，我们需要同样的声明式的解决方案。只需创建一个对象来指示 middleware 我们需要发起一些 action，然后让 middleware 执行真实的 dispatch。
这种方式我们就可以同样的方式测试 Generator 的 dispatch：只需检查 yield 后的 Effect，并确保它包含正确的指令。

redux-saga 为此提供了另外一个函数 `put`，这个函数用于创建 dispatch Effect。

```javascript
import { call, put } from 'redux-saga/effects'
//...

function* fetchProducts() {
  const products = yield call(Api.fetch, '/products')
  // 创建并 yield 一个 dispatch Effect
  yield put({ type: 'PRODUCTS_RECEIVED', products })
}
```

现在，我们可以像上一节那样轻易地测试 Generator：

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

// 创建一个假的响应对象
const products = {}

// expects a dispatch instruction
assert.deepEqual(
  iterator.next(products).value,
  put({ type: 'PRODUCTS_RECEIVED', products }),
  "fetchProducts should yield an Effect put({ type: 'PRODUCTS_RECEIVED', products })"
)
```

现在我们通过 Generator 的 `next` 方法来将假的响应对象传递到 Generator。在 middleware 环境之外，
我们可完全控制 Generator，通过简单地模拟结果并还原 Generator，我们可以模拟一个真实的环境。
相比于去模拟函数和窥探调用（spying calls），模拟数据要简单的多。