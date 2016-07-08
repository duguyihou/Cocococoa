# [转]GCD外传：dispatch_once(下)
> 注意，本篇所讨论的使用方式（动态predicate）并不提倡用于实际项目，仅作为深入探究和学习dispatch_once的一种方法，讨论的范围也在官方文档定义之外，其可靠性不能被保证。


在本系列前两篇文章中，我们一起学习了dispatch_once的作用、工作原理以及效率研究：
dispatch_once使得block中的代码执行且只执行一次，在多线程竞态时，使其他线程进入等待状态直至block执行完毕，并且还保证无竞态时执行效率与非线程安全的if语句效率相当。
dispatch_once内部使用了大量的原子操作来替代锁与信号量，这使得其效率大大提升，但带来的是维护和阅读性的降低。
dispatch_once被大量使用在构建单例上，apple也推荐如此。
但是我们可能会有两个疑问：
使用dispatch_once实现的单例，在初始化后，难以简单做到反初始化或者重初始化，如何解决？
使用dispatch_once时，static predicate一定程度限制了dispatch_once的使用场景，又如何解决。

本篇我们一起探究使用非static predicate的可能性，在后续系列文章中，我还将和大家一起探讨如何安全地重置predicate。

## 文档参考
如果查阅dispatch_once函数文档，对于predicate，文档有如下说明：

>The predicate must point to a variable stored in global or static scope. The result of using a predicate with automatic or dynamic storage (including Objective-C instance variables) is undefined.

再查阅dispatch_once_t的头文件注释，得如下说明：
> A predicate for use with dispatch_once(). It must be initialized to zero.
Note: static and global variables default to zero.

dispatch_once函数文档告诉我们：不要使用动态分配的predicate；

而dispatch_once_t的头文件注释告诉我们，predicate需要先初始化为0，并且提醒我们，static和global变量创建出来即有默认值0，但没有指明不可以动态分配。

## 先进行简单的分析
### 从程序设计角度
我们知道一个单例一般是从程序第一次调用它的get方法或者类似的生成方法时，惰性加载的，一旦加载，大多数情况下会一直为其它模块服务。

有时候我们可能需要重置这些单例，比如某个用户点击了reset按钮，或者选择了注销账号等等操作，这时候程序中某些一直活跃的相关单例就需要进行重置。

最简单的重置设计就是析构这些单例，然后重新创建，使其能够将创建过程中的所有逻辑重新运行一次，得到重置的效果。

这时候我们有两种选择，一种是使用常规的静态变量保护措施来保证单例，当我们需要reset时，重置这些变量，然后析构单例即可：

```objc
static id gSharedInstance;
+ (instancetype)sharedInstance
{
    if (!gSharedInstance)
    {
        gSharedInstance = [[self alloc] init];
    }
    return gSharedInstance;
}

+ (void)destorySharedInstance
{
    if (gSharedInstance)
    {
        [gSharedInstance release];
        gSharedInstance = nil;
    }
}
```

但是使用常规的静态变量方式显然没有dispatch_once那么高端大气上档次，或者说没有那么高效、简洁、线程安全，那么我们可能想要采取的方式就是重置dispatch_once的predicate。

然而，如果可重置的单例被设计为需要重新创建这些单例，则显得很生硬（尽管有时候会让人感觉很方便），从程序设计角度来讲，更优雅的方式是让这些单例具有可重置的入口，甚至可以单独地重置这些单例的子模块：

```objc
+ (instancetype)sharedInstance
{
    static id sharedInstance;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (void)resetSubmoduleA
{
}

- (void)resetSubmoduleB
{
}

- (void)reset
{
    [self resetSubmoduleA];
    [self resetSubmoduleB];
}
```

如此可以从程序设计层面避免重新创建单例，模块也变得更清晰。
按照这样的设计方式，甚至可以在内存吃紧时，释放不必要的cache或者submodule资源等。
尽管我并不提倡这么做，但本篇旨在讨论使用动态分配的predicate的可能性，所以我们还是继续下一环节的分析。

### 从代码编写角度
我们知道通常来说，一个变量在内存空间中有三种存储方式：堆、栈、全局区（这里我们略过memmap区等一些不常用或者不可用的内存区），那么动态分配，就是堆与栈变量了。
在栈上分配predicate几乎没什么意义，所以我们直接研究堆上的predicate。
我们先编写一个简单的范例：
```objc
@interface ClassA : NSObject
{
    dispatch_once_t _onceToken;
    int _bar;
}
@end
@implementation ClassA
- (instancetype)init
{
    if (self = [super init]) {
        //_onceToken = 0;
    }
    return self;
}
- (int)foo
{
    dispatch_once(&_onceToken, ^{
        _bar = 999;
    });
    return _bar;
}
@end
```

这段代码使用了堆上的predicate，当ClassA被实例化为InstanceA时，_onceToken被一起创建，但是值得注意的是，一个ObjectiveC对象被alloc时，其内存布局的剩余区域会被填充为0，也就是说，也就是说，当ClassA被创建后，其成员_onceToken的值已经为0（我们暂不讨论CPU的内存读写操作的[弱一致性](https://en.wikipedia.org/wiki/Weak_consistency)，放在后续段落中讨论）。

在上一篇中，我们已经剖析过dispatch_once的内部实现（写入端代码），我们发现dispatch_once_t实际上是一个long类型，也就是等同于一个指针类型，而dispatch_once函数内部确实是将predicate转换为一个指针来使用，并在不同的逻辑阶段用其指向不同的struct。
那么一个long类型变量是static还是dynamic有什么区别？一般来说，有两方面：

1. static long具有固定的内存地址，在整个程序的生命周期中，它的值都不会改变；而相反的，dynamic分配的long，其内存地址很有可能发生变化，比如其所处的内存块被realloc等等。
2. static long的值从程序load开始就已经初始化为0了，也就是说它“从未非零”过，而dynamic分配的long其在内存真正被置0之前，值是任意的。

对于第一种情况，我们的InstanceA可以保证在自身的生命周期中，其成员变量_onceToken不会改变，可以排除。

而对于第二种情况，一个变量曾经不为0会导致出错的情况，更多地是在讨论CPU内存读写操作的弱一致性。回顾dispatch_once源码，我们会发现，在if条件判定predicate值之前，并没有出现能够同步硬件内存读写的barrier（因为dispatch_once要保证在没有碰撞的情况下执行效率无限接近非线程安全的纯if语句）。

CPU的乱序执行是一个太过复杂的话题，为了彻底避免这一可能性带来的影响，我们可以在init中引入一个memory barrier：

```objc
@interface ClassA : NSObject
{
    dispatch_once_t _onceToken;
    int _bar;
}
@end
@implementation ClassA
- (instancetype)init
{
    if (self = [super init]) {
        //_onceToken = 0;
        OSMemoryBarrier();
    }
    return self;
}
- (int)foo
{
    dispatch_once(&_onceToken, ^{
        _bar = 999;
    });
    return _bar;
}
@end
```

如此一来，只要我们取得了InstanceA的句柄（也就是其指针），其成员_onceToken的值在被读取时，一定会先通过memory barrier的同步，从而保证读取到的一定是初始化为0后的值，如此一来，其从前的值是否为0，就没有影响了，但如此一来，我们付出的代价是内存屏障带来的性能降低，初始化InstanceA将需要更多时间。
到了这一步，那么我们可以放心的使用_onceToken这个成员变量作为predicate了？
还不够，我们知道，ClassA的实例InstanceA是一个动态分配的实例（当然，ObjectiveC并不支持静态分配实例），其生命周期是有限的，一旦InstanceA在某种的情况下被析构，_onceToken将立即成为一个无效内存，如果此时dispatch_once正处于内部逻辑的某个中间状态（参考上一篇教程对其内部逻辑各个状态的剖析），那么将发生无法评估的错乱，这应该也正是Apple明确在文档中标出不要使用ObjectiveC成员变量作为predicate的原因之一。
对于这种情况，我们要做的是，保证使用_onceToken的dispatch_once调用都处于InstanceA的生命周期之内。但是这个保证是在不同的Class中需要的做法都不同，我们并不能总结出一个一劳永逸的方法。但没关系，无论如何，我们总结和猜测出了各种可能的情况，并尽量避免之，接下来让我们使用Demo来测试。

## 使用Demo来测试
我们使用50个并发来测试:)
```objc
void testClassA()
{
    ClassA *a = [[[ClassA alloc] init] autorelease];
    void(^ bgFoo)() = ^{
        int f = [a foo];
        NSLog(@"bgFoo f:%i", f);
    };
    for (int i = 0; i < 50; i++)
    {
        dispatch_async(dispatch_get_global_queue(0, 0), bgFoo);
    };
    for (int i = 0; i < 50; i++)
    {
        int f = [a foo];
        NSLog(@"foo f:%i", f);
    }
}
```

在我的MacbookAir上，其打印结果为：
```
...
2015-06-28 21:41:46.464 testgcd[4623:663776] bgFoo f:999
2015-06-28 21:41:46.466 testgcd[4623:663653] foo f:999
2015-06-28 21:41:46.466 testgcd[4623:663653] foo f:999
2015-06-28 21:41:46.465 testgcd[4623:663795] bgFoo f:999
2015-06-28 21:41:46.466 testgcd[4623:663653] foo f:999
2015-06-28 21:41:46.466 testgcd[4623:663795] bgFoo f:999
...
```

额，简直太长了，我只截取了一部分log，我检查了所有100条log，每一个返回值都是999。

## 可靠吗？
对于这种使用方式的可靠性，我们不能保证，但是可以肯定的一点是，越能保证避免前文中猜测的问题，可靠性越高。但是这种使用方式应该极力避免。或者可以在你项目的“实验室版”中加以使用，并结合错误日志来长期验证？

在后面的教程中，我会和大家一起研究重置predicate使得dispatch_once重新执行的方法。
