# 一个常见的抽象概念: Effect

概括来说，从 Saga 内触发异步操作（Side Effect）总是由 yield 一些声明式的 Effect 来完成的
（你也可以直接 yield Promise，但是这会让测试变得困难，就像我们在第一节中看到的一样）。

一个 Saga 所做的实际上是组合那些所有的 Effect，共同实现所需的控制流。
最简单的是只需把 yield 一个接一个地放置，就可对 yield 过的 Effect 进行排序。你也可以使用熟悉的控制流操作符（`if`, `while`, `for`）
来实现更复杂的控制流。

我们已经看到，使用 Effect 诸如 `call` 和 `put`，与高阶 API 如 `takeEvery` 相结合，让我们实现与 `redux-thunk` 同样的东西，
但又有额外的易于测试的好处。

 但 `redux-saga` 相比 `redux-thunk` 还提供了另一种好处。
 在「高级」一节，你会遇到一些更强大的 Effect，让你可以表达更复杂的控制流的同时，仍然拥有可测试性的好处。
