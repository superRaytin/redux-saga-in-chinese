# 组合 Sagas

使用 `yield*` 为组合 Sagas 提供了一种通畅的方式，但这个方法也有一些局限性：

- 你可能会想要单独测试嵌套的 Generator。这导致了一些重复的测试代码及重复执行的开销。
我们不希望执行一个嵌套的 Generator，而仅仅是想确认它是被传入正确的参数来调用。

- 更重要的是，`yield*` 只允许任务的顺序组合，所以一次你只能 yield* 一个 Generator。

你可以直接使用 `yield` 来并行地启动一个或多个子任务。当 yield 一个 `call` 至 Generator，Saga 将等待 Generator 处理结束，
然后以返回的值恢复执行（或错误从子任务中传播过来，则抛出异常）。

```javascript
function* fetchPosts() {
  yield put( actions.requestPosts() )
  const products = yield call(fetchApi, '/products')
  yield put( actions.receivePosts(products) )
}

function* watchFetch() {
  while ( yield take(FETCH_POSTS) ) {
    yield call(fetchPosts) // waits for the fetchPosts task to terminate
  }
}
```

yield 一个队列的嵌套的 Generators，将同时启动这些子 Generators（sub-generators），并等待它们完成。
然后以所有返回的结果恢复执行：

```javascript
function* mainSaga(getState) {
  const results = yield [ call(task1), call(task2), ...]
  yield put( showResults(results) )
}
```

事实上，yield Sagas 并不比 yield 其他 effects（未来的 actions，timeouts，等等）不同。
这意味着你可以使用 effect 合并器将那些 Sagas 和所有其他类型的 Effect 合并。

例如，你可能希望用户在有限的时间内完成一些游戏：

```javascript
function* game(getState) {

  let finished
  while(!finished) {
    // 必须在 60 秒内完成
    const {score, timeout}  = yield race({
      score  : call( play, getState),
      timeout : call(delay, 60000)
    })

    if(!timeout) {
      finished = true
      yield put( showScore(score) )
    }
  }

}
```
