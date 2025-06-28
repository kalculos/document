# TypeToken

Java 有泛型，即类型参数。但是 Javac 在编译的时会将类型参数尽数擦除，使得 `List<String>` 与 `List` 等价，这就叫做泛型擦除。而 `TypeToken<C>` 就是为了
在这个大前提下在运行时表达泛型而产生的。

## 基本用法
捕获类型：  
```java
var tk = new TypeToken<List<String>>(){};
tk.getBaseTypeRaw(); // List.class
tk.getTypeParams().getFirst(); // String.class
tk.toString(); // List<String>
```

除了 subclassing，你也可以通过填充参数手动构造任意 `TypeToken<C>`:
```java
var tk = TypeToken.getParameterized(List.class, String.class); // equals to new TypeToken<List<String>>(){};
// Resolve a token by class (or parameterizedType)
var stringTk = TypeToken.resolve(String.class);
var listObjTk = TypeToken.resolve(List.class); // List<Object>, Parameters will be filled by TypeVars' upper/lower bound.
```

也支持 Wildcard:
```java
var tk = new TypeToken<List<? extends CharSequence>>(){};
tk.toString(); // List<? extends CharSequence>
tk.getTypeParams().getFirst().isWildcard(); // true
```

你可以通过 `reduceBounds` 将 `TypeToken<C>` 树内的所有 Wildcard 都归并到它们的上下界上。

```java
TypeToken.reduceBounds(tk, /* liftBySuper */ true); // TypeToken<List<CharSequence>>
```

## 类型推导

TypeToken 还支持通过已有的信息推导出具体的类型。

```java
interface Cube<T> {

}

class Box<A> implements Cube<A> {
    A type;
}

var boxTk = TypeToken.getParameterized(Box.class, String.class);
boxTk.resolveField(Box.class.getDeclaredField("type")); // String.class
boxTk.inferType(Cube.class); // TypeToken<Cube<String>>
```

目前 resolveType 尚不支持 wildcard。  
此外，`TypeToken<C>` 内还有很多工具方法，如寻找到某个超类的路径等，详情请查阅 Javadoc.  

TypeToken 是线程安全的，且内部有一个对于类型的 `WeakHashMap` 缓存以避免重复创建大量垃圾。该缓存使用 `synchronized` 代码块进行同步，在 JEP 491 (即 Java 24) 前可能会干扰到虚拟线程的性能，请酌情使用。
