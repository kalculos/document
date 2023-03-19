# 集合工具

Kiwi-Collection 提供了一些与 Java 集合相关的工具类。

## Lists

`Lists` 提供了一些针对 `List<T>` 的工具。

### 组合视图

我们有时候经常需要将两个或多个 `List` 合并作为某个参数传出或者用于遍历，这样做有两种方式：

1. **拷贝到新的 List** 这样做很简单，但是对于长集合或者高频操作来说相当消耗内存。
``` java
var newList = new ArrayList<T>(listA.size() + listB.size());
newList.addAll(listA);
newList.addAll(listB);
return newList;
```

使用 Kiwi: `Lists.ofCopiedCompoundList(listA,listB)`

2. **创建一个视图** 将两个 `List` 拼接到一起，避免花时间拷贝应用并且减少 GC 压力。
只需要使用 `Lists.ofCompoundListView(listA, listB)` 即可。

!!! warning

    `ofCompoundListView` 返回的 `List` 不支持部分操作，例如 `retainAll`, `removeAll`, `containsAll` 和 `subList`，即使未来可能会支持。
    使用组合视图的最佳实践是*使其不可变*，也就是说一旦创建了组合视图就不对其内容做变更。如果你发现组合后的 List 仍然需要大量修改操作，那么你应该考虑的是 `ofCopiedCompoundList` 而不是 `ofCompoundListView`，后者总是用于某些处理的中间过程。

    如果你必须暴露组合视图，请考虑使用 JDK 提供的 `Collections.unmodifiableList()` 进行包装。

### 非空列表

非空列表 `NonNullList<T>` 阻止你插入 `null` 元素，并且承诺集合内没有空元素。
可以使用 `Lists.noNullList(origin)` 修饰一个列表。

!!! note

    使用 `NonNullList` 时请准寻它的承诺，即被修饰过的集合内没有空元素。
    这可以通过判断 `origin.stream().anyMatch(Object::isNull)` 实现

### ArrayList 相关

1. **工厂方法** 你可以使用 `Lists.newArrayList(T...)` 创建一个新的 `ArrayList`，其容量即为参数数目。
2. **快删** 你可以使用 `Lists.fastRemove(list,index/object)` 以 `O(1)` 时间删除 `ArrayList` 内的某个元素，这会导致列表内的元素顺序发生改变。

!!! info

    `ArrayList` 的快删是通过将末尾的元素覆盖到目标元素的位置上实现的。
