# 测试 Sagas

有两个主要的测试 Sagas 的方式：一步一步测试 saga generator function，或者执行整个 saga 并断言 side effects。

## 测试 Saga Generator Function

假设我们有以下的 action:

```javascript
const CHOOSE_COLOR = 'CHOOSE_COLOR';
const CHANGE_UI = 'CHANGE_UI';

const chooseColor = (color) => ({
  type: CHOOSE_COLOR,
  payload: {
    color,
  },
});

const changeUI = (color) => ({
  type: CHANGE_UI,
  payload: {
    color,
  },
});
```

我们想要测试 saga:

```javascript
function* changeColorSaga() {
  const action = yield take(CHOOSE_COLOR);
  yield put(changeUI(action.payload.color));
}
```

由于 Sagas 总是会 yield 一个 Effect，并且这些 effects 有简单的 factory function（例如 put, take 等等），测试将检查被 yield 的 effect，并拿它和期望返回的 effect 进行比对。
调用 `next().value` 来获取 saga 的第一个被 yield 的值：

```javascript
  const gen = changeColorSaga();

  assert.deepEqual(
    gen.next().value,
    take(CHOOSE_COLOR),
    'it should wait for a user to choose a color'
  );
```

应该返回一个值，赋给 `action` 常量，它将被用于 `put` effect 的参数：

```javascript
  const color = 'red';
  assert.deepEqual(
    gen.next(chooseColor(color)).value,
    put(changeUI(color)),
    'it should dispatch an action to change the ui'
  );
```

由于没有更多其他的 `yield`，下一次调用 `next()` 时，generator 将会完成：

```javascript
  assert.deepEqual(
    gen.next().done,
    true,
    'it should be done'
  );
```

### Branching Saga

有时候你的 saga 可能会有不同的结果。为了测试不同的 branch 而不重复所有流程，你可以使用 **cloneableGenerator** utility function

这时候我们新增两个 action，`CHOOSE_NUMBER` 和 `DO_STUFF`，使用一个 action creator 来创建：

```javascript
const CHOOSE_NUMBER = 'CHOOSE_NUMBER';
const DO_STUFF = 'DO_STUFF';

const chooseNumber = (number) => ({
  type: CHOOSE_NUMBER,
  payload: {
    number,
  },
});

const doStuff = () => ({
  type: DO_STUFF,
});
```

现在，在 `CHOOSE_NUMBER` action 之前，测试中的 saga 将会 put 两个 `DO_STUFF` action，然后根据数字是奇数还是偶数 put `changeUI('red')` 或 `changeUI('blue')`，

```javascript
function* doStuffThenChangeColor() {
  yield put(doStuff());
  yield put(doStuff());
  const action = yield take(CHOOSE_NUMBER);
  if (action.payload.number % 2 === 0) {
    yield put(changeUI('red'));
  } else {
    yield put(changeUI('blue'));
  }
}
```

测试如下：

```javascript
import { put, take } from 'redux-saga/effects';
import { cloneableGenerator } from 'redux-saga/utils';

test('doStuffThenChangeColor', assert => {
  const gen = cloneableGenerator(doStuffThenChangeColor)();
  gen.next(); // DO_STUFF
  gen.next(); // DO_STUFF
  gen.next(); // CHOOSE_NUMBER

  assert.test('user choose an even number', a => {
    // cloning the generator before sending data
    const clone = gen.clone();
    a.deepEqual(
      clone.next(chooseNumber(2)).value,
      put(changeUI('red')),
      'should change the color to red'
    );

    a.equal(
      clone.next().done,
      true,
      'it should be done'
    );

    a.end();
  });

  assert.test('user choose an odd number', a => {
    const clone = gen.clone();
    a.deepEqual(
      clone.next(chooseNumber(3)).value,
      put(changeUI('blue')),
      'should change the color to blue'
    );

    a.equal(
      clone.next().done,
      true,
      'it should be done'
    );

    a.end();
  });
});
```

参与 [Task cancellation](TaskCancellation.md) 来测试 fork effects

## 测试完整的 Saga

虽然测试 saga 的每一步可能是有用的，实际上这使得 brittle tests 成为可能。不过，运行整个 saga 并断言预期的 effects 已经产生可能会更好。

假设我们有一个简单的 saga，它将调用一个 HTTP API:

```javascript
function* callApi(url) {
  const someValue = yield select(somethingFromState);
  try {
    const result = yield call(myApi, url, someValue);
    yield put(success(result.json()));
    return result.status;
  } catch (e) {
    yield put(error(e));
    return -1;
  }
}
```

我们可以使用 mock 的数据来运行这个 saga：

```javascript
const dispatched = [];

const saga = runSaga({
  dispatch: (action) => dispatched.push(action),
  getState: () => ({ value: 'test' }),
}, callApi, 'http://url');
```

然后可以写一个测试来断言被 dispatch 的 action 和模拟的调用：

```javascript
import sinon from 'sinon';
import * as api from './api';

test('callApi', async (assert) => {
  const dispatched = [];
  sinon.stub(api, 'myApi').callsFake(() => ({
    json: () => ({
      some: 'value'
    })
  }));
  const url = 'http://url';
  const result = await runSaga({
    dispatch: (action) => dispatched.push(action),
    getState: () => ({ state: 'test' }),
  }, callApi, url).done;

  assert.true(myApi.calledWith(url, somethingFromState({ state: 'test' })));
  assert.deepEqual(dispatched, [success({ some: 'value' })]);
});
```

也可以查看仓库示例：

https://github.com/redux-saga/redux-saga/blob/master/examples/counter/test/sagas.js

https://github.com/redux-saga/redux-saga/blob/master/examples/shopping-cart/test/sagas.js
