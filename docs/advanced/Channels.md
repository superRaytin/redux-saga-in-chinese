# 使用 Channels

They can also be used to queue specific actions from the Store.
到目前为止，我们已经使用了 `take` 和 `put` 来与 Redux Store 通信。Channels 概括了这些 Effects 与外部事件源或 Sagas 之间的通信。
它们还可以用于在 Store 中对特定的 actions 进行排序。

这一节，我们将会看到：

- 如何在 Store 中用 `yield actionChannel` Effect 缓存特定的 action。

- 如何使用 `eventChannel` factory function 连接 `take` Effects 至外部的事件来源。

- 如何使用通用的 `channel` factory function 创建一个 channel，并在 `take`/`put` Effects 中使用它来让两个 Saga 之间进行通信。

## 使用 `actionChannel` Effect

让我们来回顾一下那个经典的例子：

```javascript
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  while (true) {
    const {payload} = yield take('REQUEST')
    yield fork(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

以上的例子演示了经典的 *watch-and-fork* 模式。`watchRequests` 这个 saga 使用 `fork` 来避免阻塞，也因此它不会错过任何来自 store 的 action。
`handleRequest` 任务是在收到每个 `REQUEST` action 时被创建，因此，如果以很快的速度触发多个 action，那么就会有多个 `handleRequest` 任务同时执行。

假设我们的需求如下：每次只处理一个 `REQUEST` actions。比如有 4 个 `REQUEST` action，我们想要一个一个处理，处理完第一个 action 之后再处理第二个，如此等等...

我们想要的是 *queue（队列）* 所有没被处理的 action，一旦我们处理完当前的 request，就可以从队列中获取下一个的信息。

Redux-Saga 提供了一个 helper Effect: `actionChannel` 可以处理这些东西。让我们看看如何重写前一个例子：

```javascript
import { take, actionChannel, call, ... } from 'redux-saga/effects'

function* watchRequests() {
  // 1- 为 REQUEST actions 创建一个 channel
  const requestChan = yield actionChannel('REQUEST')
  while (true) {
    // 2- take from the channel
    const {payload} = yield take(requestChan)
    // 3- 注意这里我们用了一个阻塞调用
    yield call(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

第一件事是创建一个 action channel。我们使用 `yield actionChannel(pattern)`，pattern 可以理解成我们之前提到的 `take(pattern)` 中的 pattern，使用的是同样的规则。
两种形式的区别在于，如果 Saga 还没有准备好接收它们（比如一个被阻塞的 API 调用），`actionChannel` *能够缓存传入的消息*。

接下来是 `yield take(requestChan)`。除了使用 `pattern` 来从 Redux Store 接收特定的 action，`take` 还可以与 channels 一起使用（在上面我们从特定的 Redux action 创建了 channel 对象）。
`take` 将会阻塞 Saga，直到 channel 上有一个可用的消息。如果有一个消息被储存在缓存区，`take` 也可以立即恢复。

重要的是，我们要注意如何使用一个阻塞的 `call`。`call(handleRequest)` 返回之前，Saga 将保持阻塞。但与此同时，如果其他的 `REQUEST` action 在 Saga 仍被阻塞的情况下被 dispatch，
它们将被 `requestChan` 队列在内部。当 Saga 从 `call(handleRequest)` 恢复并执行下一个 `yield take(requestChan)` 时，`take` 将 resolve 被队列的消息。

默认情况下，`actionChannel` 会无限制缓存所有传入的消息。如果你想要更多地控制缓存，你可以提供一个 Buffer 参数给 effect creator。
Redux-Saga 提供了一些常用的 buffers（none, dropping, sliding），当然你也可以提供自己的 buffer 实现。详细请查看 [API 文档](../api#buffers)。

例如，如果你只想要处理最近的五个项目，那么你可以用：

```javascript
import { buffers } from 'redux-saga'
import { actionChannel } from 'redux-saga/effects'

function* watchRequests() {
  const requestChan = yield actionChannel('REQUEST', buffers.sliding(5))
  ...
}
```

## 使用 `eventChannel` factory 连接外部的事件

类似 `actionChannel` (Effect)，`eventChannel`（一个 factory function, 不是一个 Effect）为 Redux Store 以外的事件来源创建一个 Channel。

以下是一个从 interval 创建 Channel 的简单例子：

```javascript
import { eventChannel, END } from 'redux-saga'

function countdown(secs) {
  return eventChannel(emitter => {
      const iv = setInterval(() => {
        secs -= 1
        if (secs > 0) {
          emitter(secs)
        } else {
          // 这里将导致 channel 关闭
          emitter(END)
        }
      }, 1000);
      // subscriber 必须回传一个 unsubscribe 函数
      return () => {
        clearInterval(iv)
      }
    }
  )
}
```

`eventChannel` 中第一个参数是一个 *subscriber* 函数。
subscriber 的职责是初始化外部的事件来源（上面使用 `setInterval`），通过调用提供的 `emitter`，将事件来源传入的所有事件路由到 channel。
上面的例子中，我们在每秒调用 `emitter`。

> 注意：你需要清除你的事件来源，而不是通过事件 channel 传递 null 或 undefined。虽然可以通过数字传递，但我们推荐像 redux action 一样，组织你的 event channel 数据。 用 `{ number }` 而不是 `number`.

注意，也可以调用 `emitter(END)`。我们使用它来通知 channel 消费者：channel 已经关闭了，意味着没有其他消息能够通过这个 channel 了。

让我们看看如何从 Saga 使用这个 channel。（这个例子来自前面的 cancellable-counter 示例）

```javascript
import { take, put, call } from 'redux-saga/effects'
import { eventChannel, END } from 'redux-saga'

// 每隔一秒创建一个事件 Channel
function countdown(seconds) { ... }

export function* saga() {
  const chan = yield call(countdown, value)
  try {
    while (true) {
      // take(END) 将造成 saga 终止，跳到 finally 区块
      let seconds = yield take(chan)
      console.log(`countdown: ${seconds}`)
    }
  } finally {
    console.log('countdown terminated')
  }
}
```

所以 Saga yield 了一个 `take(chan)` 造成 Saga 阻塞，直到一条消息被 put 到 channel。在上面的示例中，对应的是调用 `emitter(secs)`。
注意，我们还在一个 `try/finally` 区块中执行整个 `while (true) {...}` 循环。当 interval 终止时，countdown 函数通过调用 `emitter(END)` 关闭 event channel。
在 channel 的 `take` effect 关闭 channel 会终止所有被阻塞的 Saga。
在我们例子中，终止 Saga 将造成它跳到 `finally` 区块（如果提供了的话，否则 Saga 只是简单终止）。

订阅者回传一个 `unsubscribe` 函数，用于在事件来源完成之前通过 channel 取消订阅。
在 Saga 中消费来自事件 channel 的消息，如果我们想要在事件来源完成之前*提前离开*（比如 Saga 被取消了）。
你可以从来源调用 `chan.close()` 关闭 channel 并取消订阅。

举个例子，我们可以让 Saga 支持取消：

```javascript
import { take, put, call, cancelled } from 'redux-saga/effects'
import { eventChannel, END } from 'redux-saga'

// 每隔一秒创建一个事件 Channel
function countdown(seconds) { ... }

export function* saga() {
  const chan = yield call(countdown, value)
  try {
    while (true) {
      let seconds = yield take(chan)
      console.log(`countdown: ${seconds}`)
    }
  } finally {
    if (yield cancelled()) {
      chan.close()
      console.log('countdown cancelled')
    }
  }
}
```

以下是另外一个例子，如何使用事件 channel 传递 WebSocket 事件到你的 saga（例如：使用 socket.io 类库）。
假设你正在等待一个服务器消息 `ping`，然后在一些延迟后回复一个 `pong` 消息。


```javascript
import { take, put, call, apply } from 'redux-saga/effects'
import { eventChannel, delay } from 'redux-saga'
import { createWebSocketConnection } from './socketConnection'

// 这个函数从给定的 socket 创建一个 event channel
// 设定传入 `ping` 事件的 subscription
function createSocketChannel(socket) {
  // `eventChannel` 接收一个 subscriber 函数
  // 这个 subscriber 接收一个 `emit` 参数，用来把消息放到 channel 上
  return eventChannel(emit => {

    const pingHandler = (event) => {
      // 把 event payload 放入 channel
      // 这可以让 Saga 从被回传的 channel 接收这个 payload
      emit(event.payload)
    }

    // 设定 subscription
    socket.on('ping', pingHandler)

    // subscriber 必须返回一个 unsubscribe 函数
    // unsubscribe 将在 saga 调用 `channel.close` 方法时被调用
    const unsubscribe = () => {
      socket.off('ping', pingHandler)
    }

    return unsubscribe
  })
}

// 通过调用 `socket.emit('pong')` 回复一个 `pong` 消息
function* pong(socket) {
  yield call(delay, 5000)
  yield apply(socket, socket.emit, ['pong']) // 把 `emit` 作为一个方法调用，并以 `socket` 为上下文
}

export function* watchOnPings() {
  const socket = yield call(createWebSocketConnection)
  const socketChannel = yield call(createSocketChannel, socket)

  while (true) {
    const payload = yield take(socketChannel)
    yield put({ type: INCOMING_PONG_PAYLOAD, payload })
    yield fork(pong, socket)
  }
}
```

> 注意：eventChannel 上的消息默认不会被缓存。为了给 channel 指定缓存策略（例如 `eventChannel(subscriber, buffer)`），你必须提供一个缓存给 eventChannel factory。更多信息请 [查看 API 文档](../api#buffers)。

### 使用 channels 在 Sagas 之间沟通

除了 action channel 和 event channel。你也可以直接创建 channel，默认不用连接任何来源。
你可以在 channel 上手动 `put`。当你想要在 sagas 之间进行沟通使用 channel 是非常方便的。

为了演示，让我们回顾一下之前处理请求的例子。

```javascript
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  while (true) {
    const { payload } = yield take('REQUEST')
    yield fork(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

我们看到，watch-and-fork 模式允许同时处理多个请求，并且不限制同时执行的任务的数量。
然后我们使用 `actionChannel` effect 来限制并发一次执行一个任务。

假设我们的要求是在同一时间内最多执行三次任务。当我们收到一个请求并且执行的任务少于三个时，我们会立即处理请求，否则我们将任务放入队列，并等待其中一个 *slots* 完成。

下面是一个使用 channel 的解决方案：

```javascript
import { channel } from 'redux-saga'
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  // 创建一个 channel 来队列传入的请求
  const chan = yield call(channel)

  // 创建 3 个 worker 'threads'
  for (var i = 0; i < 3; i++) {
    yield fork(handleRequest, chan)
  }

  while (true) {
    const {payload} = yield take('REQUEST')
    yield put(chan, payload)
  }
}

function* handleRequest(chan) {
  while (true) {
    const payload = yield take(chan)
    // process the request
  }
}
```

在上面的例子中，我们使用 `channel` factory 创建了一个 channel。
我们得到的这个 channel，默认缓存了所有我们放入的消息（除非有一个正在等待的 taker，当有消息这个 taker 会立即恢复）。

`watchRequests` saga fork 了 3 个 worker saga。注意，创建的 channel 将提供给所有被 fork 的 saga。
`watchRequests` 将使用这个 channel 来 *dispatch* 工作到那三个 worker saga。每一个 `REQUEST` action，Saga 只简单地在 channel 上放入 payload。
payload 然后会被任何 *空闲* 的 worker 接收。否则它将被 channel 放入队列，直到一个 worker saga 空闲下来准备接收它。

这三个 worker 都执行一个典型的 while 循环。每次迭代时 worker 将 take 下一次的请求，或者阻塞直到有可用的消息。
注意，这个机制为 3 个 worker 提供了一个自动的负载均衡。快的 worker 不会被慢的 worker 拖慢。