# 受检异常

Kiwi 里面提供了一些显式声明支持受检异常（exception-awared）的函数式接口，如 `AnyConsumer<T>`。使用这些接口编写匿名函数时，函数体内部的受检异常可以被忽略，交给 API 设计者管理。

例如：
```java
Kiwi.fromAny(()->Files.readString(Path.of("..."))).orElseThrow();
```

此处 `Files.readString` 可能投出的受检异常被 `fromAny` 的参数 `AnySupplier<T>` 捕获了，并且这个方法会返回一个 `Result<T,E>`。随后，如果 `readString` 出现了异常，那么 `orElseThrow` 返回的将不会是结果，而是投出一个 `RuntimeException` 其中包含一个 `IOException`。

它们大多是开箱即用的，用法和 `java.util.function` 中的大同小异（你需要处理可能出现的异常），并且可以转换为 `java.util.function` 中的对应原型（例如：`toConsumer`），在调用时如果遇到了受检异常将会抛出一个 `RuntimeException`。

## fromAny 和 runAny

我们不仅提供了受检异常容器的声明，也提供了用于优雅地处理他们的工具。`Kiwi.fromAny` 接受一个 `AnySupplier<T>` 并且将结果转换为一个 `Result<T,E>`，以此你就可以使用 `Result` 提供的异常处理方法或者直接作为某个过程的运行结果传递给 `Promise` 。

而 `runAny` 和 `fromAny` 很相似，只是它不要求你返回任何东西，它返回的成功 `Result` 结果总是 `null`，也就是说，如果配合 `Result#exceptNoNull` 使用的话无论如何都会失败。

!!! example

    ```java
    Kiwi.fromAny(this::fetchNetworkAPI)
      .exceptNoNull()
      .onSuccess(this::processData)
      .onFailure(it -> ... revert some operations ...);
    ```

