# 取消任务

我们已经在 [无阻塞调用](http://redux-saga-in-chinese.js.org/docs/advanced/NonBlockingCalls.html) 一节中看到了取消任务的示例。
在这节，我们将回顾一下，在一些更加详细的情况下取消的语义。

一旦任务被 fork，可以使用 `yield cancel(task)` 来中止任务执行。取消正在运行的任务。

来看看它是如何工作的，让我们先考虑一个简单的例子：一个可通过某些 UI 命令启动或停止的后台同步任务。
在接收到 `START_BACKGROUND_SYNC` action 后，我们 fork 一个后台任务，周期性地从远程服务器同步一些数据。

这个任务将会一直执行直到一个 `STOP_BACKGROUND_SYNC` action 被触发。
然后我们取消后台任务，等待下一个 `START_BACKGROUND_SYNC` action。

```javascript
import { take, put, call, fork, cancel, cancelled, delay } from 'redux-saga/effects'
import { someApi, actions } from 'somewhere'

function* bgSync() {
  try {
    while (true) {
      yield put(actions.requestStart())
      const result = yield call(someApi)
      yield put(actions.requestSuccess(result))
      yield delay(5000)
    }
  } finally {
    if (yield cancelled())
      yield put(actions.requestFailure('Sync cancelled!'))
  }
}

function* main() {
  while ( yield take(START_BACKGROUND_SYNC) ) {
    // 启动后台任务
    const bgSyncTask = yield fork(bgSync)

    // 等待用户的停止操作
    yield take(STOP_BACKGROUND_SYNC)
    // 用户点击了停止，取消后台任务
    // 这会导致被 fork 的 bgSync 任务跳进它的 finally 区块
    yield cancel(bgSyncTask)
  }
}
```

在上面的示例中，取消 `bgSyncTask` 将会导致 Generator 跳进 finally 区块。可使用 `yield cancelled()` 来检查 Generator 是否已经被取消。

取消正在执行的任务，也将同时取消被阻塞在当前 Effect 中的任务。

举个例子，假设在应用程序生命周期的某个时刻，还有挂起的（未完成的）调用链：

```javascript
function* main() {
  const task = yield fork(subtask)
  ...
  // later
  yield cancel(task)
}

function* subtask() {
  ...
  yield call(subtask2) // currently blocked on this call
  ...
}

function* subtask2() {
  ...
  yield call(someApi) // currently blocked on this call
  ...
}
```

`yield cancel(task)` 触发了 `subtask` 任务的取消，反过来它将触发 `subtask2` 的取消。

现在我们看到取消不断的往下传播（相反的，被回传的值和没有捕捉的错误不断往上）。
你可以把它看作是 caller（调用异步的操作）和 callee（被调用的操作）之间的对照。callee 是负责执行操作。如果它完成了（不管是成功或失败）结果将会往上到它的 caller，最终到 caller 的调用方。就是这样，callee 是负责完成流程。

现在如果 callee 一直处于等待，而且 caller 决定取消操作，它将触发一种信号往下传播到 callee（以及通过 callee 本身被调用的任何深层操作）。所有深层等待的操作都将被取消。

取消传播还将引发：如果加入的 task 被取消的话，task 的 joiner（那些被阻塞的 yield join(task)）也将会被取消。同样的，任何那些 joiner 潜在的 caller 都将会被取消（因为他们阻塞的操作已经从外面被取消）。

## 用 fork effect 测试 generators

当 `fork` 被调用时，它会在后台启动 task 并返回 task 对象 —— 就像我们之前所学习的。测试的时候，我们需要 `createMockTask` 这个 utility function。
在 fork test 之后，这个 function 返回的 Object 将会被传送到下一个 next 调用。例如 Mock task 可以被传送到 `cancel`。这是一个页面 `main` generator 的测试用例。

```javascript
import { createMockTask } from 'redux-saga/utils';

describe('main', () => {
  const generator = main();

  it('waits for start action', () => {
    const expectedYield = take(START_BACKGROUND_SYNC);
    expect(generator.next().value).to.deep.equal(expectedYield);
  });

  it('forks the service', () => {
    const expectedYield = fork(bgSync);
    const mockedAction = { type: 'START_BACKGROUND_SYNC' };
    expect(generator.next(mockedAction).value).to.deep.equal(expectedYield);
  });

  it('waits for stop action and then cancels the service', () => {
    const mockTask = createMockTask();

    const expectedTakeYield = take(STOP_BACKGROUND_SYNC);
    expect(generator.next(mockTask).value).to.deep.equal(expectedTakeYield);

    const expectedCancelYield = cancel(mockTask);
    expect(generator.next().value).to.deep.equal(expectedCancelYield);
  });
});
```

你也可使用 mock task 的 `setRunning`、`setResult` 和 `setError` function 来设定 mock task 的状态。例如 `mockTask.setRunning(false)`。

### 注意

记住重要的一点，`yield cancel(task)` 不会等待被取消的任务完成（即执行其 catch 区块）。`cancel` Effect 的行为和 `fork` 有点类似。
一旦取消发起，它就会尽快返回。一旦取消，任务通常应尽快完成它的清理逻辑然后返回。

## 自动取消

除了手动取消任务，还有一些情况的取消是自动触发的。

1. 在 `race` Effect 中。所有参与 race 的任务，除了优胜者（译注：最先完成的任务），其他任务都会被取消。

2. 并行的 Effect (`yield [...]`)。一旦其中任何一个任务被拒绝，并行的 Effect 将会被拒绝（受 `Promise.all` 启发）。在这种情况中，所有其他的 Effect 将被自动取消。
