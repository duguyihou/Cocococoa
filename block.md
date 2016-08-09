## block
### 对block的理解

Block分为三种，分别是全局block、栈block和堆block。ARC之后，我们并不需要手动copy到堆上， 通常都已经交给编译器来完成。

1. 使用block比delegate完成委托模式有什么优点?

委托模式在设计模式中是适配器模式中的对象适配器， **Objective-C中使用id类型指向一切对象**，使委托模式更为简洁。
了解委托模式的细节：
iOS设计模式---委托模式
**使用block实现委托模式，其优点是回调的block代码块定义在委托对象函数内部，使代码更为紧凑;**
**适配对象不再需要实现具体某个protocol，代码更为简洁。**

2. 多线程与block

GCD与Block

使用 dispatch_async 系列方法，可以以指定的方式执行block

GCD编程实例

dispatch_async的完整定义

`void dispatch_async( dispatch_queue_t queue, dispatch_block_t block); `
功能：在指定的队列里提交一个异步执行的block，不阻塞当前线程
通过queue来控制block执行的线程。主线程执行前文定义的 finishBlock对象

`dispatch_async(dispatch_get_main_queue(),^(void){finishBlock();});`


### \_\_block在arc和非arc下含义一样吗？

是不一样的。 在MRC中**block variable在block中使用是不会retain的 但是ARC中**block则会Retain。 取而代之的是用**weak或是**unsafe_unretained來更精确的描述weak reference的目的 其中前者只能在iOS5之后可以使用，但是比较好 (该对象release之后，此pointer会自动设成成nil) 而后者是ARC的环境下为了兼容4.x的解決方案。

```objc
__block MyClass* temp = …;    // MRC环境下使用
__weak MyClass* temp = …;    // ARC但只支援iOS5.0以上的版本
__unsafe_retained MyClass* temp = …;  //ARC且可以兼容4.x以后的版本
```

### block 实现原理

Objective-C是对C语言的扩展，block的实现是基于指针和函数指针。

从计算语言的发展，最早的goto，高级语言的指针，到面向对象语言的block，从机器的思维，一步步接近人的思维，以方便开发人员更为高效、直接的描述出现实的逻辑(需求)。

使用实例

cocoaTouch框架下动画效果的Block的调用

使用typed声明block

`typedef void(^didFinishBlock) (NSObject *ob);` 这就声明了一个didFinishBlock类型的block， 然后便可用

`@property (nonatomic,copy) didFinishBlock finishBlock;` 声明一个blokc对象，注意对象属性设置为copy，接到block 参数时，便会自动复制一份。

__block是一种特殊类型，

使用该关键字声明的局部变量，可以被block所改变，并且其在原函数中的值会被改变。

### KVO，NSNotification，delegate及block区别

KVO就是cocoa框架实现的观察者模式，一般同KVC搭配使用，通过KVO可以监测一个值的变化， 比如View的高度变化。是一对多的关系，一个值的变化会通知所有的观察者。

NSNotification是通知，也是一对多的使用场景。在某些情况下，KVO和NSNotification是一样的， 都是状态变化之后告知对方。NSNotification的特点，就是需要被观察者先主动发出通知， 然后观察者注册监听后再来进行响应，比KVO多了发送通知的一步，但是其优点是监听不局限于属性的变化， 还可以对多种多样的状态变化进行监听，监听范围广，使用也更灵活。

delegate 是代理，就是我不想做的事情交给别人做,是一对一关系。

block是delegate的另一种形式，是函数式编程的一种形式。使用场景跟delegate一样，相比delegate更灵活， 而且代理的实现更直观。

>KVO一般的使用场景是数据，需求是数据变化，比如股票价格变化，我们一般使用KVO（观察者模式）。 delegate一般的使用场景是行为，需求是需要别人帮我做一件事情，比如买卖股票，我们一般使用delegate。 Notification一般是进行全局通知，比如利好消息一出，通知大家去买入。delegate是强关联， 就是委托和代理双方互相知道，你委托别人买股票你就需要知道经纪人，经纪人也不要知道自己的顾客。 Notification是弱关联，利好消息发出，你不需要知道是谁发的也可以做出相应的反应， 同理发消息的人也不需要知道接收的人也可以正常发出消息。
