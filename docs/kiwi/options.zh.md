# Kiwi 系统参数

| 参数 | 描述 | 默认值 | 备注 |
| -- | -- | -- | -- |
|`kiwi.event.asmdumpdir`| kiwi 事件系统中 `AsmListenerResolver` 生成类后导出的位置 | null | 谨供调试使用 |

## 构建参数

| 参数 | 描述 | 备注 |
| -- | -- | -- |
| -Dperfasm| 运行 kiwi:event:jmh 时使用 perfasm 剖析器 | |
| -Dfastdebug | 运行 kiwi:event:jmh 时使用 fastdebug 版本的 jdk 并且输出 jitlog | JDK 位置从环境变量 `FASTDEBUG_JDK_HOME` 中读取 |
