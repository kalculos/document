# Uni & Result

`Uni<T>` 是一个 `Consumer<Consumer<T>>`。简单的说，你传给他一个 `Consumer<T>` 的闭包（so-called "Lambda")，之后它会开始产生新元素，并且传递到你的闭包里面。   
每传一次闭包（调用 `onItem` 方法），`Uni<T>` 都会重新计算并且（在无副作用的情况下）使用相同的元素序列调用你的闭包，类似一个 `Supplier<Stream<T>>`。

消费一个 `Uni<Integer>`:
```java
void main(){
    Uni<Integer> uni = Uni.infiniteAscendingNumber().limit(6);
    // 打印 1, 2, 3, 4, 5, ..............
    uni.onItem(System.out::println);
    // 打印 0, 2, 4, ...
    uni.filter(it -> it % 2 == 0).onItem(System.out::println);
}
```

提供一个 `Uni<Integer>`:
```java
Uni<Integer> infiniteAscendingNumber(){
    return consumer -> {
        int i = 0;
        while(true) consumer.accept(i++); // 可以类比为： yield i++
    };
}
```

如果你曾经使用过其他语言中生成器特性的 `yield` 功能，你可以认为 `Uni<T>` 实际上就是别的语言中常见的生成器。  

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

如果你对 `Uni<T>` 的原理感兴趣，可以移步[这一篇文章](https://developer.aliyun.com/article/1199705)。 Kiwi 中的 `Uni<T>` 就是文中 `Seq<T>` 在工程上的进一步扩展。

## Interruption

在上面的例子中，你会发现 `infiniteAscendingNumber` 里是一个死循环，而 `limit()` 却可以限制元素的个数，这是通过受检异常类型 `Interruption` 实现的。  

`Interruption` 并不捆绑 `Uni`，你可以在程序里其他需要使用中断的地方也采用它实现控制流终止。它有且仅有一个单例： `Interruption.INTERRUPTION`。抛出 Interruption 的性能损耗相比一般的异常要小，这是因为 `Interruption` 并不会花时间填充栈帧，并且也不会被创建多个实例。

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

使用 `#then` 方法你可以扩展出许多新的操作。在 `UniOp` 类中有一些比较少用的管道操作（如 dispatch 和 executor），请自行查阅源代码。


## Result

`Result<T>` 是 `Uni<T>` 的子类型。它有且仅能有 `Some<T>` 和 `Fail<?>` 两种类型 (sealed)

Example:
```java


Result<UserInfo> result = Result.fromNotNull(() -> userDao.queryById(usrId));
return switch(result){
    // 使用模式匹配和 when clause 实现错误处理
    case Some(UserInfo user) when user.isAdmin -> "Welcome back, Administrator "+user;
    case Some(UserInfo user) -> "Welcome back! " + user;
    case Fail(Exception e) -> "An error occurred when gathering user info: "+e;
    // result == Fail.none()
    case Fail(Fail.Nothing n) -> "User not found!" 
}
```

或者像 `Optional<T>` 一样使用：
```java
Result<UserInfo> result = gatherUserInfo(usrId);
return result.orElseThrow(); // or #toOptional() -> Optional<T>
```

又因为 `Result<T>` 本身也是一个 `Uni<T>`, 因此 `Uni<T>` 中的管道操作同样适用：  
(但是要注意的是，使用除 `filter`, `map` 的管道后将会转化为 `Uni`，届时将无法使用 Result 中的方法)
```java
void test(){
    gatherUserInfo(usrId)
            .filter(UserInfo::isAdmin)
            .onItem(...);
    // 或者你可以直接
    gatherUserInfo(usrId).onItem(...);
}
```

对于 `Fail<?>` 类型，`onItem` 永远不会被调用，且 `T get()` 方法总是返回 null. 通常来说，你应该只考虑使用模式匹配（上文中第一种用法）或者作为 `Uni` 使用管道操作。由于 `onItem` 不会被调用，因此你可以在管道中不考虑失败/无值的情况，管道仅在成功时被调用。

### 使用 Result 捕获受检异常
`Result.fromAny(AnySupplier<T>)` 可以捕获 `AnySupplier<T>` 中的受检异常从而使用 `Result` 处理结果.

```java
Result.fromAny(() -> Files.readString(...))
        .onFail(...)
        .onItem(...);
```

如果你希望视 null 为错误，可以使用 `Result.fromNotNull`.  同样，他还有一个 `runAny`。更多细节请咨询查阅源代码。


