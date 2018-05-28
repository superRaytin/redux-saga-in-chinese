# [redux-saga 中文文档](https://github.com/superRaytin/redux-saga-in-chinese)

**文档版本号：1.0.0-beta**

> 在线 Gitbook 地址：http://superRaytin.github.io/redux-saga-in-chinese
>
> 英文原版：https://redux-saga.js.org/

`redux-saga` 是一个用于管理应用程序 Side Effect（副作用，例如异步获取数据，访问浏览器缓存等）的 library，它的目标是让副作用管理更容易，执行更高效，测试更简单，在处理故障时更容易。

可以想像为，一个 saga 就像是应用程序中一个单独的线程，它独自负责处理副作用。
`redux-saga` 是一个 redux 中间件，意味着这个线程可以通过正常的 redux action 从主应用程序启动，暂停和取消，它能访问完整的 redux state，也可以 dispatch redux action。

redux-saga 使用了 ES6 的 Generator 功能，让异步的流程更易于读取，写入和测试。*（如果你还不熟悉的话，[这里有一些介绍性的链接](https://redux-saga.js.org/docs/ExternalResources.html)）* 通过这样的方式，这些异步的流程看起来就像是标准同步的 Javascript 代码。（有点像 `async`/`await`，但 Generator 还有一些更棒而且我们也需要的功能）。

你可能已经用了 `redux-thunk` 来处理数据的读取。不同于 redux thunk，你不会再遇到回调地狱了，你可以很容易地测试异步流程并保持你的 action 是干净的。

# 开始

## 安装

```sh
$ npm install --save redux-saga
```

或

```sh
$ yarn add redux-saga
```

或者，你可以直接在 HTML 页面的 `<script>` 标签中使用提供的 UMD 构建文件。参见 [本节](#using-umd-build-in-the-browser)。

## 使用示例

假设我们有一个 UI 界面，在单击按钮时从远程服务器获取一些用户数据（为简单起见，我们只列出 action 触发代码）。

```javascript
class UserComponent extends React.Component {
  ...
  onSomeButtonClicked() {
    const { userId, dispatch } = this.props
    dispatch({type: 'USER_FETCH_REQUESTED', payload: {userId}})
  }
  ...
}
```

这个组件 dispatch 一个 plain Object 的 action 到 Store。我们将创建一个 Saga 来监听所有的 `USER_FETCH_REQUESTED` action，并触发一个 API 调用获取用户数据。

#### `sagas.js`

```javascript
import { call, put, takeEvery, takeLatest } from 'redux-saga/effects'
import Api from '...'

// worker Saga : 将在 USER_FETCH_REQUESTED action 被 dispatch 时调用
function* fetchUser(action) {
   try {
      const user = yield call(Api.fetchUser, action.payload.userId);
      yield put({type: "USER_FETCH_SUCCEEDED", user: user});
   } catch (e) {
      yield put({type: "USER_FETCH_FAILED", message: e.message});
   }
}

/*
  在每个 `USER_FETCH_REQUESTED` action 被 dispatch 时调用 fetchUser
  允许并发（译注：即同时处理多个相同的 action）
*/
function* mySaga() {
  yield takeEvery("USER_FETCH_REQUESTED", fetchUser);
}

/*
  也可以使用 takeLatest

  不允许并发，dispatch 一个 `USER_FETCH_REQUESTED` action 时，
  如果在这之前已经有一个 `USER_FETCH_REQUESTED` action 在处理中，
  那么处理中的 action 会被取消，只会执行当前的
*/
function* mySaga() {
  yield takeLatest("USER_FETCH_REQUESTED", fetchUser);
}

export default mySaga;
```

为了能跑起 Saga，我们需要使用 `redux-saga` 中间件将 Saga 与 Redux Store 建立连接。

#### `main.js`

```javascript
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

import reducer from './reducers'
import mySaga from './sagas'

// create the saga middleware
const sagaMiddleware = createSagaMiddleware()
// mount it on the Store
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)

// then run the saga
sagaMiddleware.run(mySaga)

// render the application
```

# 文档

- [介绍](http://superRaytin.github.io/redux-saga-in-chinese/docs/introduction/index.html)
- [基本概念](http://superRaytin.github.io/redux-saga-in-chinese/docs/basics/index.html)
- [高级概念](http://superRaytin.github.io/redux-saga-in-chinese/docs/advanced/index.html)
- [技巧](http://superRaytin.github.io/redux-saga-in-chinese/docs/recipes/index.html)
- [外部资源](http://superRaytin.github.io/redux-saga-in-chinese/docs/ExternalResources.html)
- [问题解答](http://superRaytin.github.io/redux-saga-in-chinese/docs/Troubleshooting.html)
- [名词解释](http://superRaytin.github.io/redux-saga-in-chinese/docs/Glossary.html)
- [API 参考](http://superRaytin.github.io/redux-saga-in-chinese/docs/api/index.html)

# 在浏览器中使用 umd 构建版本

在 `dist/` 文件夹有一个可用的 **umd** `redux-saga` 构建文件。`redux-saga` 以 `ReduxSaga` 挂载在全局 window 对象中。这能让你在创建 Saga 中间件时不需要使用 ES6 的 `import` 语法：

```javascript
var sagaMiddleware = ReduxSaga.default()
```

umd 版本在你不使用 Webpack 或 Browserify 时相当有用。你可以从 [unpkg](https://unpkg.com/) 直接读取。

以下是可用的构建好的文件：

- [https://unpkg.com/redux-saga/dist/redux-saga.js](https://unpkg.com/redux-saga/dist/redux-saga.js)
- [https://unpkg.com/redux-saga/dist/redux-saga.min.js](https://unpkg.com/redux-saga/dist/redux-saga.min.js)

The runtime must be imported before **redux-saga**:
**重要！** 如果你的目标浏览器不支持 *ES2015 generators*，那么你必须使用转换器来编译它们（比如 [babel plugin](https://github.com/facebook/regenerator/tree/master/packages/regenerator-transform)) 并提供一个有效的 runtime，比如 [这个](https://unpkg.com/regenerator-runtime/runtime.js)。
这个 runtime 必须在 **redux-saga** 之前引入：

```javascript
import 'regenerator-runtime/runtime'
// then
import sagaMiddleware from 'redux-saga'
```

# 从资源构建示例

```sh
$ git clone https://github.com/redux-saga/redux-saga.git
$ cd redux-saga
$ npm install
$ npm test
```

以下的例子是从 Redux 仓库移植过来的（截至目前）。

### 计数器示例

有 3 个计数器例子。

#### counter-vanilla

这个例子使用了 vanilla Javascript 和 UMD 构建版本。所有资源都在 `index.html` 中引入。

在浏览器中打开 `index.html` 运行这个例子。

> 重要：你的浏览器必须支持 Generator。最新版本的 Chrome/Firefox/Edge 已经支持。

#### counter

这个例子使用了 webpack 和高阶 API `takeEvery`。

```sh
$ npm run counter

# test sample for the generator
$ npm run test-counter
```

#### cancellable-counter

这个例子使用了低阶 API 来演示任务取消的场景。

```sh
$ npm run cancellable-counter
```

### 购物车示例

```sh
$ npm run shop

# test sample for the generator
$ npm run test-shop
```

### 异步示例

```sh
$ npm run async

# test sample for the generators
$ npm run test-async
```

### 真实项目示例（使用 webpack 的热重载）

```sh
$ npm run real-world

# sorry, no tests yet
```

### TypeScript

Redux-Saga 与 TypeScript 配合使用需要 `DOM.Iterable` 或 `ES2015.Iterable`。如果你的 `target` 是 `ES6`，则不需要再设置，然而如果是 `ES5`，你将需要自己把它们加进来。
检查你的 `tsconfig.json` 文件和官方的 <a href="https://www.typescriptlang.org/docs/handbook/compiler-options.html">compiler options</a> 文档。

### Logo

你可以在 [logo 目录](https://github.com/redux-saga/redux-saga/tree/master/logo) 中找到不同风格的 Redux-Saga 官方 logo。

### 贡献者

> 定期更新

- [Leon Shi@superRaytin](https://github.com/superRaytin)
- [Kevin He@kevinxh](https://github.com/kevinxh)

**如果看到翻译不准确、句子不通顺的地方，欢迎随时指出。本文档翻译流程按照 [ETC 翻译规范](https://github.com/react-guide/ETC)，欢迎你来一起完善。**

