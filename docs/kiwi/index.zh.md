# 简介

[Kiwi](https://github.com/kalculos/kiwi) 是一系列服务于 Java 程序日常开发的工具类，以最大可复用性为目标精挑细选而成。

# 上手使用

使用主流的包管理器，或者从 [Maven Central](https://search.maven.org/search?q=io.ib67.kiwi) 自行下载 JAR 包导入使用。

当前版本为：![badge](https://img.shields.io/github/v/release/kalculos/kiwi?style=flat-square)

添加到你的 `build.gradle`:

```groovy
def kiwiVersion = "0.4.1" // kiwi 的版本号，此处可能并不是最新版本
dependencies {
    api "io.ib67.kiwi:core:$kiwiVersion"
    api "io.ib67.kiwi:collection:$kiwiVersion"
}
```

*Kiwi 始终使用最新的长期支持版本的 Java 编译。*
