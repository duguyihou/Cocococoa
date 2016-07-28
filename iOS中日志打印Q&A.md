## iOS中日志打印Q&A

**打印当前的函数和行号？**

```objc
NSLog(@"%s:%d obj=%@", __func__, __LINE__, obj);
```

其中func和LINE都是预编译的宏，编译时会分别替换为当前函数和当前行号。 下面是一些常用于打印日志的宏。

宏                   | 说明
------------------- | ----------------------------
**func**            | 打印当前函数或方法，c字符串
**LINE**            | 打印当前行号，整数
**FILE**            | 打印当前文件路径，c字符串
**PRETTY_FUNCTION** | 打印当前函数或方法（在C++中会包含参数类型），c字符串

**打印一个类名，消息名，当前堆栈信息？** 使用以下方法在运行时动态获取这些信息。 代码 | 说明 --- | --- `NSStringFromSelector(SEL)` | 获取selector的名字 `NSStringFromSelector(_cmd)` | 获取当前方法名 `NSStringFromClass([object class])` | 获取object的类名 `[NSThread callStackSymbols]` | 获取当前线程的栈，是一个NSArry，包含堆栈中所有函数名。

**将日志打印到一个文件?** 使用freopen函数重定向标准输出和标准出错文件。因为printf函数会向标准输出（stdout）打印，而NSLog函数会向标准出错（stderr）打印。重新定向标准输出（stdout）和标准出错（stderr）到一个文件将会使他们打印日志到一个文件中。

```objc
freopen(“/tmp/log.txt”, “a+”, stdout);
freopen(“/tmp/log.txt”, “a+”, stderr);
```
