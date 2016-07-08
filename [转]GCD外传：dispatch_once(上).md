# [转]GCD外传：dispatch_once(上)
相信大家对`dispatch_once`都不陌生了，这一篇我将和大家一起探究dispatch_once的更多细节。

`dispatch_once`的作用正如其名：对于某个任务执行一次，且只执行一次。`dispatch_once`函数有两个参数，第一个参数`predicate`用来保证执行一次，第二个参数是要执行一次的任务block。

```objc
static dispatch_once_t predicate;
dispatch_once(&predicate, ^{
    // some one-time task
});
```
`dispatch_once`被广泛使用在**单例、缓存**等代码中，用以保证在初始化时执行一次某任务。
`dispatch_once`在单线程程序中毫无意义，但在多线程程序中，其**低负载、高可依赖性、接口简单**等特性，赢得了广大消费者的一致五星好评。

## 本系列中我将和大家一起一步步分析dispatch_once的低负载特性。
要讨论`dispatch_once`的低负载性，我们要讨论三种场景：
1. 第一次执行，block需要被调用，调用结束后需要置标记变量
2. 非第一次执行，而此时#1尚未完成，线程需要等待#1完成
3. 非第一次执行，而此时#1已经完成，线程直接跳过block而进行后续任务

对于场景#1，整体任务的效率瓶颈完全不在于`dispatch_once`，而在于block本身占用的cpu时间，并且也只会发生一次。

对于场景#2，发生的次数并不会很多，甚至很多时候一次都不会发生，假如发生了，那么也只是一个符合预期的行为：后来的线程需要等待第一线程完成。即使你写一个受虐型的单元测试来故意模拟场景#2，也不能说明什么问题，得不到的永远在骚动，被偏爱的都有恃无恐。

对于场景#3，在程序进行过程中，可能发生成千上万次或者天文数字次，这才是效率提升的关键之处，下面我将细细道来。

## 需求的初衷
`dispatch_once`本来是被用作第一次的执行保护，等第一次执行完毕之后，其职责就完成了，作为程序设计者，当然希望它对后续执行没有任何影响，但这是做不到的，所以只能寄希望于尽量降低后续调用的负载。
负载的Benchmark
对于后续调用的负载，到底要降低到什么程度，需要一个基准值，负荷最低的空白对照就是非线程安全的纯if判断语句了，在我的电脑上，一次包含if判断语句的函数单例返回大概在0.5纳秒左右，而`dispatch_once`确实做到了接近这个数值，有兴趣可以亲自写一段测试代码来试试。
```objc
//dispatch_once
static id object;
static dispatch_once_t predicate;
dispatch_once(&predicate, ^{
    object = ...;
});
return object;
//if判断
static id object = nil;
if (!object)
{
    object = ...;
}
return object;
```
## 负载的探究：重实现dispatch_once
### 线程锁
使用`pthread_mutex_lock`是我首先想到的实现方式：
```objc
void DWDispatchOnce(dispatch_once_t *predicate, dispatch_block_t block) {
    static pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    pthread_mutex_lock(&mutex);
    if(!*predicate) {
        block();
        *predicate = 1;
    }
    pthread_mutex_unlock(&mutex);
}
```
这样的实现确实是线程安全的，但是`pthread_mutex_lock`的效率太低了，后续调用负载是两次锁操作（加锁解锁），在我的macbookpro上，这个函数需要30ns，这战斗力太渣了，抛弃。

### 自旋锁
自旋锁比之互斥锁，其优势在于某些情况下负载更低，然后，我来改一下我的函数实现：
```objc
void DWDispatchOnce(dispatch_once_t *predicate, dispatch_block_t block) {
    static OSSpinLock lock = OS_SPINLOCK_INIT;
    OSSpinLockLock(&lock);
    if(!*predicate) {
        block();
        *predicate = 1;
    }
    OSSpinLockUnlock(&lock);
}
```
嗯，提升很不错，这次提升到了6.5ns，自旋锁在低碰撞情况下，效率果然不是盖的，不过对于`dispatch_once`来说，还是太渣了，6ns实在是太龟速了。

### 原子操作
原子操作是低级CPU操作，不用锁也是线程安全的（实际上，原子操作使用硬件级别的锁），原子操作使得自己实现软件级别锁成为可能。当锁负载太高时，可以直接使用原子操作来替代锁。

>以原子操作来替代锁的编程方式很取巧，比较容易出现问题。bug很难找，使用需谨慎。


我们使用“原子比较交换函数” `__sync_bool_compare_and_swap`来实现新的`DWDispatchOnce，__sync_bool_compare_and_swap`的作用大概等同于：
```objc
BOOL DWCompareAndSwap(long *ptr, long testValue, long newValue) {
    if(*ptr == testValue) {
        *ptr = newValue;
        return YES;
    }
    return NO;
}
```

不同的是，`__sync_bool_compare_and_swap`是一个被实现为cpu原子操作的函数，所以比较和交换操作是一个整体操作并且是线程安全的。

所以新的实现就成为：

```objc
void DWDispatchOnce(dispatch_once_t *predicate, dispatch_block_t block)
{
    if(*predicate == 2)
    {
        __sync_synchronize();
        return;
    }

    volatile dispatch_once_t *volatilePredicate = predicate;

    if(__sync_bool_compare_and_swap(volatilePredicate, 0, 1)) {
        block();
        __sync_synchronize();
        *volatilePredicate = 2;
    } else {
        while(*volatilePredicate != 2)
            ;//注意这里没有循环体
        __sync_synchronize();
    }
}
```

新的实现首先检查`predicate`是否为2，假如为2，则调用`__sync_synchronize`这个builtin函数并返回，调用此函数会产生一个memory barrier，用以保证cpu读写顺序严格按照程序的编写顺序来进行，关于memory barrier的更多信息，还是查[wiki](https://en.wikipedia.org/wiki/Memory_barrier)吧。

紧接着是一个`volatile`修饰符修饰的指针临时变量，如此编译器就会假定此指针指向的值可能会随时被其它线程改变，从而防止编译器对此指针指向的值的读写进行优化，比如cache，reorder等。

然后进行“原子比较交换”，如果`predicate`为0，则将`predicate`置为1，表示正在执行block，并返回true，如此便进入了block执行分支，在block执行完毕之后，我们依旧需要一个memory barrier，最后我们将`predicate`置为2，表示执行已经完成，后续调用应该直接返回。

当某个线程A正在执行block时，任何线程B再进入此函数，便会进入else分支，然后在此分支中进行等待，直至线程A将predicate置为2，然后调用`__sync_synchronize`并返回

这个实现是线程安全的，并且是无锁的，但是，依旧需要消耗11.5ns来执行，比自旋锁都慢，实际上memory barrier是很慢的。至于为什么比自旋锁还慢，memory barrier有好几种，`__sync_synchronize`产生的是mfenceCPU指令，是最蛋疼的一种，跟那蛋疼到忧伤的SSE4指令集是一路货。但不管怎么样，我想说的是，memory barrier是有不小的开销的。

假如去除掉memory barrier会如何呢？
```objc
void DWDispatchOnce(dispatch_once_t *predicate, dispatch_block_t block) {
    if(*predicate == 2)
        return;

    volatile dispatch_once_t *volatilePredicate = predicate;

    if(__sync_bool_compare_and_swap(volatilePredicate, 0, 1)) {
        block();
        *volatilePredicate = 2;
    } else {
        while(*volatilePredicate != 2)
            ;
        }
}
```

这其实是一个不线程安全的实现，现代的CPU都是异步的，为了满足用户“又想马儿好，又想马儿不吃草”的奢望，CPU厂商堪称无所不用其极，所以现代的CPU在提升速度上有很多优化，其中之一就是流水线特性，当执行一条cpu指令时，发生了如下事情：
```objc
1.从内存加载指令
2.指令解码（解析指令是什么，操作是什么）
3.加载输入数据
4.执行操作
5.保存输出数据
```

在古董级cpu上面，cpu是这样干活的：
```objc
加载指令
解码
加载数据
执行
保存输出数据
加载指令
解码
加载数据
执行
保存输出数据
加载指令
解码
加载数据
执行
保存输出数据
...
```
在现代CPU上，cpu是这样干活的：
```objc
加载指令                     ...
解码            加载指令
加载数据         解码
执行            加载数据
保存输出数据     执行
               保存输出数据
...
```

这可就快得多了，cpu会将其认为可以同时执行的指令并行执行，并根据优化速度的需要来调整执行顺序，比如：
```objc
x = 1;
y = 2;
```

cpu可能会先执行y=2，另外，编译器也可能为了优化而为你生成一个先执行y=2的代码，即使关闭编译器优化，cpu还是可能会先执行y=2，在多核处理器中，其它的cpu就会看到这个两个赋值操作顺序颠倒了，即使赋值操作没有颠倒，其它cpu也可能颠倒读取顺序，最后导致的结果可能是另一个线程在读取到y为2时，却发现x还没被赋值为1。
解决这种问题的方法就是加入memory barrier，但是memory barrier的目的就在于防止cpu“跑太快”，所以，开销的惩罚那是大大的。
所以，对于`dispatch_once`：
```objc
static SomeClass *obj;
static dispatch_once_t predicate;
dispatch_once(&predicate, ^{ obj = [[SomeClass alloc] init]; });
[obj doSomething];
```

假如`obj`在`predicate`之前被读取，那一个线程可能另一个线程执行完block之前就取得了一个nil值；假如`predicate`被读取为“已完成”，并且此时另一个线程正在初始化这个`obj`，那么接下来调用函数可能会导致程序崩溃。

所以，`dispatch_once`需要memory barrier或者类似的东西，但是它肯定没有使用memory barrier，因为memory barrier实在是很慢。要明白`dispatch_once`如何避免memory barrier，先要了解cpu的分支预测和预执行。

### cpu的分支预测和预执行
流水线特性使得CPU能更快地执行线性指令序列，但是当遇到条件判断分支时，麻烦来了，在判定语句返回结果之前，cpu不知道该执行哪个分支，那就得等着（术语叫做pipeline stall），这怎么能行呢，所以，CPU会进行预执行，cpu先猜测一个可能的分支，然后开始执行分支中的指令。现代CPU一般都能做到超过90%的猜测命中率，这可比NBA选手发球命中率高多了。然后当判定语句返回，加入cpu猜错分支，那么之前进行的执行都会被抛弃，然后从正确的分支重新开始执行。

在`dispatch_once`中，唯一一个判断分支就是`predicate`，`dispatch_once`会让CPU预执行条件不成立的分支，这样可以大大提升函数执行速度。但是这样的预执行导致的结果是使用了未初始化的`obj`并将函数返回，这显然不是预期结果。

### 不对称barrier
编写barrier时，应该是对称的，在写入端，要有一个barrier来保证顺序写入，同时，在读取端，也要有一个barrier来保证顺序读取。但是，我们的`dispatch_once`实现要求写入端快不快无所谓，而读取端尽可能的快。所以，我们要解决前述的预执行引起的问题。

当一个预执行最终被发现是错误的猜测时，所有的预执行状态以及结果都会被清除，然后cpu会从判断分支处重新执行正确的分支，也就意味着被读取的未初始化的`obj`也会被抛弃，然后读取。假如`dispatch_once`能做到在执行完block并正确赋值给`obj`后，告诉其它cpu核心：你们这群无知的cpu啊，你们刚才都猜错了！然后这群“无知的cpu”就会重新从分支处开始执行，进而获取正确的`obj`值并返回。

从最早的预执行到条件判断语句最终结果被计算出来，这之间有很长时间（记作Ta），具体多长取决于cpu的设计，但是不论如何，这个时间最多几十圈cpu时钟时间，假如写入端能在【初始化并写入`obj`】与【置`predicate`值】之间等待足够长的时间Tb使得Tb大于等于Ta，那问题就都解决了。

> 如果觉得这个”解决”难以理解，那么反过来思考，假如Tb小于Ta，那么Tb就有可能被Ta完全包含，也就是说，另一个线程B（耗时为Ta）在预执行读取了未初始化的`obj`值之后，回过头来确认猜测正确性时，`predicate`可能被执行block的线程A置为了“完成”，这就导致线程B认为自己的预执行有效（实际上它读取了未初始化的值）。而假如Tb大于等于Ta，任何读取了未初始化的`obj`值的预执行都会被判定为未命中，从而进入内层`dispatch_once`而进行等待。

要保证足够的等待时间，需要一些trick。在intel的CPU上，`dispatch_once`动用了`cpuid`指令来达成这个目的。`cpuid`本来是用作取得cpu的信息，但是这个指令也同时强制将指令流串行化，并且这个指令是需要比较长的执行时间的（在某些cpu上，甚至需要几百圈cpu时钟），这个时间Tc足够超过Ta了。

查看`dispatch_once`读取端的实现：

```objc
DISPATCH_INLINE DISPATCH_ALWAYS_INLINE DISPATCH_NONNULL_ALL DISPATCH_NOTHROW
void
_dispatch_once(dispatch_once_t *predicate, dispatch_block_t block)
{
    if (DISPATCH_EXPECT(*predicate, ~0l) != ~0l) {
        dispatch_once(predicate, block);
    }
}
#define dispatch_once _dispatch_once
```

没有barrier，并且这个代码是在头文件中的，是强制inline的，`DISPATCH_EXPECT`是用来告诉cpu `*predicate`等于`~0l`是更有可能的判定结果，这就使得cpu能猜测到更正确的分支，并提高效率，最重要的是，这一句是个简单的if判定语句，负载无限接近benchmark。
在写入端，`dispatch_once`在执行了block之后，会调用`dispatch_atomic_maximally_synchronizing_barrier()`;宏函数，在intel处理器上，这个函数编译出的是`cpuid`指令，在其他厂商处理器上，这个宏函数编译出的是合适的其它指令。

如此一来，`dispatch_once`就保证了场景#3的执行速度无限接近benchmark，实现了写入端的最低负载。

下一篇，我将和大家一起探究`dispatch_once`写入端的实现，并讨论使用非static predicate的可能性
