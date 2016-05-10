#同时执行多个任务

`yield` 指令可以很简单的将异步控制流以同步的写法表现出来，与此同时我们也会需要同时执行多个任务，我们不应该单纯的写出：

```javascript
// 错的，effects将被按顺序执行
const users  = yield call(fetch, '/users'),
      repos = yield call(fetch, '/repos')
```

因为第二个effect将会在第一个`call`执行完毕才开始。所以我们需要写：

```javascript
import { call } from 'redux-saga/effects'

// 正确写法, effects 将会同步执行
const [users, repos]  = yield [
  call(fetch, '/users'),
  call(fetch, '/repos')
]
```

当我们需要 `yield` 一个包含 effects的数组， generator会被阻塞直到所有的effects都执行完毕，或者当一个effect被拒绝 （就像`Promise.all`的行为）
