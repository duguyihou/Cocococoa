## Run Loop是什么，使用的目的，何时使用和关注点
Run Loop是一让线程能随时处理事件但不退出的机制。RunLoop 实际上是一个对象，
这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行Event Loop 的逻辑。
线程执行了这个函数后，就会一直处于这个函数内部 "接受消息->等待->处理" 的循环中，
直到这个循环结束（比如传入 quit 的消息），函数返回。让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。

OSX/iOS 系统中，提供了两个这样的对象：NSRunLoop 和 CFRunLoopRef。CFRunLoopRef
是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。
NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。
线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，
RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。

**系统默认注册了5个Mode:**

1. kCFRunLoopDefaultMode: App的默认 Mode，通常主线程是在这个 Mode 下运行的。
2. UITrackingRunLoopMode: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。
3. UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。
4. GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到。
5. kCFRunLoopCommonModes: 这是一个占位的 Mode，没有实际作用。

**Run Loop的四个作用:**

使程序一直运行接受用户输入
决定程序在何时应该处理哪些Event
调用解耦
节省CPU时间

主线程的run loop默认是启动的。iOS的应用程序里面，程序启动后会有一个如下的main() 函数：
```objc
int main(int argc, char *argv[])
 {
        @autoreleasepool {
          return UIApplicationMain(argc, argv, nil, NSStringFromClass([appDelegate class]));
       }
  }
```
重点是UIApplicationMain() 函数，这个方法会为main thread 设置一个NSRunLoop 对象，
这就解释了本文开始说的为什么我们的应用可以在无人操作的时候休息，需要让它干活的时候又能立马响应。

对其它线程来说，run loop默认是没有启动的，如果你需要更多的线程交互则可以手动配置和启动，
如果线程只是去执行一个长时间的已确定的任务则不需要。在任何一个Cocoa程序的线程中，都可以通过：
```objc
NSRunLoop   *runloop = [NSRunLoop currentRunLoop];
```
来获取到当前线程的run loop。

一个run loop就是一个事件处理循环，用来不停的监听和处理输入事件并将其分配到对应的目标上进行处理。

NSRunLoop是一种更加高明的消息处理模式，他就高明在对消息处理过程进行了更好的抽象和封装，
这样才能是的你不用处理一些很琐碎很低层次的具体消息的处理，在NSRunLoop中每一个消息就被打
包在input source或者是timer source中了。使用run loop可以使你的线程在有工作的时候工作，
没有工作的时候休眠，这可以大大节省系统资源。
**什么时候使用runloop**
仅当在为你的程序创建辅助线程的时候，你才需要显式运行一个run loop。Run loop是程序主线程基础设施的关键部分。
所以，Cocoa和Carbon程序提供了代码运行主程序的循环并自动启动run loop。IOS程序中UIApplication的run方法
（或Mac OS X中的NSApplication）作为程序启动步骤的一部分，它在程序正常启动的时候就会启动程序的主循环。
类似的，RunApplicationEventLoop函数为Carbon程序启动主循环。如果你使用xcode提供的模板创建你的程序，
那你永远不需要自己去显式的调用这些例程。

对于辅助线程，你需要判断一个run loop是否是必须的。如果是必须的，那么你要自己配置并启动它。
你不需要在任何情况下都去启动一个线程的run loop。比如，你使用线程来处理一个预先定义的长时间运行的任务时，
你应该避免启动run loop。Run loop在你要和线程有更多的交互时才需要，比如以下情况：
- 使用端口或自定义输入源来和其他线程通信
- 使用线程的定时器
- Cocoa中使用任何performSelector…的方法
- 使线程周期性工作

**关注点**
1. Cocoa中的NSRunLoop类并不是线程安全的
我们不能再一个线程中去操作另外一个线程的run loop对象，那很可能会造成意想不到的后果。
不过幸运的是CoreFundation中的不透明类CFRunLoopRef是线程安全的，而且两种类型的run loop完全可以混合使用。
Cocoa中的NSRunLoop类可以通过实例方法：
```objc
- (CFRunLoopRef)getCFRunLoop;
```
获取对应的CFRunLoopRef类，来达到线程安全的目的。
2. Run loop的管理并不完全是自动的。
我们仍必须设计线程代码以在适当的时候启动run loop并正确响应输入事件，当然前提是线程中需要用到run loop。
而且，我们还需要使用while/for语句来驱动run loop能够循环运行，下面的代码就成功驱动了一个run loop：
```objc
BOOL isRunning = NO;
do {
     isRunning = [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDatedistantFuture]];
} while (isRunning);
```
3. Run loop同时也负责autorelease pool的创建和释放
在使用手动的内存管理方式的项目中，会经常用到很多自动释放的对象，如果这些对象不能够被即时释放掉，
会造成内存占用量急剧增大。Run loop就为我们做了这样的工作，每当一个运行循环结束的时候，
它都会释放一次autorelease pool，同时pool中的所有自动释放类型变量都会被释放掉。
