## 技巧

### 节流（Throttling）

你可以通过在监听的 Saga 里调用一个 delay 函数，针对一系列发起的 action 进行节流。
举个例子，假设用户在文本框输入文字的时候，UI 触发了一个 `INPUT_CHANGED` action：

```javascript
import { throttle } from 'redux-saga/effects'

function* handleInput(input) {
  // ...
}

function* watchInput() {
  yield throttle(500, 'INPUT_CHANGED', handleInput)
}
```

通过使用 `throttle` helper，`watchInput` 不会在 500ms 启动一个新的 `handleInput` task，但在相同时间內，它仍然接受最新的 `INPUT_CHANGED` 到底层的 `buffer`，所以它会忽略所有的 `INPUT_CHANGED` action。
这确保了 Saga 在 500ms 这段时间，最多接受一个 `INPUT_CHANGED` action，并且可以继续处理 trailing action。

### 防抖动（Debouncing）

为了对 action 队列进行防抖动，可以在被 `fork` 的任务里放一个内置的 `delay`。

```javascript

import { call, cancel, fork, take, delay } from 'redux-saga/effects'

function* handleInput(input) {
  // 500ms 防抖动
  yield delay(500)
  ...
}

function* watchInput() {
  let task
  while (true) {
    const { input } = yield take('INPUT_CHANGED')
    if (task) {
      yield cancel(task)
    }
    task = yield fork(handleInput, input)
  }
}
```

在上面的示例中，`handleInput` 在执行之前等待了 500ms。如果用户在此期间输入了更多文字，我们将收到更多的 `INPUT_CHANGED` action。
并且由于 `handleInput` 仍然会被 `delay` 阻塞，所以在执行自己的逻辑之前它会被 `watchInput` 取消。

上面的例子可以使用 redux-saga 的 `takeLatest` helper 重写：

```javascript

import { call, takeLatest, delay } from 'redux-saga/effects'

function* handleInput({ input }) {
  // debounce by 500ms
  yield delay(500)
  ...
}

function* watchInput() {
  // 将取消当前执行的 handleInput task
  yield takeLatest('INPUT_CHANGED', handleInput);
}
```

## ajax 重试（Retrying XHR calls）

为了重试指定次数的 XHR 调用，使用一个 for 循环和一个 delay：

```javascript

import { call, put, take, delay } from 'redux-saga/effects'

function* updateApi(data) {
  for(let i = 0; i < 5; i++) {
    try {
      const apiResponse = yield call(apiRequest, { data });
      return apiResponse;
    } catch(err) {
      if(i < 4) {
        yield delay(2000);
      }
    }
  }
  // 重试 5 次后失败
  throw new Error('API request failed');
}

export default function* updateResource() {
  while (true) {
    const { data } = yield take('UPDATE_START');
    try {
      const apiResponse = yield call(updateApi, data);
      yield put({
        type: 'UPDATE_SUCCESS',
        payload: apiResponse.body,
      });
    } catch (error) {
      yield put({
        type: 'UPDATE_ERROR',
        error
      });
    }
  }
}

```

在上面的例子中，`apiRequest` 将重试 5 次，每次延迟 2 秒。在第 5 次失败后，将会通过父级 saga 抛出一个异常，这将会 dispatch `UPDATE_ERROR` action。

如果你想要无限重试，可以把 `for` 循环替换成 `while (true)`。你也可以使用 `takeLatest` 来替代 `take`，这样就只会重试最后一次的请求。在错误处理中加入一个 `UPDATE_RETRY` action，我们就可以通知使用者更新没有成功，但是它会重试。

```javascript
import { delay } from 'redux-saga/effects'

function* updateApi(data) {
  while (true) {
    try {
      const apiResponse = yield call(apiRequest, { data });
      return apiResponse;
    } catch(error) {
      yield put({
        type: 'UPDATE_RETRY',
        error
      })
      yield delay(2000);
    }
  }
}

function* updateResource({ data }) {
  const apiResponse = yield call(updateApi, data);
  yield put({
    type: 'UPDATE_SUCCESS',
    payload: apiResponse.body,
  });
}

export function* watchUpdateResource() {
  yield takeLatest('UPDATE_START', updateResource);
}

```

## 撤销（Undo）

Undo 通过允许 action 顺利进行来尊重使用者，在假设使用者不知道他们在做什么之前。[GoodUI](https://goodui.org/#8)
[redux 文档](http://redux.js.org/docs/recipes/ImplementingUndoHistory.html) 描述了一个
稳定的方式来实现一个基于修改 reducer 包含 `past`、`present`、`future` state 的 undo。
甚至有一个 [redux-undo](https://github.com/omnidan/redux-undo) library，它建立了一个高阶 reducer 来帮助开发者做那些繁重的工作。

然而，这个方法附带了一些开销，因为它的 store 引用了应用之前的 state（s）。

使用 redux-saga 的 `delay` 和 `race` 我们可以实现一个简单的、一次性的 undo，不需要 enhance 我们的 reducer 或 store 先前的 state。

```javascript
import { take, put, call, spawn, race, delay } from 'redux-saga/effects'
import { updateThreadApi, actions } from 'somewhere'

function* onArchive(action) {

  const { threadId } = action
  const undoId = `UNDO_ARCHIVE_${threadId}`

  const thread = { id: threadId, archived: true }

  // show undo UI element, and provide a key to communicate
  yield put(actions.showUndo(undoId))

  // optimistically mark the thread as `archived`
  yield put(actions.updateThread(thread))

  // allow the user 5 seconds to perform undo.
  // after 5 seconds, 'archive' will be the winner of the race-condition
  const { undo, archive } = yield race({
    undo: take(action => action.type === 'UNDO' && action.undoId === undoId),
    archive: delay(5000)
  })

  // hide undo UI element, the race condition has an answer
  yield put(actions.hideUndo(undoId))

  if (undo) {
    // revert thread to previous state
    yield put(actions.updateThread({ id: threadId, archived: false }))
  } else if (archive) {
    // make the API call to apply the changes remotely
    yield call(updateThreadApi, thread)
  }
}

function* main() {
  while (true) {
    // wait for an ARCHIVE_THREAD to happen
    const action = yield take('ARCHIVE_THREAD')
    // use spawn to execute onArchive in a non-blocking fashion, which also
    // prevents cancellation when main saga gets cancelled.
    // This helps us in keeping state in sync between server and client
    yield spawn(onArchive, action)
  }
}
```
