# Future & Promise

Kiwi 提供了一个在异步编程中常见的 `Future` 的抽象以及一个开箱即用的实现。

## Future

`Future` 即是某个不知道什么时候完成计算的结果。虽然你现在不一定可以拿到结果，但是当结果产生时，它将会通知你。

结果分为 *成功* 和 *失败* 。成功和失败的定义取决于 `Future` 的泛型和使用它的人，失败可以是一些异常也可以是某些业务对象的副产品，而成功往往是运算得到的结果。

一个 `Future` 至多产生一个结果。

!!! example

    订阅事件：
    ``` java
    Future<Order,? extends Exception> future = someAsyncAPI();
    future.onSuccess(this::processOrder)
        .onFailure(exception -> ...)
        .onComplete(result -> {
            // ...
        });
    ```

无论是成功还是失败都会触发 `onComplete` 并且传入一个 `Result`。
除了订阅，也可以主动查询状态:

!!! example

    查询状态:
    ``` java
    future.isDone();
    ```

    或者堵塞当前线程到产生结果
    ``` java
    future.sync(); // 此处可能抛出 InterruptedException (1)
    ```

    1. 可以通过使用 `syncUninterruptible()` 避免受检异常。

## Result

`Result` 是一个状态确定的，不再可变的 `Future`，对于它的一切订阅操作都会根据情况直接被执行（实际上，已经完成的 `Future` 都会这样做）。

除此之外，`Result` 还提供了异常处理的方法用于处理运算结果。

!!! example

    ``` java
    var value = asyncResult.orElseThrow(); // 如果结果是一个异常就会尝试将它投掷出来。
    // 或者：
    var value = asyncResult.orElse(defaultValue); // 备选方案
    // 也可以:
    asyncResult.stream();
    asyncResult.isSuccess(); // 是否成功，对应的还有 isFailed
    ```
    但是用的比较多的还是 `onSuccess` 和 `onFailure` 。

    创建 Result:
    ``` java
    Result.ok(value); // 成功的结果
    Result.fail(failure); // 失败
    ```

你可以在 [受检异常处理](./exception-aware-functional-interfaces.zh.md) 一章找到 `Result` 在 Kiwi 中的实际使用案例。

## Promise

`Promise` 是一个可被完成的 `Future`。相比 `Future` 只多增了三个方法。

``` java
promise.success(value); // 传递成功的结果给所有的订阅者，并且更新这个 Promise / Future 的状态。
promise.failure(exception); // 以此类推。
// 也可以直接从 Result 读取结果。
promise.fromResult(result);
```

此外，`Promise` 还提供了几个值得注意的类。

### AbstractPromise

这是一个基本的 `Promise` 骨架实现，用户可以继承这个类快速开发适配业务场景的 `Promise`，如 `ScheduledFuture/Promise`，etc. 只需要实现 `sync()` 即可。（往往还需要重写 `setResult` 达到目的）

### TaskPromise

Kiwi 提供的开箱即用的 `Promise` 实现，其 `sync()` 采用同步块 ( `synchronized` ) 和 `Object.wait() / notifyAll()` 实现，对虚拟线程不友好。

!!! note

    在未来可能采用 `CountDownLatch` 实现。但无论使用何种方法实现，大多数情况下你应该总是使用 `onSuccess` 一类的订阅方法，这才是设计 `Future` 的本意。

!!! example

    一个使用 Promise + Future 的例子。
    ``` java
    public Future<HttpResponse,IOException> fetchAPI(){ // (1)
        var promise = new SynchronizedPromise(new TaskPromise()); // (2)
        executor.submit(() -> promise.fromResult(fromAny(()->fetchBlocking())));
        return promise;
    }
    ```

    1.  此处返回 `Future` 而不是 `Promise` 以此让返回结果不可变，改变 `Future` 状态的能力只在自己手里。
    2.  此处使用了一个 `SynchronizedPromise`，因为 `TaskPromise` 是线程不安全的。

    `fetchAPI` 将会非常快的返回结果，接着用户将会订阅这个 `Future` 在未来的某个时刻接收到通知。

    ``` java
    fetchAPI().onSuccess(...).onFailure(...)
    ```

### SynchronizedPromise

这是一个 `Promise` 的包装器，其所有的方法均使用了一个 `ReentrantLock` 用于保证线程安全。

``` java
var promise = new SynchronizedPromise(anotherPromise);
```
