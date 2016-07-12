## 多线程
多线程是个复杂的概念，按字面意思是同步完成多项任务，提高了资源的使用效率，从硬件、操作系统、应用软件
不同的角度去看，多线程被赋予不同的内涵，对于硬件，现在市面上多数的CPU都是多核的，
多核的CPU运算多线程更为出色;从操作系统角度，是多任务，现在用的主流操作系统都是多任务的，
可以一边听歌、一边写博客;对于应用来说，多线程可以让应用有更快的回应，可以在网络下载时，
同时响应用户的触摸操作。在iOS应用中，对多线程最初的理解，就是并发，它的含义是原来先做烧水，再摘菜，
再炒菜的工作，会变成烧水的同时去摘菜，最后去炒菜。
## iOS 中的多线程
iOS中的多线程，是Cocoa框架下的多线程，通过Cocoa的封装，可以让我们更为方便的使用线程，
做过C++的同学可能会对线程有更多的理解，比如线程的创立，信号量、共享变量有认识，Cocoa框架下会方便很多，
它对线程做了封装，有些封装，可以让我们创建的对象，本身便拥有线程，也就是线程的对象化抽象，
从而减少我们的工程，提供程序的健壮性。

- GCD是(Grand Central Dispatch)的缩写 ，从系统级别提供的一个易用地多线程类库，具有运行时的特点，
能充分利用多核心硬件。GCD的API接口为C语言的函数，函数参数中多数有Block，关于Block的使用参看这里，
为我们提供强大的“接口”，对于GCD的使用参见本文
- NSOperation与Queue
NSOperation是一个抽象类，它封装了线程的细节实现，我们可以通过子类化该对象，加上NSQueue来同面向对象的思维，
管理多线程程序。具体可参看这里：一个基于NSOperation的多线程网络访问的项目。
- NSThread
NSThread是一个控制线程执行的对象，它不如NSOperation抽象，通过它我们可以方便的得到一个线程，
并控制它。但NSThread的线程之间的并发控制，是需要我们自己来控制的，可以通过NSCondition实现。

参看 iOS多线程编程之NSThread的使用

其他多线程

在Cocoa的框架下，通知、Timer和异步函数等都有使用多线程，(待补充).
## 在项目什么时候选择使用GCD，什么时候选择NSOperation?
项目中使用NSOperation的优点是NSOperation是对线程的高度抽象，在项目中使用它，会使项目的程序结构更好，
子类化NSOperation的设计思路，是具有面向对象的优点(复用、封装)，使得实现是多线程支持，而接口简单，
建议在复杂项目中使用。
项目中使用GCD的优点是GCD本身非常简单、易用，对于不复杂的多线程操作，会节省代码量，而Block参数的使用，
会是代码更为易读，建议在简单项目中使用。
## 在使用GCD以及block时要注意些什么？它们两是一回事儿么?block在ARC中和传统的MRC中的行为和用法有没有什么区别，需要注意些什么？
https://onevcat.com/2014/03/common-background-practices/
https://onevcat.com/2011/11/objc-block/
https://onevcat.com/2012/06/arc-hand-by-hand/
## GCD里面有哪几种Queue？你自己建立过串行queue吗？背后的线程模型是什么样的？
1.主队列 dispatch_main_queue(); 串行 ，更新UI
2.全局队列 dispatch_global_queue(); 并行，四个优先级：background，low，default，high
3.自定义队列 dispatch_queue_t queue ; 可以自定义是并行：DISPATCH_QUEUE_CONCURRENT或者串行DISPATCH_QUEUE_SERIAL
## GCD实现1，2并行和3串行和45串行，4，5是并行。即3依赖1，2的执行，45依赖3的执行。
**队列组的方式**
```objc
- (void) methodone{
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"%d",1);
});

dispatch_group_async(group, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"%d",2);
});

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"3");

    dispatch_group_t group1 = dispatch_group_create();

    dispatch_group_async(group1, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSLog(@"%d",4);
    });

    dispatch_group_async(group1, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSLog(@"%d",5);
    });

});

}
```
串行队列：队列中的任务只会顺序执行
```objc
dispatch_queue_t q = dispatch_queue_create(“....”, dispatch_queue_serial);
```
并行队列： 队列中的任务通常会并发执行。 　
```objc
dispatch_queue_t q = dispatch_queue_create("......", dispatch_queue_concurrent);  
```
全局队列：是系统开发的，直接拿过来用就可以；与并行队列类似，但调试时，无法确认操作所在队列。
```objc
dispatch_queue_t q = dispatch_get_global_queue(dispatch_queue_priority_default, 0);
```
主队列：每一个应用开发程序对应唯一一个主队列，直接get即可；在多线程开发中，使用主队列更新UI。
```objc
dispatch_queue_t q = dispatch_get_main_queue();
```
主队列是GCD自带的串行队列，会在主线程中执行。异步全局并发队列 开启新线程，并发执行。
并行队列里开启同步任务是有执行顺序的，只有异步才没有顺序。
串行队列开启异步任务，是有顺序的。
串行队列开启异步任务后嵌套同步任务造成死锁。
## 常用的多线程处理方式及优缺点
iOS有四种多线程编程的技术，分别是：NSThread，Cocoa NSOperation，GCD（全称：Grand Central Dispatch）,pthread。
**四种方式的优缺点介绍:**
1. NSThread优点：**NSThread 比其他两个轻量级**。
缺点：需要自己管理线程的生命周期，线程同步。线程同步对数据的加锁会有一定的系统开销。
2. Cocoa NSOperation优点:不需要关心线程管理， 数据同步的事情，可以把精力放在自己需要执行的操作上。
Cocoa operation相关的类是NSOperation, NSOperationQueue.NSOperation是个抽象类,
使用它必须用它的子类，可以实现它或者使用它定义好的两个子类: NSInvocationOperation和NSBlockOperation.
创建NSOperation子类的对象，把对象添加到NSOperationQueue队列里执行。
3. GCD(全优点)Grand Central dispatch(GCD)是Apple开发的一个多核编程的解决方案。
在iOS4.0开始之后才能使用。GCD是一个替代NSThread, NSOperationQueue,NSInvocationOperation等技术的很高效强大的技术。
4. pthread是一套通用的多线程API，适用于Linux\Windows\Unix,跨平台，可移植，使用C语言，
生命周期需要程序员管理，IOS开发中使用很少。
**GCD线程死锁**
GCD 确实好用 ，很强大，相比NSOpretion 无法提供 取消任务的功能。
如此强大的工具用不好可能会出现线程死锁。 如下代码：
```objc
- (void)viewDidLoad{
[super viewDidLoad];     
NSLog(@"=================4");
dispatch_sync(dispatch_get_main_queue(),
             ^{ NSLog(@"=================5"); });
NSLog(@"=================6");
}
```
**GCD Queue 分为三种：**
1，The main queue ：主队列，主线程就是在个队列中。
2，Global queues ： 全局并发队列。
3，用户队列:是用函数 dispatch_queue_create创建的自定义队列
**dispatch_sync 和 dispatch_async 区别：**
dispatch_async(queue,block) async 异步队列，dispatch_async
函数会立即返回, block会在后台异步执行。
dispatch_sync(queue,block) sync 同步队列，dispatch_sync
函数不会立即返回，及阻塞当前线程,等待 block同步执行完成。
**分析上面代码：**
viewDidLoad 在主线程中， 及在dispatch_get_main_queue() 中，执行到sync 时 向
dispatch_get_main_queue()插入 同步 threed。sync 会等到 后面block 执行完成才返回， sync 又再 dispatch_get_main_queue() 队列中，它是串行队列，sync 是后加入的，前一个是主线程，所以 sync 想执行 block 必须等待主线程执行完成，主线程等待 sync 返回，去执行后续内容。照成死锁，sync 等待mainThread 执行完成， mianThread 等待sync 函数返回。下面例子：
```objc
- (void)viewDidLoad{
[super viewDidLoad];
dispatch_async(dispatch_get_global_queue(0, 0), ^{
               NSLog(@"=================1");
              dispatch_sync(dispatch_get_main_queue(), ^{
              NSLog(@"=================2"); });
NSLog(@"=================3"); });
}
```
程序会完成执行，为什么不会出现死锁。
首先： async 在主线程中 创建了一个异步线程 加入 全局并发队列，async 不会等待block 执行完成，立即返回，
1. async 立即返回， viewDidLoad 执行完毕，及主线程执行完毕。
2. 同时，全局并发队列立即执行异步 block ， 打印 1， 当执行到 sync 它会等待 block 执行完成才返回，
 及等待dispatch_get_main_queue() 队列中的 mianThread 执行完成， 然后才开始调用block 。
 因为1 和 2 几乎同时执行，因为2 在全局并发队列上， 2 中执行到sync 时 1 可能已经执行完成或 等了一会，
 mainThread 很快退出， 2 等已执行后继续内容。如果阻塞了主线程，2 中的sync 就无法执行啦，
 mainThread 永远不会退出， sync 就永远等待着。
##  线程与进程的区别和联系?
1). 进程和线程都是由操作系统所体会的程序运行的基本单元，系统利用该基本单元实现系统对应用的并发性

2). 进程和线程的主要差别在于它们是不同的操作系统资源管理方式。

3). 进程有独立的地址空间，一个进程崩溃后，在保护模式下不会对其它进程产生影响，
而线程只是一个进程中的不同执行路径。

4.)线程有自己的堆栈和局部变量，但线程之间没有单独的地址空间，一个线程死掉就等于整个进程死掉。
所以多进程的程序要比多线程的程序健壮，但在进程切换时，耗费资源较大，效率要差一些。

5). 但对于一些要求同时进行并且又要共享某些变量的并发操作，只能用线程，不能用进程。

进程，是并发执行的程序在执行过程中分配和管理资源的基本单位，是一个动态概念，竟争计算机系统资源的基本单位。
每一个进程都有一个自己的地址空间，即进程空间或（虚空间）。进程空间的大小 只与处理机的位数有关，
一个 16 位长处理机的进程空间大小为 216 ，而 32 位处理机的进程空间大小为 232 。
进程至少有 5 种基本状态，它们是：初始态，执行态，等待状态，就绪状态，终止状态。

线程，在网络或多用户环境下，一个服务器通常需要接收大量且不确定数量用户的并发请求，
为每一个请求都创建一个进程显然是行不通的，——无论是从系统资源开销方面或是响应用户请求的效率方面来看。
因此，操作系统中线程的概念便被引进了。线程，是进程的一部分，一个没有线程的进程可以被看作是单线程的。
线程有时又被称为轻权进程或轻量级进程，也是 CPU 调度的一个基本单位。

**进程的执行过程是线状的**，尽管中间会发生中断或暂停，但该进程所拥有的资源只为该线状执行过程服务。
一旦发生进程上下文切换，这些资源都是要被保护起来的。这是进程宏观上的执行过程。
而进程又可有单线程进程与多线程进程两种。我们知道，进程有 一个进程控制块 PCB ，相关程序段 和
该程序段对其进行操作的数据结构集 这三部分，单线程进程的执行过程在宏观上是线性的，
微观上也只有单一的执行过程；而多线程进程在宏观上的执行过程同样为线性的，但微观上却可以有多个执行操作（线程），
如不同代码片段以及相关的数据结构集。**线程的改变只代表了 CPU 执行过程的改变，
而没有发生进程所拥有的资源变化**。除了 CPU 之外，**计算机内的软硬件资源的分配与线程无关**，
线程只能共享它所属进程的资源。与进程控制表和 PCB 相似，每个线程也有自己的线程控制表 TCB ，
而这个 TCB 中所保存的线程状态信息则要比 PCB 表少得多，这些信息主要是相关指针用堆栈（系统栈和用户栈），
寄存器中的状态数据。**进程拥有一个完整的虚拟地址空间，不依赖于线程而独立存在；反之，线程是进程的一部分，
没有自己的地址空间，与进程内的其他线程一起共享分配给该进程的所有资源**。

线程可以有效地提高系统的执行效率，但并不是在所有计算机系统中都是适用的，如某些很少做进程调度和切换的实时系统。
使用线程的好处是有多个任务需要处理机处理时，减少处理机的切换时间；而且，
线程的创建和结束所需要的系统开销也比进程的创建和结束要小得多。最适用使用线程的系统是多处理机系统和网络系统或分布式系统。


###  列举几种进程的同步机制，并比较其优缺点。
 原子操作 ?信号量机制 ? ?自旋锁 ? ?管程，会合，分布式系统
### 进程之间通信的途径
共享存储系统消息传递系统管道：以文件系统为基础
### 进程死锁的原因
资源竞争及进程推进顺序非法
### 死锁的4个必要条件
互斥、请求保持、不可剥夺、环路
### 死锁的处理
鸵鸟策略、预防策略、避免策略、检测与解除死锁

### **说出几种锁，介绍其区别**
[@synchronized, NSLock, pthread, OSSpinLock showdown, done right](http://perpendiculo.us/2009/09/synchronized-nslock-pthread-osspinlock-showdown-done-right/)
