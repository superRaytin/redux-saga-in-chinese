# 名词解释

这是 Redux Saga 核心的术语词汇表。

### Effect

一个 effect 就是一个 Plain Object JavaScript 对象，包含一些将被 saga middleware 执行的指令。

使用 redux-saga 提供的工厂函数来创建 effect。
举个例子，你可以使用 `call(myfunc, 'arg1', 'arg2')` 指示 middleware 调用 `myfunc('arg1', 'arg2')` 并将结果返回给 yield effect 的那个 Generator。

### Task

一个 task 就像是一个在后台运行的进程。在基于 redux-saga 的应用程序中，可以同时运行多个 task。通过 `fork` 函数来创建 task：

```javascript
function* saga() {
  ...
  const task = yield fork(otherSaga, ...args)
  ...
}
```

### 阻塞调用/非阻塞调用

阻塞调用的意思是，Saga 在 yield Effect 之后会等待其执行结果返回，结果返回后才会恢复执行 Generator 中的下一个指令。

非阻塞调用的意思是，Saga 会在 yield Effect 之后立即恢复执行。

示例：

```javascript
function* saga() {
  yield take(ACTION)              // 阻塞: 将等待 action
  yield call(ApiFn, ...args)      // 阻塞: 将等待 ApiFn (如果 ApiFn 返回一个 Promise 的话)
  yield call(otherSaga, ...args)  // 阻塞: 将等待 otherSaga 结束

  yied put(...)                   // 阻塞: 将同步发起 action (使用 Promise.then)

  const task = yield fork(otherSaga, ...args)  // 非阻塞: 将不会等待 otherSaga
  yield cancel(task)                           // 非阻塞: 将立即恢复执行
  // or
  yield join(task)                             // 阻塞: 将等待 task 结束
}
```

### Watcher/Worker

指的是一种使用两个单独的 Saga 来组织控制流的方式。

- Watcher: 监听发起的 action 并在每次接收到 action 时 `fork` 一个 worker。

- Worker: 处理 action 并结束它。

示例：

```javascript
function* watcher() {
  while(true) {
    const action = yield take(ACTION)
    yield fork(worker, action.payload)
  }
}

function* worker(payload) {
  // ... do some stuff
}
```
