# 反射工具

Kiwi 提供了一些常用反射操作的封装。

## AccessibleClass

`AccessibleClass` 尽一切可能达到目的（例如：使用 `IMPL_LOOKUP` 查找 `MethodHandle`），而对应的 `AccessibleField` 获取/写入内容时候将会尝试使用 Getter / Setter 然后再使用 `Unsafe`。

!!! example

    ```java
    var clazz = Kiwi.accessClass(Order.class);
    clazz.toMap(order); // Map<String,Object>, 将字段内容全部提取出来
    clazz.toString(order); // 将所有内容打印出来，用于 debug
    clazz.newInstance(); // 使用所有方法尝试创建对象，但是不保证这个对象会被初始化（例如使用 allocateInstance 时）
    // ...

    var field = clazz.staticField("someStaticFieldName");
    field.set/get(); // setter / getter.
    ```

具体内容请查阅 Javadoc.

## Unsafe

对 `jdk.internal.Unsafe` 的一层封装。可以直接得到 `Unsafe` 也可以使用其静态 shortcut 访问 (例如`Unsafe.ensureClassInitialized()`）

## MutableReference

JVM 中分有多种引用类型，强引用，弱引用。
而 Kiwi 提供的 `MutableReference` 是一个可以随时修改强弱的引用，但是不保证对象还在。
