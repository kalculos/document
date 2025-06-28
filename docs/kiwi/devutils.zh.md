# 反射工具

!!! warning 

    Kiwi 提供的若干反射工具中有一部分类被标注了 `@ApiStatus.Experimental`. 这是因为他们直接
    或间接的依赖于 `sun.misc.Unsafe` 以及 MethodHandles 中的特权 Lookup. 这两者均有可能在未来的某个 JDK
    版本上被禁止使用，因此你应该避免将业务代码依赖在这些不稳定的 API 上。

## Unsafe & Diag

`io.ib67.kiwi.reflection.Unsafe` 提供了对 `jdk.internal.Unsafe` / `sun.misc.Unsafe` 以及 Trusted Lookup 的访问。这些类
属于 OpenJDK 的实现细节，并有可能在未来被移除，或是其操作权限范围，语义有可能被改变。因此请仅当你知道且必要的情况下使用这些工具。

`Diag` 是一些用于帮助开发者诊断问题的工具。目前只有 `dumpObjectTree(int indent, Object obj)` 可用。其返回一个 `Uni<String>`, 该 Uni 中的
每个 String 对应一行，你可以直接打印到 `PrintStream` (System.out) 或是储存起来转储使用。

输出示例：  
`Diag.dumpObjectTree(2, List.of(1,2,3,4)).onItem(System.out::println);`

```
j.u.ImmutableCollections$ListN #2d72f75e {
  elements (super) = 4 [
    0. 1
    1. 2
    2. 3
    3. 4
  ]
  allowNulls (super) = false
}
```

文法：  
  - 对象: `fqdn.Class[#Sub[#Sub..] #hashCode { kvPair[] }`  
  - kvPair: `[(inaccessible)] fieldName [(super)] = value`  
  - value:  
    - Primitive and String: Their value. `1` `"hello"` `true`  
    - Primitive Array: `num [ elements[] ]`  
    - Array: `len [ (index. element\n)[] ]`  
    - Visited Object (recursive reference): `(visited) #hashCode`  
    - null: `null`  

说人话：  
  - `(inaccessible) data` 就是无法访问到这个字段的值（通常不可能出现）  
  - `element (super)` 就是这个字段是你给的对象的父类里面来的  
  - 数组开头的数字是元素个数，对于非原语的数组至少一行一个元素，每个元素开头的数字是它的下标  
  - 对于在打印之前已经访问过的数据，会被标记 `(visited)` 表示已经访问过了，避免递归  
  - 对象的类名会被缩写成每段包名只保留第一个字母的样子，比如 `j.l.String`  

通常你只应该在调试代码里使用 `Diag` 工具。

## Reflections
`Reflections` 包含了一些辅助反射的工具，并且这些工具没有涉及到 `Unsafe` 内容，可以放心使用。具体有哪些工具请移步 Javadocs