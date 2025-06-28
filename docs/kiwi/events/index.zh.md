# What's Kiwi-Event

`kiwi-event` 是一个基于 Kiwi 实现的简易 EventBus. 一个 EventBus 负责将事件传达给所有监听这个事件的 Handler.

# 上手使用

当前版本为：![badge](https://img.shields.io/github/v/release/kalculos/kiwi?style=flat-square)

添加到你的 `build.gradle`:

```groovy
dependencies {
    api "io.ib67.kiwi:event:$kiwiVersion"
}
```

在配置好依赖之后，我们就可以开始编写事件监听器了。

## 注册与监听
在使用 EventBus 之前，首先需要用于传递信息的事件。
```java
public record NumberEvent(int i) implements Event { }
```

接着，我们需要一个 EventBus 用来监听/传递事件。
```java
var bus = new HierarchyEventBus();
bus.register(NumberEvent.class, this::handleNumber);

void handleNumber(NumberEvent event){
    // ... do your business
}
```

之后你就可以往里面投递事件了。
```java
bus.post(new NumberEvent(1));
```

`HierarchyEventBus` 同样还支持传递到类层级上:
```java
bus.register(Event.class, this::handleAllEvent);

void handleAllEvent(Event event){
    // ... all events, including NumberEvent
}
```
`HierarchyEventBus` 是线程安全的。

## 泛型事件
在 `HierarchyEventBus` 中，所有的类型检索都是通过 [TypeToken](../typetoken.zh.md) 进行的，因此它天然的支持区分开泛型事件。
要实现支持泛型的事件，只需要在该事件的 `type()` 方法中根据情况特殊处理即可。
```java
record Box<A>(A a) implements Event{
    @Override
    public TypeToken<?> type(){
        return TypeToken.getParameterized(Box.class, a.getClass());
    }
}
```

在实际情景中，你可能会希望对 `type()` 的返回值进行缓存以加快事件派发的速度（在上文中由于篇幅关系省略掉了这些代码）。  
`HierarchyEventBus` 在第一次遇见不认识的类型时，会对该类型 -> Event 上整个链路的所有类型均按照泛型语义进行一次初始化。如果该事件类型同时具有两条通往 Event 的类型路径，则选取定义顺序里靠的最近的那个。

## 使用注解注册

kiwi-event 同样支持使用注解方法来注册事件监听器。

```java
class MyListener implements EventListenerHost {
    @SubscribeEvent
    void onNumber(NumberEvent number){
        // ...
    }
    
    @SubscribeEvent
    void onBox(BoxEvent<Integer> intBox){
        // ...
    }
}

new MyListener().registerTo(bus);
```

需要注意几点：  
1. 用于存放监听器的类需要实现 `EventListenerHost` 接口。  
2. 监听器方法应该使用 `default`, `public` 或 `protected` 访问修饰符，否则可能会导致性能下降。  
3. 如果使用 `registerTo` 方法注册到 EventBus 上，请确保你的项目 Jigsaw 模块（如果有）对 Kiwi 开放。  

`registerTo` 方法将会委托 `AsmListenerResolver` 扫描当前类上的所有 `@SubscribeEvent` 方法（不支持父类），并且为每个方法单独生成一个实现 `EventHandler` 的访问器注册到 bus 上。    
如果你需要使用 GraalVM Native-Image，那么 `AsmListenerResolver` 将会无法工作。届时请使用 `io.ib67.kiwi.event.util.ReflectionListenerResolver` 扫描然后自行注册，具有较好的兼容性（使用 tracing-agent 后）

这两种 ListenerResolver 均会使用 `MethodHandles.Lookup`. 因此如果你的代码不满足条件 (2) 可能会出现访问权限错误，届时自行传递 `MethodHandles.lookup()` 即可

## 关于性能
!!! note

    在对 kiwi-event 进行基准测试以及解读前，请确保你已经充分地阅读了 kiwi:event 中 jmh sourceSet 的代码
    并且明确测试目标，其基准测试代码的含义，以及理解测试后得到的数字的含义。不要假设他们想告诉 _你想让他们告诉_ 的东西。
    必要时，请咨询专家或者 Maintainer 的意见。


根据基准测试的结论来看，目前 HierarchyEventBus 的传递速度并不受类型层级的原因而产生明显的性能降级。传递速度主要与整条链路上的监听器个数有关。

EventBus 传递事件的代码链路上比较简单，并且不会引入额外的内存分配，传递一次事件只需要微秒（通常更低）量级的时间，可以适应绝大多数的情景。