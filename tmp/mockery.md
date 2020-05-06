# Mockery

## 关键名词

1. test doubles ： 有stubs，mocks，spies三种类型
2. stubs
3. mocks
4. spies

Stubs and mocks are created the same. The difference between the two is that a stub only returns a preset result when called, while a mock needs to have expectations set on the method calls it expects to receive.

Spies are a type of test doubles that keep track of the calls they received, and allow us to inspect these calls after the fact.

The stub or mock object will have the type of MyClass, through inheritance.

5. Partial Test Doubles （用于只想模拟一个类的部分方法时）
   1. runtime partial test doubles
   2. generated partial test doubles
   3. proxied partial test doubles

## 重点

Mockery::close()的作用 ： This static call cleans up the Mockery container used by the current test, and run any verification tasks needed for our expectations.