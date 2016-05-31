## 技巧

### 节流

你可以通过在监听的 Saga 里调用一个 delay 函数，针对一系列发起的 action 进行节流。
举个例子，假设用户在文本框输入文字的时候，UI 触发了一个 `INPUT_CHANGED` action：

```javascript

const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms))

function* handleInput(input) {
  ...
}

function* watchInput() {
  while(true) {
    const { input } = yield take('INPUT_CHANGED')
    yield fork(handleInput, input)
    // 节流 500ms
    yield call(delay, 500)
  }
}
```

通过延迟 `fork`，`watchInput` 将会被阻塞 500ms，发生在此期间内的所有 `INPUT_CHANGED` action 都会被忽略。
这保证了 Saga 在每 500ms 内只触发一次 `INPUT_CHANGED` action。

但上面的代码还有一个小问题。`take` 一个 action 后，`watchInput` 将睡眠 500ms，这意味着它会忽略发生在这期间内的所有 action。
这也许是节流的目的，但要注意的是 watcher 也将错过那个尾部 action（trailer action）：即最后的那个 action 也许正好发生在这 500ms 内。
如果你是对文本框的输入操作进行节流，这可能是不可取的，因为，如果最后的输入操作发生在这 500ms 内，你可能会希望响应最后的那个输入，即使节流限制已经过去了。

下面是一个更详细的版本，这个版本将会跟踪记录尾部操作：

```javascript
function* watchInput(wait) {
  let lastAction
  let lastTime = Date.now()
  let countDown = 0 // 因为要处理第一个 action

  while(true) {
    const winner = yield race({
      action: take('INPUT_CHANGED'),
      timeout: countDown ? call(delay, countDown) : null
    })
    const now = Date.now()
    countDown -= (now - lastTime)
    lastTime = now

    if(winner.action) {
      lastAction = action
    }
    if(lastAction && countDown <= 0) {
      yield fork(worker, lastAction)
      lastAction = null
      countDown = wait
    }
  }
}
```

在新版本中，我们维护了一个 `countDown` 变量，用于跟踪剩余的超时时间。
最初的 `countDown` 是 `0`，因为我们要处理第一个 action。处理完第一个 action 之后，countDown 将被设置为节流周期 `wait`，
意思是在处理下一个 action 之前，至少需要等待 `wait`ms。

然后，在每一次迭代中，我们在下一个尾部 action 与剩余超时时间之间启动一个 `race`。现在我们不会错过任何一个 action 了，
相反，我们使用 `lastAction` 变量来跟踪记录最后的那个 action，并且将 countDown 更新为剩余超时时间。

`if(lastAction && countDown <= 0) {...}` 区块确保即使节流周期过期了（即 `countDown` 小于或等于 0），我们仍然可以处理最后的尾部操作（即 `lastAction` 不是 null/undefined）。
处理完 action 后，我们立即重置 `lastAction` 和 `countDown`。这样就可以等待另一个 `wait`ms 周期和处理另一个 action。

### 防抖动

为了对 action 队列进行防抖动，可以在被 `fork` 的任务里放置一个 `delay`。

```javascript

const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms))

function* handleInput(input) {
  // 500ms 防抖动
  yield call(delay, 500)
  ...
}

function* watchInput() {
  let task
  while(true) {
    const { input } = yield take('INPUT_CHANGED')
    if(task)
      yield cancel(task)
    task = yield fork(handleInput, input)
  }
}
```

在上面的示例中，`handleInput` 在执行之前等待了 500ms。如果用户在此期间输入了更多文字，我们将收到更多的 `INPUT_CHANGED` action。
并且由于 `handleInput` 仍然会被 `delay` 阻塞，所以在执行自己的逻辑之前它会被 `watchInput` 取消。
