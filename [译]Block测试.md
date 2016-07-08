# [译]Block测试
>源自[Objective-C Blocks Quiz](http://blog.parse.com/learn/engineering/objective-c-blocks-quiz/)

你想知道Objective-C中blocks是怎么工作的吗？那么让我们通过几个测试题来了解下吧。
本文所有的例子都经过以下版本的LLVM检验过：
```
Apple clang version 4.1 (tags/Apple/clang-421.11.66) (based on LLVM 3.1svn)
Target: x86_64-apple-darwin11.4.2
Thread model: posix
```
1. always works ?
2. only works with ARC ?
3. only works without ARC ?
4. never works ?

## Example A

```
void exampleA() {
  char a = 'A';
  ^{
    printf("%cn", a);
  }();
}
```

## Example B

```
void exampleB_addBlockToArray(NSMutableArray *array) {
  char b = 'B';
  [array addObject:^{
    printf("%cn", b);
  }];
}

void exampleB() {
  NSMutableArray *array = [NSMutableArray array];
  exampleB_addBlockToArray(array);
  void (^block)() = [array objectAtIndex:0];
  block();
}
```

## EXample C

```
void exampleC_addBlockToArray(NSMutableArray *array) {
  [array addObject:^{
    printf("Cn");
  }];
}

void exampleC() {
  NSMutableArray *array = [NSMutableArray array];
  exampleC_addBlockToArray(array);
  void (^block)() = [array objectAtIndex:0];
  block();
}
```

## Example D


```
typedef void (^dBlock)();

dBlock exampleD_getBlock() {
  char d = 'D';
  return ^{
    printf("%cn", d);
  };
}

void exampleD() {
  exampleD_getBlock()();
}
```

## Example E

```
typedef void (^eBlock)();

eBlock exampleE_getBlock() {
  char e = 'E';
  void (^block)() = ^{
    printf("%cn", e);
  };
  return block;
}

void exampleE() {
  eBlock block = exampleE_getBlock();
  block();
}
```

## 解析

Example A: always works
>不管在 ARC 还是 MRC 下，不论 block 存放在 stack 还是 heap 内存中，当example A 被调用时，block 仍然有效，都能正常执行.

Example B: only works with ARC
>在 MRC 下，exampleB_addBlockToArray 中的 block 是 NSStackBlock 类型，存放在stack内存中。当执行 exampleB 时，stack 内存被释放，block 失效.

>在 ARC 下，block 是 autoreleased NSMallocBlock 类型，存放在 heap 内存中，所以 Exmaple B only works with ARC.

Example C: always works
>当 block 不需要从外部获取变量时，它不需要在 runtime 设置任何状态。此时，block 被编译成 NSGlobalBlock 类型，放在内存 data 段，就像 C 函数一样，属于代码的一部分，所以 always works.

Example D: only works with ARC
>这题有点类似于 Example B. 在 MRC 下，exampleD_getBlock 中的block 会被创建在 stack 内存中，当函数返回时，block马上失效。鉴于本题的错误实在太明显，编译器在编译时，就会抛出错误 error: returning block that lives on the local stack.

>而在 ARC 下，block 会被编译成 autoreleased NSMallocBlock 类型，存放于 heap 内存中。

>所以 only works with ARC.

Example E: only works with ARC
>本题类似于 Example D，区别在于本题代码不会出现编译错误，而是在运行时才会崩溃。更槽糕的是，如果你关闭了编译器优化选项，代码运行正常，而无法发现这个隐藏的bug。

>而在 ARC 下，block 会被编译成 autoreleased NSMallocBlock 类型，存放于 heap 内存中。

>所以 only works with ARC.

## 总结
以上这么多例子告诉我们什么？告诉我们要使用ARC！在ARC下，block总能正确运行。如果你不用ARC，最好能保证在 stack 内存中声明定义的block，能够拷贝到heap内存，保证block的正常运行。

当然，事情并不是这么简单，查看[苹果官方文档](https://developer.apple.com/library/ios/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html)
>Blocks “just work” when you pass blocks up the stack in ARC mode, such as in a return. You don’t have to call Block Copy any more. You still need to use [^{} copy] when passing “down” the stack into arrayWithObjects: and other methods that do a retain.
