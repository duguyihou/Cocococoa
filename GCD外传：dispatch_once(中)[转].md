# [转]GCD外传：dispatch_once(中)

本篇，我将和大家一起探究`dispatch_once`写入端的实现
让我们先看看`dispatch_once`的实现（Grand Central Dispatch是开源的，大家可以到git://git.macosforge.org/libdispatch.git克隆源码）
```objc
struct _dispatch_once_waiter_s {
    volatile struct _dispatch_once_waiter_s *volatile dow_next;
    _dispatch_thread_semaphore_t dow_sema;
};

#define DISPATCH_ONCE_DONE ((struct _dispatch_once_waiter_s *)~0l)

#ifdef __BLOCKS__
void
dispatch_once(dispatch_once_t *val, dispatch_block_t block)
{
    struct Block_basic *bb = (void *)block;

    dispatch_once_f(val, block, (void *)bb->Block_invoke);
}
#endif

DISPATCH_NOINLINE
void
dispatch_once_f(dispatch_once_t *val, void *ctxt, dispatch_function_t func)
{
    struct _dispatch_once_waiter_s * volatile *vval = (struct _dispatch_once_waiter_s**)val;
    struct _dispatch_once_waiter_s dow = { NULL, 0 };
    struct _dispatch_once_waiter_s *tail, *tmp;
    _dispatch_thread_semaphore_t sema;

    if (dispatch_atomic_cmpxchg(vval, NULL, &dow)) {
        dispatch_atomic_acquire_barrier();//这是一个空的宏函数，什么也不做
        _dispatch_client_callout(ctxt, func);
        dispatch_atomic_maximally_synchronizing_barrier();
        //dispatch_atomic_release_barrier(); // assumed contained in above
        tmp = dispatch_atomic_xchg(vval, DISPATCH_ONCE_DONE);
        tail = &dow;
        while (tail != tmp) {
            while (!tmp->dow_next) {
                _dispatch_hardware_pause();
            }
            sema = tmp->dow_sema;
            tmp = (struct _dispatch_once_waiter_s*)tmp->dow_next;
            _dispatch_thread_semaphore_signal(sema);
        }
    } else {
        dow.dow_sema = _dispatch_get_thread_semaphore();
        for (;;) {
            tmp = *vval;
            if (tmp == DISPATCH_ONCE_DONE) {
                break;
            }
            dispatch_atomic_store_barrier();
            if (dispatch_atomic_cmpxchg(vval, tmp, &dow)) {
                dow.dow_next = tmp;
                _dispatch_thread_semaphore_wait(dow.dow_sema);
            }
        }
        _dispatch_put_thread_semaphore(dow.dow_sema);
    }
}
```

一堆宏函数加一堆让人头大的线程同步代码。一步一步看：
`dispatch_once`内部其实是调用了`dispatch_once_f`，f指的是调用c函数（没有f指的是调用block），实际上执行block最终也是调用c函数（详见我的《[Block非官方编程指南](http://www.dreamingwish.com/article/block教程系列.html)》）。当`dispatch_once_f`被调用时，`val`是外部传入的`predicate`，`ctxt`传入的是Block的指针，`func`指的是Block内部的执行体函数，执行它就是执行block。
接下来是声明了一堆变量，`vval`是volatile标记过的`val`，volatile修饰符的作用上一篇已经介绍过，告诉编译器此指针指向的值随时可能被其他线程改变，从而使得编译器不对此指针进行代码编译优化。

>dow意为dispatch_once wait


`dispatch_atomic_cmpxchg`是上一篇我们讲过的“原子比较交换函数”`__sync_bool_compare_and_swap`的宏替换，接下来进入分支：

##  执行block的分支
当`dispatch_once`第一次执行时，`predicate`也即val为0，那么此“原子比较交换函数”将返回true并将`vval`指向值赋值为`&dow`，即为“等待中”，`_dispatch_client_callout`其内部做了一些判定，但实际上是调用了`func`而已。到此，block中的用户代码执行完毕。

接下来就是上篇提及的`cpuid`指令等待，使得其他线程的【读取到未初始化值的】预执行能被判定为猜测未命中，从而使得这些线程能够进入`dispatch_once_f`里的另一个分支从而进行等待。

`cpuid`指令完毕后，调用`dispatch_atomic_xchg`进行赋值，置其为DISPATCH_ONCE_DONE，即“完成”，这里`dispatch_atomic_xchg`是内建“原子交换函数”`__sync_swap`的优化版宏替换，其将第二个参数的值赋给第一个参数（解引用指针），然后返回第一个参数被赋值前的解引用值，其原型为：

```objc
type __sync_swap(type *ptr, type value, ...)
```

接下来是对信号量链的处理：

1. 在block执行过程中，没有其他线程进入本函数来等待，则vval指向值保持为&dow，即tmp被赋值为&dow，即下方while循环不会被执行，此分支结束。
2. 在block执行过程中，有其他线程进入本函数来等待，那么会构造一个信号量链表（vval指向值变为信号量链的头部，链表的尾部为&dow），此时就会进入while循环，在此while循环中，遍历链表，逐个signal每个信号量，然后结束循环。

>`while (!tmp->dow_next)`此循环是等待在&dow上，因为线程等待分支#2会中途将`val`赋值为`&dow`，然后为`->dow_next`赋值，这期间`->dow_next`值为NULL，需要等待，详见下面线程等待分支#2的描述


>`_dispatch_hardware_pause`此句是为了提示cpu减少额外处理，提升性能，节省电力。

## 线程等待分支

当**执行block分支**#1未完成，且有线程再进入本函数时，将进入线程等待分支：
先调用`_dispatch_get_thread_semaphore`创建一个信号量，此信号量被赋值给`dow.dow_sema`。
然后进入一个无限for循环，假如发现`vval`的指向值已经为`DISPATCH_ONCE_DONE`，即“完成”，则直接break，然后调用`_dispatch_put_thread_semaphore`**函数销毁信号量并退出函数**。

>`_dispatch_get_thread_semaphore`内部使用的是“有即取用，无即创建”策略来获取信号量。
`_dispatch_put_thread_semaphore`内部使用的是“销毁旧的，存储新的”策略来缓存信号量。

假如`vval`的解引用值并非`DISPATCH_ONCE_DONE`，则进行一个“原子比较并交换”操作（此操作可以避免两个等待线程同时操作链表带来的问题），假如此时`vval`指向值已不再是`tmp`（这种情况发生在多个线程同时进入线程等待分支#2，并交错修改链表）则for循环重新开始，再尝试重新获取一次`vval`来进行同样的操作；若指向值还是tmp，则将`vval`的指向值赋值为`&dow`，此时`val->dow_next`值为NULL，可能会使得block执行分支#1进行while等待（如前述），紧接着执行`dow.dow_next = tmp`这句来增加链表节点（同时也使得block执行分支#1的while等待结束），然后等待在信号量上，当block执行分支#1完成并遍历链表来signal时，唤醒、释放信号量，然后一切就完成了。

## 小结

综上所述，dispatch_once的主要处理的情况如下：

- 线程A执行Block时，任何其它线程都需要等待。
- 线程A执行完Block应该立即标记任务完成状态，然后遍历信号量链来唤醒所有等待线程。
- 线程A遍历信号量链来signal时，任何其他新进入函数的线程都应该直接返回而无需等待。
- 线程A遍历信号量链来signal时，若有其它等待线程B仍在更新或试图更新信号量链，应该保证此线程B能正确完成其任务：a.直接返回 b.等待在信号量上并很快又被唤醒。
- 线程B构造信号量时，应该考虑线程A随时可能改变状态（“等待”、“完成”、“遍历信号量链”）。
- 线程B构造信号量时，应该考虑到另一个线程C也可能正在更新或试图更新信号量链，应该保证B、C都能正常完成其任务：a.增加链节并等待在信号量上 b.发现线程A已经标记“完成”然后直接销毁信号量并退出函数。

>总结: 无锁的线程同步编程非常精巧，为了提升效率，每一处线程竞争都必须被考虑到并妥善处理。但这种编程方式又极其令人神往，原子操作的魅力便在于此，它就像是一个精密的钟表，每一处接合都如此巧妙。
下一篇，我们将一起研究使用非static的predicate的可能性。
