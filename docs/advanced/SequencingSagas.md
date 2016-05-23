# 通过 `yield*` 对 Sagas 进行排序

你可以使用内置的 `yield*` 操作符来组合多个 Sagas，使得它们保持顺序。
这让你可以一种简单的程序风格来排列你的 *宏观任务（macro-tasks）*。

```javascript
function* playLevelOne(getState) { ... }

function* playLevelTwo(getState) { ... }

function* playLevelThree(getState) { ... }

function* game(getState) {

  const score1 = yield* playLevelOne(getState)
  put(showScore(score1))

  const score2 = yield* playLevelTwo(getState)
  put(showScore(score2))

  const score3 = yield* playLevelThree(getState)
  put(showScore(score3))

}
```

注意，使用 `yield*` 将导致该 Javascript 运行环境 *漫延* 至整个序列。
由此产生的迭代器（来自 `game()`）将 yield 所有来自于嵌套迭代器里的值。一个更强大的替代方案是使用更通用的中间件组合机制。
