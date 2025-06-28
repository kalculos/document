# What's Kiwi

Expressive Java utilites and robost libraries tuned for performance.

Documentation & API work in progress.

# 上手使用

当前版本为：![badge](https://img.shields.io/github/v/release/kalculos/kiwi?style=flat-square)

添加到你的 `build.gradle`:

```groovy
def kiwiVersion = "1.0.0" // kiwi 的版本号，此处可能并不是最新版本
repositories {
    maven {
        name = "Kalculos"
        url = "https://repo.sfclub.cc"
    }
}
dependencies {
    api "io.ib67.kiwi:lang:$kiwiVersion"
}
```

从 1.0.0 开始，Kiwi 使用 Java 21 编译。对于 0.xx 版本的 Kiwi 文档，可以在 [这里](https://github.com/kalculos/document/tree/11fb0e2015e3bbd50efa342d81960c8bc94acd2a) 找到。
