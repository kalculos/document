# Uni & Result

`Uni<T>` 是 kiwi-lang 的核心部分，在 kiwi 的其他组件中也不乏它的身影。简单的说，你可以把它理解为一个包装成 `Stream<T>` 模式的 `Supplier<T>`.   
因为其作为 `Supplier<T>` 的特性，这个 "流" 永远不会被消耗完——因为当你开始消耗的时，它会产生一个新的流，也就是元素的序列。

```java
void main(){
    Uni<Integer> uni = Uni.infiniteAscendingNumber().limit(6);
    // 打印 1, 2, 3, 4, 5, ..............
    uni.onItem(System.out::println);
    // 打印 0, 2, 4, ...
    uni.filter(it -> it % 2 == 0).onItem(System.out::println);
}
```
再深入一些，`Uni<T>` 实际上就是别的语言中常见的生成器。
```java
Uni<Integer> infiniteAscendingNumber(){
    return consumer -> {
        int i = 0;
        while(true) consumer.accept(i++); // 可以类比为： yield i++
    };
}
```

使用 `Uni<T>` 的生成器特性，你可以很方便的处理复杂的数据结构：
```java
    public static Uni<Method> iterateMethodsWithSuper(Class<?> clazz) {
        if (clazz == null) return Fail.none();
        return c -> {
            for (Method declaredMethod : clazz.getDeclaredMethods()) {
                c.onValue(declaredMethod);
            }
            for (Class<?> anInterface : clazz.getInterfaces()) {
                iterateMethodsWithSuper(anInterface).onItem(c);
            }
            iterateMethodsWithSuper(clazz.getSuperclass()).onItem(c);
        };
    }
    iterateMethodsWithSuper(...).onItem(...); // 递归遍历一颗类型树上的所有方法
```

如果你对 `Uni<T>` 的原理感兴趣，可以移步[这一篇文章](https://developer.aliyun.com/article/1199705), Kiwi 中的 `Uni<T>` 就是文中 `Seq<T>` 在工程上的进一步扩展。

## Interruption

在上面的例子中，你会发现 `infiniteAscendingNumber` 里是一个死循环，而 `limit()` 却可以限制元素的个数，这是通过受检异常类型 `Interruption` 实现的。它并不捆绑 `Uni`，你可以在程序里其他需要使用中断的地方也采用它实现控制流终止。

`Interruption` 有且仅有一个单例： `Interruption.INTERRUPTION`。抛出 Interruption 的性能损耗相比一般的异常要小，这是因为 `Interruption` 并不会花时间填充栈帧，并且也不会被创建多个实例。

`Uni<T>` 中还提供了专门针对 `Interruption` 的若干 `Consumer<T>` 变种，详情请移步源代码。

## UniOp

虽然 `Uni<T>` 中已经定义了若干管道方法，但是仍然有未能覆盖到的使用情景。好在，你可以使用 `Uni#then(UnaryOperator<T>)` 定义新的管道操作。

例子：对于 `Uni<T>` 中的每个元素，分派到 `Executor` 再运行接下来的管道

```java
import java.util.concurrent.ForkJoinPool;

void parallelUni() {
    users().then(UniOp.executor(ForkJoinPool.commonPool())).onItem(User::notifyPasswordEmail);
    // Equals to
    userStream().parallelStream().forEach(User::notifyPasswordEmail);
    
    users().then(
            UniOp.dispatch(
                    User::isVip, User::sendGift,
                    User::isBanned, it -> {} // empty operation
            )
    ).onItem(User::doSomethingForNormalUser);
}
```

使用 `#then` 方法你可以加入许多新的操作。在 `lang` 模块里已经编写若干比较少用的管道在 `UniOp` 类中，请自行查阅源代码。


## Result

`Result<T>` 是 `Uni<T>` 的子类型。它有且仅有 `Some<T>` 和 `Fail<?>` 两种类型

Example:
```java
Result<UserInfo> result = gatherUserInfo(usrId);
return switch(result){
    case Some(UserInfo user) when user.isAdmin -> "Welcome back, Administrator "+user;
    case Some(UserInfo user) -> "Welcome back! " + user;
    case Fail(Exception e) -> "An error occurred when gathering user info: "+e;
    case Fail(Fail.Nothing n) -> "User not found!" 
}
```

或者像 `Optional<T>` 一样使用：
```java
Result<UserInfo> result = gatherUserInfo(usrId);
return result.orElseThrow();
```

又因为 `Result<T>` 本身也是一个 `Uni<T>`, 因此 `Uni<T>` 中的管道操作同样适用：
```java
void test(){
    gatherUserInfo(usrId)
            .filter(UserInfo::isAdmin)
            .onItem(...);
    // 或者你可以直接
    gatherUserInfo(usrId).onItem(...);
}
```

对于 `Fail<?>` 类型，`onItem` 永远不会被调用，且 `T get()` 方法总是返回 null. 通常来说，你应该只考虑使用模式匹配（上文中第一种用法）或者作为 `Uni` 使用管道操作。由于 `onItem` 不会被调用，因此你可以在管道中大胆地使用各种方法而不需要考虑失败的情况。

### 使用 Result 捕获受检异常
`Result.fromAny(AnySupplier<T>)` 可以捕获 `AnySupplier<T>` 中的受检异常从而使用 `Result` 处理结果.

```java
Result.fromAny(() -> Files.readString(...))
        .onFail(...)
        .onItem(...);
```

同样，他还有一个 `runAny`。更多细节请咨询查阅源代码。


