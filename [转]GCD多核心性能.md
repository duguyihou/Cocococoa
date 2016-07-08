# [转]GCD多核心性能
## 概念
为了在单一进程中充分发挥多核的优势，我们有必要使用多线程技术（我们没必要去提多进程，这玩意儿和GCD没关系）。在低层，GCD全局dispatch queue仅仅是工作线程池的抽象。这些队列中的Block一旦可用，就会被dispatch到工作线程中。提交至用户队列的Block最终也会通过全局队列进入相同的工作线程池（除非你的用户队列的目标是主线程，但是为了提高运行速度，我们绝不会这么干）。

有两种途径来通过GCD“榨取”多核心系统的性能：
* 将单一任务或者一组相关任务并发至全局队列中运算；
* 将多个不相关的任务或者关联不紧密的任务并发至用户队列中运算；

## 全局队列
设想下面的循环：
```objc
for(id obj in array)
    [self doSomethingIntensiveWith:obj];
```
假定`-doSomethingIntensiveWith:`是线程安全的且可以同时执行多个.一个array通常包含多个元素，这样的话，我们可以很简单地使用GCD来平行运算：
```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
for(id obj in array)
    dispatch_async(queue, ^{
        [self doSomethingIntensiveWith:obj];
    });
```

如此简单，我们已经在多核心上运行这段代码了。
当然这段代码并不完美。有时候我们有一段代码要像这样操作一个数组，但是在操作完成后，我们还需要对操作结果进行其他操作：
```objc
for(id obj in array)
    [self doSomethingIntensiveWith:obj];
[self doSomethingWith:array];
```
这时候使用GCD的 `dispatch_async` 就悲剧了.我们还不能简单地使用`dispatch_sync`来解决这个问题, 因为这将导致每个迭代器阻塞，就完全破坏了平行计算。

解决这个问题的一种方法是使用`dispatch group`。一个`dispatch group`可以用来将多个block组成一组以监测这些Block全部完成或者等待全部完成时发出的消息。使用函数`dispatch_group_create`来创建，然后使用函数`dispatch_group_async`来将block提交至一个`dispatch queue`，同时将它们添加至一个组。所以我们现在可以重新编码：
```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
for(id obj in array)
    dispatch_group_async(group, queue, ^{
        [self doSomethingIntensiveWith:obj];
    });
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
dispatch_release(group);

[self doSomethingWith:array];
```
如果这些工作可以异步执行，那么我们可以更风骚一点，将函数`-doSomethingWith:`放在后台执行。我们使用`dispatch_group_async`函数建立一个block在组完成后执行：
```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
for(id obj in array)
    dispatch_group_async(group, queue, ^{
        [self doSomethingIntensiveWith:obj];
    });
dispatch_group_notify(group, queue, ^{
    [self doSomethingWith:array];
});
dispatch_release(group);
```
不仅所有数组元素都会被平行操作，后续的操作也会异步执行，并且这些异步运算都会将程序的其他部分的负载考虑在内。注意如果`-doSomethingWith:`需要在主线程中执行，比如操作GUI，那么我们只要将`main queue`而非全局队列传给`dispatch_group_notify`函数就行了。

对于同步执行，GCD提供了一个简化方法叫做`dispatch_apply`。这个函数调用单一block多次，并平行运算，然后等待所有运算结束，就像我们想要的那样：
```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_apply([array count], queue, ^(size_t index){
        [self doSomethingIntensiveWith:[array objectAtIndex:index]];
    });
    [self doSomethingWith:array];
```

这很棒，但是异步咋办？`dispatch_apply`函数可是没有异步版本的。但是我们使用的可是一个为异步而生的API啊！所以我们只要用`dispatch_async`函数将所有代码推到后台就行了：
```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_async(queue, ^{
    dispatch_apply([array count], queue, ^(size_t index){
        [self doSomethingIntensiveWith:[array objectAtIndex:index]];
    });
    [self doSomethingWith:array];
});
```
这种方法的关键在于确定我们的代码是在一次对不同的数据片段进行相似的操作。如果你确定你的任务是线程安全的（不在本篇讨论范围内）那么你可以使用GCD来重写你的循环了，更平行更风骚。

要看到性能提升，你还得进行一大堆工作。比之线程，GCD是轻量和低负载的，但是将block提交至queue还是很消耗资源的——block需要被拷贝和入队，同时适当的工作线程需要被通知。不要将一张图片的每个像素作为一个block提交至队列，GCD的优点就半途夭折了。如果你不确定，那么请进行试验。将程序平行计算化是一种优化措施，在修改代码之前你必须再三思索，确定修改是有益的（还有确保你修改了正确的地方）。

## Subsystem并发运算
前面的章节我们讨论了在程序的单个subsystem中发挥多核心的优势。下来我们要跨越多个子系统。

例如，设想一个程序要打开一个包含meta信息的文档。文档数据本身需要解析并转换至模型对象来显示，meta信息也需要解析和转换。但是，文档数据和meta信息不需要交互。我们可以为文档和meta各创建一个dispatch queue，然后并发执行。文档和meta的解析代码都会各自串行执行，从而不用考虑线程安全（只要没有文档和meta之间共享的数据），但是它们还是并发执行的。

一旦文档打开了，程序需要响应用户操作。例如，可能需要进行拼写检查、代码高亮、字数统计、自动保存或者其他什么。如果每个任务都被实现为在不同的dispatch queue中执行，那么这些任务会并发执行，并各自将其他任务的运算考虑在内（respect to each other），从而省去了多线程编程的麻烦。

使用dispatch source（下次我会讲到），我们可以让GCD将事件直接传递给用户队列。例如，程序中监视socket连接的代码可以被置于它自己的dispatch queue中，这样它会异步执行，并且执行时会将程序其他部分的运算考虑在内。另外，如果使用用户队列的话，这个模块会串行执行，简化程序。

> 我们讨论了如何使用GCD来提升程序性能以及发挥多核系统的优势。尽管我们需要比较谨慎地编写并发程序，GCD还是使得我们能更简单地发挥系统的可用计算资源。
下一篇中，我们将讨论dispatch source，也就是GCD的监视内部、外部事件的机制。
