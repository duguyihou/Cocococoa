# GCD
参考资料：
[Grand Central Dispatch (GCD) Reference](https://developer.apple.com/library/mac/documentation/Performance/Reference/GCD_libdispatch_Ref/index.html#//apple_ref/c/func/dispatch_set_finalizer_f)
[Concurrency Programming Guide](https://developer.apple.com/library/mac/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091)
> 直观上理解，GCD偏向于系统级的API，也就是说它更接近于底层，在编写规范的前提下它相较NSOperation的性能要略优。而Cocoa的异步框架即NSOperation相关提供的API，更偏向于应用层面，它是对系统底层调用（包括GCD等）的封装，从功能层面上讲相较GCD更为丰富（NSOperation + Queue的形式具备一些GCD未直接包含的功能）
通过查阅官方文档以及国外一些Blog的阐述，基本达成的共识是：在APP中，尽可能的使用Cocoa，即high-level Api，除非在实际性能测试数据上发现不得不用更底层的api的时候，才进一步考虑使用GCD。

引用两段话：
```
GCD is not restricted to system-level applications, but before you use it for higher-level applications, you should consider whether similar functionality provided in Cocoa (via NSOperation and block objects) would be easier to use or more appropriate for your needs. See Concurrency Programming Guide for more information.
Always use the highest-level abstraction available to you, and drop down to lower-level abstractions when measurement shows that they are needed.
GCD – 1
```
## 三种多线程技术
1. NSThread 每个NSThread对象对应一个线程，量级较轻（真正的多线程）
2. 以下两点是苹果专门开发的“并发”技术，使得程序员可以不再去关心线程的具体使用问题
* NSOperation/NSOperationQueue 面向对象的线程技术
* GCD —— Grand Central Dispatch（派发） 是基于C语言的框架，可以充分利用多核，是苹果推荐使用的多线程技术
>以上这三种编程方式从上到下，抽象度层次是从低到高的，抽象度越高的使用越简单，也是Apple最推荐使用的，在项目中很多框架技术分别使用了不同多线程技术。

## 三种多线程技术的对比
* NSThread:
>– 优点：NSThread 比其他两个轻量级，使用简单
– 缺点：需要自己管理线程的生命周期、线程同步、加锁、睡眠以及唤醒等。线程同步对数据的加锁会有一定的系统开销
* NSOperation：
>– 不需要关心线程管理，数据同步的事情，可以把精力放在自己需要执行的操作上
– NSOperation是面向对象的
* GCD：
>– Grand Central Dispatch是由苹果开发的一个多核编程的解决方案。iOS4.0+才能使用，是替代NSThread， NSOperation的高效和强大的技术
– GCD是基于C语言的

GCD从语言、运行时库、系统扩展等三个方面给让使用者更充分的操作多核设备，同时它基于队列的概念，因为每一个CPU core单位时间（时间片）内只能运行某个队列的某个task，并通过优先级、FIFO等策略进行task的切换运行。

## GCD超越传统多线程编程的优势：
1. **易用**: GCD比之thread更简单易用。由于GCD基于work unit而非像thread那样基于运算，所以GCD可以控制诸如**等待任务结束、监视文件描述符、周期执行代码以及工作挂起**等任务。基于block的血统导致它能极为简单得在不同代码作用域之间传递上下文。
2. **效率**: GCD被实现得如此轻量和优雅，使得它在很多地方比之专门创建消耗资源的线程更实用且快速。这关系到易用性：导致GCD易用的原因有一部分在于你可以不用担心太多的效率问题而仅仅使用它就行了。
3. **性能**: GCD自动根据系统负载来增减线程数量，这就减少了上下文切换以及增加了计算效率。

## Dispatch Objects
尽管GCD是纯c语言的，但它被组建成面向对象的风格。GCD对象被称为dispatch object。Dispatch object像Cocoa对象一样是引用计数的。使用dispatch_release和dispatch_retain函数来操作dispatch object的引用计数来进行内存管理。但主意不像Cocoa对象，dispatch object并不参与垃圾回收系统，所以即使开启了GC，你也必须手动管理GCD对象的内存。

Dispatch queues 和 dispatch sources（后面会介绍到）可以被挂起和恢复，可以有一个相关联的任意上下文指针，可以有一个相关联的任务完成触发函数。可以查阅“man dispatch_object”来获取这些功能的更多信息

## Dispatch Queues
GCD的基本概念就是dispatch queue。dispatch queue是一个对象，它可以接受任务，并将任务以先到先执行的顺序来执行。dispatch queue可以是并发的或串行的。并发任务会像NSOperationQueue那样基于系统负载来合适地并发进行，串行队列同一时间只执行单一任务。

GCD共提供三种队列形式：

1. The main queue：与主线程功能相同，iOS的UI绘制、交互响应都要在此线程上执行。实际上，提交至main queue的任务会在主线程中执行。main queue可以调用`dispatch_get_main_queue()`来获得。因为main queue是与主线程相关的，所以这是一个串行队列。

2. Global queues: 全局队列是并发队列，并由整个进程共享。进程中存在三个全局队列：高、中（默认）、低三个优先级队列。可以调用`dispatch_get_global_queue`函数传入优先级来访问队列。task的进出队列，都遵循FIFO策略，但队列中的task‘看起来’是并发执行的完成时间却可能是以任意的顺序结束。很容易理解，因为每个task的执行时间长短通常不一样。

3. 用户队列: 用函数 `dispatch_queue_create` 创建的队列. 这些队列是串行的。正因为如此，它们可以用来完成同步机制, 有点像传统线程中的mutex。

要执行一个block（task），前提是需要知道我们要将task放入哪个或哪类（上述其中之一）queue？

## 创建队列
要使用用户队列，首先要创建一个。需要调用函数`dispatch_queue_create`。函数的第一个参数是一个标签，这纯是为了debug。Apple建议使用倒置域名来命名队列，比如“com.example.myqueue”。这些名字会在崩溃日志中被显示出来，也可以被调试器调用，这在调试中会很有用。第二个参数目前还不支持，传入NULL就行了。
## 提交Job
向一个队列提交Job很简单：调用`dispatch_async`函数，传入一个队列和一个block。队列会在轮到这个block执行时执行这个block的代码。下面的例子是一个在后台执行一个巨长的任务：
```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self goDoSomethingLongAndInvolved];
        NSLog(@"Done doing something long and involved");
});
```

`dispatch_async` 函数会立即返回, block会在后台异步执行。
若在任务完成时更新界面，就需要在主线程中执行。需要使用嵌套的`dispatch`，在外层中执行后台任务，在内层中将任务dispatch到`main queue`：
```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self goDoSomethingLongAndInvolved];
        dispatch_async(dispatch_get_main_queue(), ^{
            [textField setStringValue:@"Done doing something long and involved"];
        });
});
```

`dispatch_sync`函数和`dispatch_async`相同，但是会等待block中的代码执行完成并返回。结合 `__block`类型修饰符，可以用来从执行中的block获取一个值。
例如，你可能有一段代码在后台执行，而它需要从界面控制层获取一个值。那么你可以使用`dispatch_sync`简单办到：
```objc
__block NSString *stringValue;
dispatch_sync(dispatch_get_main_queue(), ^{
        // __block variables aren't automatically retained
        // so we'd better make sure we have a reference we can keep
        stringValue = [[textField stringValue] copy];
});
[stringValue autorelease];
// use stringValue in the background now
```
更好的做法,使用更“异步”的风格。不同于取界面层的值时要阻塞后台线程，使用嵌套的block来中止后台线程，然后从主线程中获取值，然后再将后期处理提交至后台线程：
```
dispatch_queue_t bgQueue = myQueue;
    dispatch_async(dispatch_get_main_queue(), ^{
        NSString *stringValue = [[[textField stringValue] copy] autorelease];
        dispatch_async(bgQueue, ^{
            // use stringValue in the background now
        });
    });
```
取决于你的需求，myQueue可以是用户队列也可以使全局队列。

## 不再使用锁（Lock）
用户队列可以用于替代锁来完成同步机制。在传统多线程编程中，你可能有一个对象要被多个线程使用，你需要一个锁来保护这个对象：
```objc
 NSLock *lock;
```

```objc
- (id)something
{
    id localSomething;
    [lock lock];
    localSomething = [[something retain] autorelease];
    [lock unlock];
    return localSomething;
}

- (void)setSomething:(id)newSomething
{
    [lock lock];
    if(newSomething != something)
    {
        [something release];
        something = [newSomething retain];
        [self updateSomethingCaches];
    }
    [lock unlock];
}
```
使用GCD，可以使用queue来替代：
```objc
 dispatch_queue_t queue;
```
要用于同步机制，queue必须是一个用户队列(从OS X v10.7和iOS 4.3开始，还必须指定为DISPATCH_QUEUE_SERIAL)，而非全局队列，所以使用`dispatch_queue_create`初始化一个。然后可以用`dispatch_async` 或者 `dispatch_sync`将共享数据的访问代码封装起来：
```
- (id)something
{
    __block id localSomething;
    dispatch_sync(queue, ^{
        localSomething = [something retain];
    });
    return [localSomething autorelease];
}

- (void)setSomething:(id)newSomething
{
    dispatch_async(queue, ^{
        if(newSomething != something)
        {
            [something release];
            something = [newSomething retain];
            [self updateSomethingCaches];
        }
    });
}
```
使用GCD途径有几个好处：
1. 平行计算: 注意在第二个版本的代码中， -setSomething:是怎么使用dispatch_async的。调用 -setSomething:会立即返回，然后这一大堆工作会在后台执行。如果updateSomethingCaches是一个很费时费力的任务，且调用者将要进行一项处理器高负荷任务，那么这样做会很棒。
2. 安全: 使用GCD，我们就不可能意外写出具有不成对Lock的代码。在常规Lock代码中，我们很可能在解锁之前让代码返回了。使用GCD，队列通常持续运行，你必将归还控制权。
3. 控制: 使用GCD我们可以挂起和恢复dispatch queue，而这是基于锁的方法所不能实现的。我们还可以将一个用户队列指向另一个dspatch queue，使得这个用户队列继承那个dispatch queue的属性。使用这种方法，队列的优先级可以被调整——通过将该队列指向一个不同的全局队列，若有必要的话，这个队列甚至可以被用来在主线程上执行代码。
4. 集成: GCD的事件系统与dispatch queue相集成。对象需要使用的任何事件或者计时器都可以从该对象的队列中指向，使得这些句柄可以自动在该队列上执行，从而使得句柄可以与对象自动同步。

> 现在你已经知道了GCD的基本概念、怎样创建dispatch queue、怎样提交Job至dispatch queue以及怎样将队列用作线程同步。接下来我会向你展示如何使用GCD来编写平行执行代码来充分利用多核系统的性能^ ^。我还会讨论GCD更深层的东西，包括事件系统和queue targeting。

原文链接:http://www.dreamingwish.com/frontui/article/default/gcd介绍（一）-基本概念和dispatch-queue.html
