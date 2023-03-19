# 杂类
此处是一些小工具，为了方便将他们放在一个页面里介绍。

## 元组
Kiwi 内置了 `Pair` 和 `Triple`，分别用于将两个或三个对象绑定到一起

!!! example

    ```java
    var pair = Kiwi.pairOf(player,role);
    pair.left; //  player

    Kiwi.tripleOf(player,data,something);
    ```

## AutoClosable 的 ReentrantLock 包装器
虽然已经有 `synchronized` 了，但是有时候我们不能用它：

1. **代码和虚拟线程有关。** 使用 `synchronized` 将会堵塞平台线程，干扰正常挂起。
2. **只有 Lock。** 你必须得使用 API 或者旧代码里面留下的 `Lock`

一个 `ClosableLock` 可以通过 `Kiwi.withLock(lock)` 创建。

!!! example

    before:
    ```java
    try{
        lock.lock();
        // do your business...
    }catch(Exception e){
        // ... if lock isn't acquired
    }finally{
        lock.unlock();
    }
    ```

    after:
    ```java
    try(var ignored = withLock(lock)){
    // do your businesses...
    }
    ```

也可以使用 `withTryLock` 代替 `withLock` 从而限时等待锁。

## 泛型魔法

!!! warning

    **请慎重考虑使用，这不是类型转换工具，只是用于欺骗编译器，乱用会导致运行时的 `ClassCastException` 异常。**

Java 的泛型通过 *类型擦除* 实现，这意味着 `Collection<String>` 和 `Collection<Order>` 实际上是同一个类型，因为在 javac 编译之后他们的类型信息都会被擦为 `Collection` (本质上：`Collection<Object`)。

有时候你会被上下界限定弄得头昏脑胀，然而你又能绝对确认类型安全。那么这个时候你就需要使用 `typeMagic` 进行强制转换。

!!! example

    ```java
    @RequiredArgsConstructor
    public class CompoundSabeeStmt<I> implements SabeeStmt<I, Unit> {
        private final List<SabeeStmt<?, ?>> statements;

        @Override
        public Unit apply(I i) {
            for (SabeeStmt<I, ?> statement : typeMagic(statements)) {
                statement.apply(i);
            }
            return Unit.INSTANCE;
        }
    }
    ```

## 同步化异步

!!! warning

    乱用此工具很容易被揍

使用 `deAsync(Consumer<Promise,?>) throws InterruptedException` 可以堵塞当前线程，一直到 `Promise` 被完成为止。

用在虚拟线程上可以用于实现一些比较神奇的功能，例如控制挂起的某种 DSL。

!!! example

    ```java
        var result = fromAny(()->deAsync(promise -> asyncApi.onCallback(promise::success));) // 处理受检异常
        .orElseThrow();
    ```
## 延迟初始化

Kiwi 提供了两个 `byLazy` 用于创建会把计算过程延后的 `Supplier` 和 `Function`。

在第一次 `get` 或 `apply` 之后，计算结果将被缓存起来。

!!! example

    ```java
    var businessData = Kiwi.byLazy(() -> someExpensiveOperations());
    ```

## 方法占位符

`Kiwi.todo(message)` 用于填充一个空的方法使其通过编译，但是这个方法被调用的时候会抛出异常。

!!! example

    ```java
    public List<BakedModel> bakeModels(){
        return Kiwi.todo("bake models"); // (1)
    }
    ```

    1.  也可以直接 Kiwi.todo()


# 平台

`KiwiPlatform` 是一个用于获取当前运行平台情况的工具类。

!!! example

    ``` java
    KiwiPlatform.IS_AOT; // (1)
    KiwiPlatform.HAS_GUI; // 是否有 GUI 支持，Windows 下总是 `true`
    PlatformType platformType = KiwiPlatform.CURRENT; // 当前系统类型
    ```

    1.  当前是否运行在 native-image 内，可能需要配合 `--initialize-at-build-time` 使用。
