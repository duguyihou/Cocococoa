## OC中load方法和initialize方法的异同
**对于load方法，官方的文档说明如下：**  

Invoked whenever a class or category is added to the Objective-C runtime; implement this method to perform class-specific behavior upon loading.
The load message is sent to classes and categories that are both dynamically loaded and statically linked, but only if the newly loaded class or category implements a method that can respond.

The order of initialization is as follows:

- All initializers in any framework you link to.
- All +load methods in your image.
- All C++ static initializers and C/C++ __attribute__(constructor) functions in your image.
- All initializers in frameworks that link to you.
In addition:

- A class’s +load method is called after all of its superclasses’ +load methods.
- A category +load method is called after the class’s own +load method.
In a custom implementation of load you can therefore safely message other unrelated classes from the same image, but any load methods implemented by those classes may not have run yet.

文档也说清楚了，**对于load方法，只要文件被引用就会被调用。load方法调用顺序是父类的load方法优先调用于子类的load方法，而本类的load方法优先于category调用**。
**对于+initialize方法，官方的文档说明如下：**
Initializes the class before it receives its first message.

The runtime sends initialize to each class in a program just before the class, or any class that inherits from it, is sent its first message from within the program. The runtime sends the initialize message to classes in a thread-safe manner. Superclasses receive this message before their subclasses. The superclass implementation may be called multiple times if subclasses do not implement initialize—the runtime will call the inherited implementation—or if subclasses explicitly call [super initialize]. If you want to protect yourself from being run multiple times, you can structure your implementation along these lines:
```Objective-C
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

Because initialize is called in a **thread-safe** manner and the order of initialize being called on different classes is not guaranteed, it’s important to do the minimum amount of work necessary in initialize methods.

Specifically, any code that takes locks that might be required by other classes in their initialize methods is liable to lead to **deadlocks**.

Therefore you should not rely on initialize for complex initialization, and should instead limit it to straightforward, class local initialization.
initialize is invoked **only once per class**. If you want to perform independent initialization for the class and for categories of the class, you should implement load methods.

**文档也很明确的说明了：**文件被引用并不代表initialize就会被调用，只有类或者子类中第一次有函数调用时，都会调用initialize。initialize是线程安全的，我们不能在initialize方法中加锁，这有可能导致死锁。我们也不应该在函数中实现复杂的代码。initialize只会被调用一次。

**+load和+initialize共同点：**

- 在不考虑开发者主动使用的情况下，系统最多会调用一次
- 如果父类和子类都被调用，父类的调用一定在子类之前
- 这两个方法不适合做复杂的操作，应该是足够简单
- 在使用时都不要过重地依赖于这两个方法，除非真正必要。在使用时一定要注意防止死锁！
- 都不需要调用[super load]、[super initialize]

**+load和+initialize不同点：**

- load方法没有自动释放池，如果做数据处理，需要释放内存，则开发者得自己添加autoreleasepool来管理内存的释放。
- 和load不同，即使子类不实现initialize方法，也会把父类的实现继承过来调用一遍。注意的是在此之前，父类的方法已经被执行过一次了，同样不需要super调用。

## 在使用GCD以及block时要注意些什么？它们两是一回事儿么？block在ARC中和传统的MRC中的行为和用法有没有什么区别，需要注意些什么？
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



## 对block的理解

Block分为三种，分别是全局block、栈block和堆block。ARC之后，我们并不需要手动copy到堆上，通常都已经交给编译器来完成。
1). 使用block和使用delegate完成委托模式有什么优点?

首先要了解什么是委托模式，委托模式在iOS中大量应用，其在设计模式中是适配器模式中的对象适配器，Objective-C中使用id类型指向一切对象，使委托模式更为简洁。了解委托模式的细节：

iOS设计模式—-委托模式

使用block实现委托模式，其优点是回调的block代码块定义在委托对象函数内部，使代码更为紧凑;

适配对象不再需要实现具体某个protocol，代码更为简洁。

2). 多线程与block

GCD与Block

使用 dispatch_async 系列方法，可以以指定的方式执行block

GCD编程实例

dispatch_async的完整定义

void?dispatch_async(
dispatch_queue_t?queue,
dispatch_block_t?block);
功能：在指定的队列里提交一个异步执行的block，不阻塞当前线程

通过queue来控制block执行的线程。主线程执行前文定义的 finishBlock对象

dispatch_async(dispatch_get_main_queue(),^(void){finishBlock();});

## __block在arc和非arc下含义一样吗？

是不一样的。
在MRC中__block variable在block中使用是不会retain的
但是ARC中__block则会Retain。
取而代之的是用__weak或是__unsafe_unretained來更精确的描述weak reference的目的
其中前者只能在iOS5之后可以使用，但是比较好 (该对象release之后，此pointer会自动设成成nil)
而后者是ARC的环境下为了兼容4.x的解決方案。
```objc
__block MyClass* temp = …;    // MRC环境下使用
__weak MyClass* temp = …;    // ARC但只支援iOS5.0以上的版本
__unsafe_retained MyClass* temp = …;  //ARC且可以兼容4.x以后的版本

```

##  block 实现原理
Objective-C是对C语言的扩展，block的实现是基于指针和函数指针。

从计算语言的发展，最早的goto，高级语言的指针，到面向对象语言的block，从机器的思维，一步步接近人的思维，以方便开发人员更为高效、直接的描述出现实的逻辑(需求)。

使用实例

cocoaTouch框架下动画效果的Block的调用

使用typed声明block

typedef?void(^didFinishBlock)?(NSObject?*ob);
这就声明了一个didFinishBlock类型的block，
然后便可用

@property?(nonatomic,copy)?didFinishBlock?finishBlock;
声明一个blokc对象，注意对象属性设置为copy，接到block 参数时，便会自动复制一份。

__block是一种特殊类型，

使用该关键字声明的局部变量，可以被block所改变，并且其在原函数中的值会被改变。

## 对runtime的理解
1. 消息是如何转发的？
动态解析过程大致是这样的：通过resolveInstanceMethod允许开发者决定是否动态添加方法，若返回NO，就直接进入doesNotRecognizeSelector，流程结束，否则需要通过class_addMethod动态添加方法并返回YES并进入下一步。forwardingTargetForSelector是第二步，允许开发者决定将由哪个对象响应这个selector，如果返回nil，则直接进入doesNotRecognizeSelector，流程结束，否则需要返回一个对象，但不能是self。进入第三步指定方法签名methodSignatureForSelector，若返回nil，则直接进入doesNotRecognizeSelector且流程结束，否则指定签名，并进入下一步forwardInvocation。forwardInvocation允许开发者修改响应者、方法实现等。若没有实现forwardInvocation，则直接进入doesNotRecognizeSelector，流程结束。

2. 方法调用会被缓存吗？如何缓存过，又是如何查找的呢？
方法是会缓存进来了，不然下次再调用又要重新查一次，效率是不高的。采用散列（哈希）的方式来缓存，查询的效率是比较高的，因此内部会采用散列缓存起来。

3. 对象的内存是如何布局的？
成员变量（包括父类）都保存在对象本身的存储空间内；本类的实例方法保存在类对象中，本类的类方法保存在元类对象中；父类的实例方法保存在各级super class中，父类的类方法保存在各级super meta class中。

4. runtime有哪些应用场景？

- 给category添加属性
- Method-Swizzling hook方法，然后交换方法实现来达到调用系统方法之前先做一些额外的处理
- 埋点处理
- 字典与模型互转
- 模型自动获取所有属性并转换成SQL语句操作数据库

### 们说的oc是动态运行时语言是什么意思?
多态。 主要是将数据类型的确定由编译时，推迟到了运行时。
这个问题其实浅涉及到两个概念，运行时和多态。
简单来说，运行时机制使我们直到运行时才去决定一个对象的类别，以及调用该类别对象指定方法。
多态：不同对象以自己的方式响应相同的消息的能力叫做多态。意思就是假设生物类(life)都用有一个相同的方法-eat;
那人类属于生物，猪也属于生物，都继承了life后，实现各自的eat，但是调用是我们只需调用各自的eat方法。
也就是不同的对象以自己的方式响应了相同的消息(响应了eat这个选择器)。
###  关于多态性
多态，子类指针可以赋值给父类。
这个题目其实可以出到一切面向对象语言中，
因此关于多态，继承和封装基本最好都有个自我意识的理解，也并非一定要把书上资料上写的能背出来

## 对MVC和MVVM的理解 你还熟悉什么设计模式？
MVC是出现比较早的架构设计模式，而且到现在已经是很成熟了。出现MVVM的原因是MVC中的V越来越复杂，于是才有人想要给V瘦身。

>设计模式：并不是一种新技术，而是一种编码经验，使用比如java中的接口，iphone中的协议，继承关系等基本手段，用比较成熟的逻辑去处理某一种类型的事情，总结为所谓设计模式。面向对象编程中，java已经归纳了23种设计模式。

- mvc设计模式 ：模型，视图，控制器，可以将整个应用程序在思想上分成三大块，对应是的数据的存储或处理，前台的显示，业务逻辑的控制。 Iphone本身的设计思想就是遵循mvc设计模式。其不属于23种设计模式范畴。

- 代理模式：代理模式给某一个对象提供一个代理对象，并由代理对象控制对源对象的引用.比如一个工厂生产了产品，并不想直接卖给用户，而是搞了很多代理商，用户可以直接找代理商买东西，代理商从工厂进货.常见的如QQ的自动回复就属于代理拦截，代理模式在iphone中得到广泛应用.

- 单例模式：说白了就是一个类不通过alloc方式创建对象，而是用一个静态方法返回这个类的对象。系统只需要拥有一个的全局对象，这样有利于我们协调系统整体的行为，比如想获得[UIApplication sharedApplication];任何地方调用都可以得到 UIApplication的对象，这个对象是全局唯一的。

- 观察者模式： 当一个物体发生变化时，会通知所有观察这个物体的观察者让其做出反应。实现起来无非就是把所有观察者的对象给这个物体，当这个物体的发生改变，就会调用遍历所有观察者的对象调用观察者的方法从而达到通知观察者的目的。

- 工厂模式：
```objc
public class Factory{
public static Sample creator(int which){
if (which==1)
return new SampleA();
else if (which==2)
return new SampleB();
}
}
```
### 对于单例(Singleton)的理解
在objective-c中要实现一个单例类，至少需要做以下四个步骤：
1. 为单例对象实现一个静态实例，并初始化，然后设置成nil。
2. 实现一个实例构造方法检查上面声明的静态实例是否为nil，如果是则新建并返回一个本类的实例，
3. 重写allocWithZone方法，用来保证其他人直接使用alloc和init试图获得一个新实力的时候不产生一个新实例，
4. 适当实现allocWitheZone，copyWithZone，release和autorelease。

## 如何使用Xcode设计通用应用?
使用MVC模式设计应用，其中Model层完成脱离界面，即在Model层，其是可运行在任何设备上，在controller层，根据iPhone与iPad(独有UISplitViewController)的不同特点选择不同的viewController对象。在View层，可根据现实要求，来设计，其中以xib文件设计时，其设置其为universal。

## UITableView的调优
通常来说，在开发中注意以下问题，可以使列表滚动比较流畅，但是对于特别复杂的列表就需要做额外的优化处理：
- 重用cell，设置好cellIdentifier
- 重用header、footer view，设置好identifier
- 若高度固定，直接使用rowHight；若不固定则使用heightForRowAtIndexPath代理方法
- 缓存cell的高度、header/footer view的高度
- 不要修改view的opaque，默认就是YES,表示不透明度
- 不要动态添加子view到cell上，直接在初始时创建，然后做显示与隐藏操作
- 尽量不要直接使用cornerRadius，采用镂空图或者Core Graphics API来绘制
- 将I/O操作、复杂运算放到子线程中处理，再回到主线程更新UI

如果列表比较复杂，对于上面的做好后，还是不够流畅，就需要通过线程s工具来检测哪些地方可以优化了。

## 用Instrument优化动画性能的经历
[iOS App性能优化](http://www.hrchen.com/2013/05/performance-with-instruments/)

1. Separate by Thread: 每个线程应该分开考虑。只有这样你才能揪出那些大量占用CPU的"重"线程。
2. Invert Call Tree: 从上倒下跟踪堆栈,这意味着你看到的表中的方法,将已从第0帧开始取样,这通常你是想要的,只有这样你才能看到CPU中话费时间最深的方法.也就是说FuncA{FunB{FunC}} 勾选此项后堆栈以C->B-A 把调用层级最深的C显示在最外面。
3. Hide System Libraries: 勾选此项你会显示你app的代码,这是非常有用的. 因为通常你只关心cpu花在自己代码上的时间不是系统上的。
4. Flatten Recursion: 递归函数, 每个堆栈跟踪一个条目。
5. Top Functions: 一个函数花费的时间直接在该函数中的总和，以及在函数调用该函数所花费的时间的总时间。因此，如果函数A调用B，那么A的时间报告在A花费的时间加上B.花费的时间,这非常真有用，因为它可以让你每次下到调用堆栈时挑最大的时间数字，归零在你最耗时的方法。

内存泄漏有两种泄漏。第一个是真正的内存泄漏，一个对象尚未被释放，但是不再被引用的了。因此，存储器不能被重新使用。第二类泄漏是比较麻烦一些。这就是所谓的“无界内存增长”。这发生在内存继续分配，并永远不会有机会被释放。如果永远这样下去你的程序占用的内存会无限大,当超过一定内存的话 会被系统的看门狗给kill掉。

内存警告是ios处理app最好的方式，尤其是在内存越来越吃紧的时候,你需要清除一些内存。内存一直增长其实也不一定是你的代码出了问题,也有可能是UIKit 系统框架本身导致的。

**内存泄露**
这一类泄漏是前面提到的 - 当一个对象不再被引用时出现的那种,检测泄漏可以理解为一个很复杂的事情，但泄漏的工具记得已分配的所有对象，通过定期扫描每个对象以确定是否有任何不能从任何其他对象访问的。
## 对ARC的理解

ARC是编译器帮我们完成的，我们不再手动添加retain、relase、autorelease，而且在运行期还会帮助我们优化。但是ARC并不是万能的，它并不能自我理解循环引用问题，依然需要我们手动解决循环引用的问题。

ARC管理都会放到自动释放池中，如果我们需要做一些循环操作，生成大量的临时变量，我们还是需要加一下autoreleasepool，以及时地释放内存。

ARC下对于属性修饰符不同，其内存管理策略也不一样：
- strong：强引用，引用计数加1
- weak：弱引用，引用计数没有加1
- copy：强引用，引用计数加1

ARC下还是有可能出现内存泄露的，内存得不到释放，特别是使用block的时候，一定要学会分析是否形成循环引用。

## 野指针是什么，iOS开发中什么情况下会有野指针？

所谓野指针，是指指向内存已经被释放的内存区的指针。

当进入播放页面时马上又返回上一个页面，偶尔出现闪退，原因就是出现了野指针（访问了已释放的对象内存区）。当进入播放页面时，就会立刻去解析视频数据，内部是FFMPEG操作，当快速返回上一个页面时，FFMPEG还在操作中，导致访问了已释放的对象。
使用block时，也会出现野指针。

## OC的数组中，添加nil对象会有什么问题?
对于数组跟字典，插入nil对象都会引起崩溃。
```objc
NSMutableArray *array = [[NSMutableArray alloc] init];

//  -[__NSArrayM insertObject:atIndex:]: object cannot be nil'
[array addObject:nil];
```
但是，如果我们在初始化时，通过下面的API来添加nil，是不会有事的，只是表示结束而已。
```objc
NSArray *array = [[NSArray alloc] initWithObjects:@"sss", nil, @"sfsdf"];
// 结果只有sss，后面的因为中间有nil而被过滤了
NSLog(@"%@", array);
```
## Object-c的类可以多重继承么?可以实现多个接口么?Category是什么?重写一个类的方式用继承好还是分类好?为什么?
 Object-c的类不可以多重继承;可以实现多个接口，通过实现多个接口可以完成C++的多重继承;Category是类别，一般情况用分类好，用Category去重写类的方法，仅对本Category有效，不会影响到其他类与原有类的关系。
##  \#import 跟#include 又什么区别，@class呢, #import<> 跟 #import””又什么区别
\#import是Objective-C导入头文件的关键字，#include是C/C++导入头文件的关键字,使用#import头文件会自动只导入一次，不会重复导入，相当于#include和#pragma once;@class告诉编译器某个类的声明，当执行时，才去查看类的实现文件，可以解决头文件的相互包含;#import<>用来包含系统的头文件，#import””用来包含用户头文件。

## Object-C有多继承吗？没有的话用什么代替？cocoa 中所有的类都是NSObject 的子类
多继承在这里是用protocol 委托代理 来实现的

你不用去考虑繁琐的多继承 ,虚基类的概念.

ood的多态特性 在 obj-c 中通过委托来实现.

## 对于语句NSString*obj = [[NSData alloc] init]; obj在编译时和运行时分别时什么类型的对象?
编译时是NSString的类型;运行时是NSData类型的对象

## 常见的object-c的数据类型有那些， 和C的基本数据类型有什么区别?如：NSInteger和int
object-c的数据类型有NSString，NSNumber，NSArray，NSMutableArray，NSData等等，这些都是class，创建后便是对象，而C语言的基本数据类型int，只是一定字节的内存空间，用于存放数值;NSInteger是基本数据类型，并不是NSNumber的子类，当然也不是NSObject的子类。NSInteger是基本数据类型Int或者Long的别名(NSInteger的定义typedef long NSInteger)，它的区别在于，NSInteger会根据系统是32位还是64位来决定是本身是int还是Long。

## id 声明的对象有什么特性?
Id 声明的对象具有运行时的特性，即可以指向任意类型的objcetive-c的对象;

## Objective-C如何对内存管理的,说说你的看法和解决方法?
Objective-C的内存管理主要有三种方式ARC(自动内存计数)、手动内存计数、内存池。

1. (Garbage Collection)自动内存计数：这种方式和java类似，在你的程序的执行过程中。始终有一个高人在背后准确地帮你收拾垃圾，你不用考虑它什么时候开始工作，怎样工作。你只需要明白，我申请了一段内存空间，当我不再使用从而这段内存成为垃圾的时候，我就彻底的把它忘记掉，反正那个高人会帮我收拾垃圾。遗憾的是，那个高人需要消耗一定的资源，在携带设备里面，资源是紧俏商品所以iPhone不支持这个功能。所以“Garbage Collection”不是本入门指南的范围，对“Garbage Collection”内部机制感兴趣的同学可以参考一些其他的资料，不过说老实话“Garbage Collection”不大适合适初学者研究。

解决: 通过alloc – initial方式创建的, 创建后引用计数+1, 此后每retain一次引用计数+1, 那么在程序中做相应次数的release就好了.

2. (Reference Counted)手动内存计数：就是说，从一段内存被申请之后，就存在一个变量用于保存这段内存被使用的次数，我们暂时把它称为计数器，当计数器变为0的时候，那么就是释放这段内存的时候。比如说，当在程序A里面一段内存被成功申请完成之后，那么这个计数器就从0变成1(我们把这个过程叫做alloc)，然后程序B也需要使用这个内存，那么计数器就从1变成了2(我们把这个过程叫做retain)。紧接着程序A不再需要这段内存了，那么程序A就把这个计数器减1(我们把这个过程叫做release);程序B也不再需要这段内存的时候，那么也把计数器减1(这个过程还是release)。当系统(也就是Foundation)发现这个计数器变 成员了0，那么就会调用内存回收程序把这段内存回收(我们把这个过程叫做dealloc)。顺便提一句，如果没有Foundation，那么维护计数器，释放内存等等工作需要你手工来完成。

解决:一般是由类的静态方法创建的, 函数名中不会出现alloc或init字样, 如[NSString string]和[NSArray arrayWithObject:], 创建后引用计数+0, 在函数出栈后释放, 即相当于一个栈上的局部变量. 当然也可以通过retain延长对象的生存期.

3. (NSAutoRealeasePool)内存池：可以通过创建和释放内存池控制内存申请和回收的时机.

解决:是由autorelease加入系统内存池, 内存池是可以嵌套的, 每个内存池都需要有一个创建释放对, 就像main函数中写的一样. 使用也很简单, 比如[[[NSString alloc]initialWithFormat:@”Hey you!”] autorelease], 即将一个NSString对象加入到最内层的系统内存池, 当我们释放这个内存池时, 其中的对象都会被释放.

## 使用nonatomic一定是线程安全的吗？
不是的。
atomic原子操作，系统会为setter方法加锁。 具体使用 @synchronized(self){//code }
nonatomic不会为setter方法加锁。
atomic：线程安全，需要消耗大量系统资源来为属性加锁
nonatomic：非线程安全，适合内存较小的移动设备

## 原子(atomic)跟非原子(non-atomic)属性有什么区别?
1. atomic提供多线程安全。是防止在写未完成的时候被另外一个线程读取，造成数据错误
2. non-atomic:在自己管理内存的环境中，解析的访问器保留并自动释放返回的值，如果指定了 nonatomic ，那么访问器只是简单地返回这个值。

## 看下面的程序,第一个NSLog会输出什么?这时str的retainCount是多少?第二个和第三个呢? 为什么?
```objc
NSMutableArray*?ary?=?[[NSMutableArray?array]?retain];
NSString?*str?=?[NSString?stringWithFormat:@"test"];
[str?retain];
[aryaddObject:str];
NSLog(@”%@%d”,str,[str?retainCount]);
[str?retain];
[str?release];
[str?release];
NSLog(@”%@%d”,str,[str?retainCount]);
[aryremoveAllObjects];
NSLog(@”%@%d”,str,[str?retainCount]);
```
str的retainCount创建+1，retain+1，加入数组自动+1 3
retain+1，release-1，release-1 2
数组删除所有对象，所有数组内的对象自动-1 1

## 内存管理的几条原则是什么?按照默认法则.那些关键字生成的对象需要手动释放?在和property结合的时候怎样有效的避免内存泄露?
谁申请，谁释放
遵循Cocoa Touch的使用原则;
内存管理主要要避免“过早释放”和“内存泄漏”，对于“过早释放”需要注意@property设置特性时，一定要用对特性关键字，对于“内存泄漏”，一定要申请了要负责释放，要细心。
关键字alloc 或new 生成的对象需要手动释放;
设置正确的property属性，对于retain需要在合适的地方释放，

## 如何对iOS设备进行性能测试?
Profile-> Instruments ->Time Profiler

## Object C中创建线程的方法是什么?如果在主线程中执行代码，方法是什么?如果想延时执行代码、方法又是什么?
线程创建有三种方法：使用NSThread创建、使用GCD的dispatch、使用子类化的NSOperation,然后将其加入NSOperationQueue;在主线程执行代码，方法是`performSelectorOnMainThread`，如果想延时执行代码可以用`performSelector:onThread:withObject:waitUntilDone:`

## 类别(category)的作用?继承和类别在实现中有何区别?
category 可以在不获悉，不改变原来代码的情况下往里面添加新的方法，只能添加，不能删除修改，并且如果类别和原来类中的方法产生名称冲突，则类别将覆盖原来的方法，因为类别具有更高的优先级。

类别主要有3个作用：
1. 将类的实现分散到多个不同文件或多个不同框架中。
2. 创建对私有方法的前向引用。
3. 向对象添加非正式协议。
继承可以增加，修改或者删除方法，并且可以增加属性。
###  类别和类扩展的区别
category和extensions的不同在于 后者可以添加属性。另外后者添加的方法是必须要实现的。
extensions可以认为是一个私有的Category。

##  oc中的协议和java中的接口概念有何不同?
OC中的代理有2层含义，官方定义为 formal和informal protocol。前者和Java接口一样。
informal protocol中的方法属于设计模式考虑范畴，不是必须实现的，但是如果有实现，就会改变类的属性。
其实关于正式协议，类别和非正式协议我很早前学习的时候大致看过，也写在了学习教程里
“非正式协议概念其实就是类别的另一种表达方式“这里有一些你可能希望实现的方法，你可以使用他们更好的完成工作”。
这个意思是，这些是可选的。比如我门要一个更好的方法，我们就会申明一个这样的类别去实现。然后你在后期可以直接使用这些更好的方法。
这么看，总觉得类别这玩意儿有点像协议的可选协议。”
现在来看，其实protocal已经开始对两者都统一和规范起来操作，因为资料中说“非正式协议使用interface修饰“，
现在我们看到协议中两个修饰词：“必须实现(@requied)”和“可选实现(@optional)”。

## KVC & KVO
KVC:键 – 值编码是一种间接访问对象的属性使用字符串来标识属性，而不是通过调用存取方法，直接或通过实例变量访问的机制。

很多情况下可以简化程序代码。apple文档其实给了一个很好的例子。

KVO:键值观察机制，他提供了观察某一属性变化的方法，极大的简化了代码。

具体用看到嗯哼用到过的一个地方是对于按钮点击变化状态的的监控。

比如我自定义的一个button
```objc
[self addObserver:self forKeyPath:@"highlighted" options:0 context:nil];
#pragma mark?KVO
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
if ([keyPath isEqualToString:@"highlighted"]) {
[self setNeedsDisplay];
}
}
```
对于系统是根据keypath去取的到相应的值发生改变，理论上来说是和kvc机制的道理是一样的。

对于kvc机制如何通过key寻找到value：

“当通过KVC调用对象时，比如：[self valueForKey:@”someKey”]时，程序会自动试图通过几种不同的方式解析这个调用。首先查找对象是否带有 someKey 这个方法，如果没找到，会继续查找对象是否带有someKey这个实例变量(iVar)，如果还没有找到，程序会继续试图调用 -(id) valueForUndefinedKey:这个方法。如果这个方法还是没有被实现的话，程序会抛出一个NSUndefinedKeyException异常错误。

Key-Value Coding查找方法的时候，不仅仅会查找someKey这个方法，还会查找getsomeKey这个方法，前面加一个get，或者_someKey以及_getsomeKey这几种形式。同时，查找实例变量的时候也会不仅仅查找someKey这个变量，也会查找_someKey这个变量是否存在。

## 代理(Delegate)的作用?
>代理的目的是改变或传递控制链。

- 允许一个类在某些特定时刻通知到其他类，而不需要获取到那些类的指针。可以减少框架复杂度。
- 代理可以理解为java中的回调监听机制的一种类似。

## 通知和协议的不同之处?
协议有控制链(has-a)的关系，通知没有。
delegate针对one-to-one关系，用于sender接受到reciever的某个功能反馈值。

notification针对one-to-one/many/none,reciver,用于通知多个object某个事件。
## 什么是推送(push)消息?
推送通知更是一种技术。
简单点就是客户端获取资源的一种手段。
普通情况下，都是客户端主动的pull。
推送则是服务器端主动push。

## Respond chain
事件响应链。包括点击事件，画面刷新事件等。在视图栈内从上至下，或者从下之上传播。
可以说点事件的分发，传递以及处理。具体可以去看下touch事件这块。
可以从责任链模式，来讲通过事件响应链处理，其拥有的扩展性

## OC的垃圾回收机制?
OC2.0有Garbage collection，但是iOS平台不提供。
一般我们了解的objective-c对于内存管理都是手动操作的，但是也有自动释放池。
但是差了大部分资料，貌似不要和arc机制搞混就好了。

## NSOperation queue?
存放NSOperation的集合类。
操作和操作队列，基本可以看成java中的线程和线程池的概念。用于处理ios多线程开发的问题。
网上部分资料提到一点是，虽然是queue，但是却并不是带有队列的概念，放入的操作并非是按照严格的先进现出。
这边又有个疑点是，对于队列来说，先进先出的概念是Afunc添加进队列，Bfunc紧跟着也进入队列，Afunc先执行这个是必然的，
但是Bfunc是等Afunc完全操作完以后，B才开始启动并且执行，因此队列的概念离乱上有点违背了多线程处理这个概念。
但是转念一想其实可以参考银行的取票和叫号系统。
因此对于A比B先排队取票但是B率先执行完操作，我们亦然可以感性认为这还是一个队列。
但是后来看到一票关于这操作队列话题的文章，其中有一句提到
“因为两个操作提交的时间间隔很近，线程池中的线程，谁先启动是不定的。”
瞬间觉得这个queue名字有点忽悠人了，还不如pool~
综合一点，我们知道他可以比较大的用处在于可以帮组多线程编程就好了。

## Lazy load
最好也最简单的一个列子就是tableView中图片的加载显示了。
一个延时载，避免内存过高，一个异步加载，避免线程堵塞。

## 是否在一个视图控制器中嵌入两个tableview控制器?
一个视图控制只提供了一个View视图，理论上一个tableViewController也不能放吧，
只能说可以嵌入一个tableview视图。当然，题目本身也有歧义，如果不是我们定性思维认为的UIViewController，而是宏观的表示视图控制者，那我们倒是可以把其看成一个视图控制者，它可以控制多个视图控制器，比如TabbarController那样的感觉。

## 一个tableView是否可以关联两个不同的数据源?你会怎么处理?
首先我们从代码来看，数据源如何关联上的，其实是在数据源关联的代理方法里实现的。
因此我们并不关心如何去关联他，他怎么关联上，方法只是让我返回根据自己的需要去设置如相关的数据源。
因此，我觉得可以设置多个数据源啊，但是有个问题是，你这是想干嘛呢?想让列表如何显示，不同的数据源分区块显示?

## 什么时候使用NSMutableArray，什么时候使用NSArray?
当数组在程序运行时，需要不断变化的，使用NSMutableArray，当数组在初始化后，便不再改变的，使用NSArray。需要指出的是，使用NSArray只表明的是该数组在运行时不发生改变，即不能往NSAarry的数组里新增和删除元素，但不表明其数组內的元素的内容不能发生改变。NSArray是线程安全的，NSMutableArray不是线程安全的，多线程使用到NSMutableArray需要注意。

## 给出委托方法的实例，并且说出UITableVIew的Data Source方法
CocoaTouch框架中用到了大量委托，其中UITableViewDelegate就是委托机制的典型应用，是一个典型的使用委托来实现适配器模式，其中UITableViewDelegate协议是目标，tableview是适配器，实现UITableViewDelegate协议，并将自身设置为talbeview的delegate的对象，是被适配器，一般情况下该对象是UITableViewController。

UITableVIew的Data Source方法有- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section;

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;

## 如果我们不创建内存池，是否有内存池提供给我们?
界面线程维护着自己的内存池，用户自己创建的数据线程，则需要创建该线程的内存池

## 什么时候需要在程序中创建内存池?
用户自己创建的数据线程，则需要创建该线程的内存池

## 类NSObject的哪些方法经常被使用?
NSObject是Objetive-C的基类，其由NSObject类及一系列协议构成。
其中类方法alloc、class、 description 对象方法init、dealloc、– performSelector:withObject:afterDelay:等经常被使用

## 什么是简便构造方法?
简便构造方法一般由CocoaTouch框架提供，如NSNumber的 `+ numberWithBool:`  `+ numberWithChar:` `+ numberWithDouble:` `+ numberWithFloat:` `+ numberWithInt:`
Foundation下大部分类均有简便构造方法，我们可以通过简便构造方法，获得系统给我们创建好的对象，并且不需要手动释放。

## UIView的动画效果有那些?
UIViewAnimationOptionCurveEaseInOut
UIViewAnimationOptionCurveEaseIn
UIViewAnimationOptionCurveEaseOut
UIViewAnimationOptionTransitionFlipFromLeft
UIViewAnimationOptionTransitionFlipFromRight
UIViewAnimationOptionTransitionCurlUp
UIViewAnimationOptionTransitionCurlDown

## 在iPhone应用中如何保存数据?
1. 通过web服务，保存在服务器上
2. 通过NSCoder固化机制，将对象保存在文件中
3. 通过SQlite或CoreData保存在文件数据库中

## ios 平台怎么做数据的持久化?coredata 和sqlite有无必然联系？coredata是一个关系型数据库吗？
iOS 中可以有四种持久化数据的方式：属性列表(plist)、对象归档、 SQLite3 和 Core Data； core data 可以使你以图形界面的方式快速的定义 app 的数据模型，同时在你的代码中容易获取到它。 coredata 提供了基础结构去处理常用的功能，例如保存，恢复，撤销和重做，允许你在 app 中继续创建新的任务。在使用 core data 的时候，你不用安装额外的数据库系统，因为 core data 使用内置的 sqlite 数据库。 core data 将你 app 的模型层放入到一组定义在内存中的数据对象。 coredata 会追踪这些对象的改变，同时可以根据需要做相反的改变，例如用户执行撤销命令。当 core data 在对你 app 数据的改变进行保存的时候， core data 会把这些数据归档，并永久性保存。 mac os x 中sqlite 库，它是一个轻量级功能强大的关系数据引擎，也很容易嵌入到应用程序。可以在多个平台使用， sqlite 是一个轻量级的嵌入式 sql 数据库编程。与 core data 框架不同的是， sqlite 是使用程序式的， sql 的主要的 API 来直接操作数据表。 Core Data 不是一个关系型数据库，也不是关系型数据库管理系统 (RDBMS) 。虽然 Core Dta 支持SQLite 作为一种存储类型，但它不能使用任意的 SQLite 数据库。 Core Data 在使用的过程种自己创建这个数据库。 Core Data 支持对一、对多的关系。

## 和coredata一起有哪几种持久化存储机制?
存入到文件、 存入到NSUserDefaults(系统plist文件中)、存入到Sqlite文件数据库

## 什么是NSManagedObject模型?
NSManagedObject是NSObject的子类 ，也是coredata的重要组成部分，它是一个通用的类,实现了core data 模型层所需的基本功能，用户可通过子类化NSManagedObject，建立自己的数据模型。
## 什么是NSManagedobjectContext?
NSManagedobjectContext对象负责应用和数据库之间的交互。

## NSPredicate
通过给定的逻辑条件作为约束条件，完成对数据的筛选。
```objc
predicate = [NSPredicate predicateWithFormat:@"customerID == %d",n];
a = [customers filteredArrayUsingPredicate:predicate];
```
##  简单介绍下NSURLConnection类及+ sendSynchronousRequest:returningResponse:error:与– initWithRequest:delegate:两个方法的区别?
NSURLConnection主要用于网络访问，其中+ sendSynchronousRequest:returningResponse:error:是同步访问数据，即当前线程会阻塞，并等待request的返回的response，而– initWithRequest:delegate:使用的是异步加载，当其完成网络访问后，会通过delegate回到主线程，并其委托的对象。

## 多线程
多线程是个复杂的概念，按字面意思是同步完成多项任务，提高了资源的使用效率，从硬件、操作系统、应用软件不同的角度去看，多线程被赋予不同的内涵，对于硬件，现在市面上多数的CPU都是多核的，多核的CPU运算多线程更为出色;从操作系统角度，是多任务，现在用的主流操作系统都是多任务的，可以一边听歌、一边写博客;对于应用来说，多线程可以让应用有更快的回应，可以在网络下载时，同时响应用户的触摸操作。在iOS应用中，对多线程最初的理解，就是并发，它的含义是原来先做烧水，再摘菜，再炒菜的工作，会变成烧水的同时去摘菜，最后去炒菜。
### iOS 中的多线程
iOS中的多线程，是Cocoa框架下的多线程，通过Cocoa的封装，可以让我们更为方便的使用线程，做过C++的同学可能会对线程有更多的理解，比如线程的创立，信号量、共享变量有认识，Cocoa框架下会方便很多，它对线程做了封装，有些封装，可以让我们创建的对象，本身便拥有线程，也就是线程的对象化抽象，从而减少我们的工程，提供程序的健壮性。

- GCD是(Grand Central Dispatch)的缩写 ，从系统级别提供的一个易用地多线程类库，具有运行时的特点，能充分利用多核心硬件。GCD的API接口为C语言的函数，函数参数中多数有Block，关于Block的使用参看这里，为我们提供强大的“接口”，对于GCD的使用参见本文
- NSOperation与Queue
NSOperation是一个抽象类，它封装了线程的细节实现，我们可以通过子类化该对象，加上NSQueue来同面向对象的思维，管理多线程程序。具体可参看这里：一个基于NSOperation的多线程网络访问的项目。
- NSThread
NSThread是一个控制线程执行的对象，它不如NSOperation抽象，通过它我们可以方便的得到一个线程，并控制它。但NSThread的线程之间的并发控制，是需要我们自己来控制的，可以通过NSCondition实现。

参看 iOS多线程编程之NSThread的使用

其他多线程

在Cocoa的框架下，通知、Timer和异步函数等都有使用多线程，(待补充).
### 在项目什么时候选择使用GCD，什么时候选择NSOperation?
项目中使用NSOperation的优点是NSOperation是对线程的高度抽象，在项目中使用它，会使项目的程序结构更好，子类化NSOperation的设计思路，是具有面向对象的优点(复用、封装)，使得实现是多线程支持，而接口简单，建议在复杂项目中使用。
项目中使用GCD的优点是GCD本身非常简单、易用，对于不复杂的多线程操作，会节省代码量，而Block参数的使用，会是代码更为易读，建议在简单项目中使用。

## 常用的多线程处理方式及优缺点
iOS有四种多线程编程的技术，分别是：NSThread，Cocoa NSOperation，GCD（全称：Grand Central Dispatch）,pthread。

**四种方式的优缺点介绍:**

1. NSThread优点：**NSThread 比其他两个轻量级**。
缺点：需要自己管理线程的生命周期，线程同步。线程同步对数据的加锁会有一定的系统开销。

2. Cocoa NSOperation优点:不需要关心线程管理， 数据同步的事情，可以把精力放在自己需要执行的操作上。Cocoa operation相关的类是NSOperation, NSOperationQueue.NSOperation是个抽象类,使用它必须用它的子类，可以实现它或者使用它定义好的两个子类: NSInvocationOperation和NSBlockOperation.创建NSOperation子类的对象，把对象添加到NSOperationQueue队列里执行。

3. GCD(全优点)Grand Central dispatch(GCD)是Apple开发的一个多核编程的解决方案。在iOS4.0开始之后才能使用。GCD是一个替代NSThread, NSOperationQueue,NSInvocationOperation等技术的很高效强大的技术。

4. pthread是一套通用的多线程API，适用于Linux\Windows\Unix,跨平台，可移植，使用C语言，生命周期需要程序员管理，IOS开发中使用很少。

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
1，async 立即返回， viewDidLoad 执行完毕，及主线程执行完毕。
2，同时，全局并发队列立即执行异步 block ， 打印 1， 当执行到 sync 它会等待 block 执行完成才返回， 及等待dispatch_get_main_queue() 队列中的 mianThread 执行完成， 然后才开始调用block 。因为1 和 2 几乎同时执行，因为2 在全局并发队列上， 2 中执行到sync 时 1 可能已经执行完成或 等了一会，mainThread 很快退出， 2 等已执行后继续内容。如果阻塞了主线程，2 中的sync 就无法执行啦，mainThread 永远不会退出， sync 就永远等待着。


###  线程与进程的区别和联系?
1). 进程和线程都是由操作系统所体会的程序运行的基本单元，系统利用该基本单元实现系统对应用的并发性

2). 进程和线程的主要差别在于它们是不同的操作系统资源管理方式。

3). 进程有独立的地址空间，一个进程崩溃后，在保护模式下不会对其它进程产生影响，而线程只是一个进程中的不同执行路径。

4.)线程有自己的堆栈和局部变量，但线程之间没有单独的地址空间，一个线程死掉就等于整个进程死掉。所以多进程的程序要比多线程的程序健壮，但在进程切换时，耗费资源较大，效率要差一些。

5). 但对于一些要求同时进行并且又要共享某些变量的并发操作，只能用线程，不能用进程。

进程，是并发执行的程序在执行过程中分配和管理资源的基本单位，是一个动态概念，竟争计算机系统资源的基本单位。每一个进程都有一个自己的地址空间，即进程空间或（虚空间）。进程空间的大小 只与处理机的位数有关，一个 16 位长处理机的进程空间大小为 216 ，而 32 位处理机的进程空间大小为 232 。进程至少有 5 种基本状态，它们是：初始态，执行态，等待状态，就绪状态，终止状态。

线程，在网络或多用户环境下，一个服务器通常需要接收大量且不确定数量用户的并发请求，为每一个请求都创建一个进程显然是行不通的，——无论是从系统资源开销方面或是响应用户请求的效率方面来看。因此，操作系统中线程的概念便被引进了。线程，是进程的一部分，一个没有线程的进程可以被看作是单线程的。线程有时又被称为轻权进程或轻量级进程，也是 CPU 调度的一个基本单位。

**进程的执行过程是线状的**，尽管中间会发生中断或暂停，但该进程所拥有的资源只为该线状执行过程服务。一旦发生进程上下文切换，这些资源都是要被保护起来的。这是进程宏观上的执行过程。而进程又可有单线程进程与多线程进程两种。我们知道，进程有 一个进程控制块 PCB ，相关程序段 和 该程序段对其进行操作的数据结构集 这三部分，单线程进程的执行过程在宏观上是线性的，微观上也只有单一的执行过程；而多线程进程在宏观上的执行过程同样为线性的，但微观上却可以有多个执行操作（线程），如不同代码片段以及相关的数据结构集。**线程的改变只代表了 CPU 执行过程的改变，而没有发生进程所拥有的资源变化**。除了 CPU 之外，**计算机内的软硬件资源的分配与线程无关**，线程只能共享它所属进程的资源。与进程控制表和 PCB 相似，每个线程也有自己的线程控制表 TCB ，而这个 TCB 中所保存的线程状态信息则要比 PCB 表少得多，这些信息主要是相关指针用堆栈（系统栈和用户栈），寄存器中的状态数据。**进程拥有一个完整的虚拟地址空间，不依赖于线程而独立存在；反之，线程是进程的一部分，没有自己的地址空间，与进程内的其他线程一起共享分配给该进程的所有资源**。

线程可以有效地提高系统的执行效率，但并不是在所有计算机系统中都是适用的，如某些很少做进程调度和切换的实时系统。使用线程的好处是有多个任务需要处理机处理时，减少处理机的切换时间；而且，线程的创建和结束所需要的系统开销也比进程的创建和结束要小得多。最适用使用线程的系统是多处理机系统和网络系统或分布式系统。


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


## Object-C有私有方法吗？私有变量呢？
objective-c – 类里面的方法只有两种, 静态方法和实例方法. 这似乎就不是完整的面向对象了,按照OO的原则就是一个对象只暴露有用的东西. 如果没有了私有方法的话, 对于一些小范围的代码重用就不那么顺手了. 在类里面声名一个私有方法
```objc
@interface?Controller?:?NSObject?{?NSString?*something;?}
+?(void)thisIsAStaticMethod;
–?(void)thisIsAnInstanceMethod;
@end
@interface?Controller?(private)?-
(void)thisIsAPrivateMethod;
@end
@private可以用来修饰私有变量

```

在Objective‐C中，所有实例变量默认都是私有的，所有实例方法默认都是公有的

## C和obj-c 如何混用
1. obj-c的编译器处理后缀为m的文件时，可以识别obj-c和c的代码，处理mm文件可以识别obj-c,c,c++代码，但cpp文件必须只能用c/c++代码，而且cpp文件include的头文件中，也不能出现obj-c的代码，因为cpp只是cpp
2. 在mm文件中混用cpp直接使用即可，所以obj-c混cpp不是问题
3. 在cpp中混用obj-c其实就是使用obj-c编写的模块是我们想要的。

如果模块以类实现，那么要按照cpp class的标准写类的定义，头文件中不能出现obj-c的东西，包括#import cocoa的。实现文件中，即类的实现代码中可以使用obj-c的东西，可以import,只是后缀是mm。
如果模块以函数实现，那么头文件要按c的格式声明函数，实现文件中，c++函数内部可以用obj-c，但后缀还是mm或m。
> 总结：只要cpp文件和cpp include的文件中不包含obj-c的东西就可以用了，cpp混用obj-c的关键是使用接口，而不能直接使用 实现代 码，实际上cpp混用的是obj-c编译后的o文件，这个东西其实是无差别的，所以可以用。obj-c的编译器支持cpp

## Objective-C堆和栈的区别？

- 管理方式：对于栈来讲，是由编译器自动管理，无需我们手工控制；对于堆来说，释放工作由程序员控制，容易产生memory leak。

- 申请大小：

**栈**：在Windows下,栈是向低地址扩展的数据结构，是一块连续的内存的区域。这句话的意思是栈顶的地址和栈的最大容量是系统预先规定好的，在 WINDOWS下，栈的大小是2M（也有的说是1M，总之是一个编译时就确定的常数），如果申请的空间超过栈的剩余空间时，将提示overflow。因 此，能从栈获得的空间较小。

**堆**：堆是向高地址扩展的数据结构，是不连续的内存区域。这是由于系统是用链表来存储的空闲内存地址的，自然是不连续的，而链表的遍历方向是由低地址向高地址。堆的大小受限于计算机系统中有效的虚拟内存。由此可见，堆获得的空间比较灵活，也比较大。

- 碎片问题：对于堆来讲，频繁的new/delete势必会造成内存空间的不连续，从而造成大量的碎片，使程序效率降低。对于栈来讲，则不会存在这个问题，因为栈是先进后出的队列，他们是如此的一一对应，以至于永远都不可能有一个内存块从栈中间弹出

- 分配方式：堆都是动态分配的，没有静态分配的堆。栈有2种分配方式：静态分配和动态分配。静态分配是编译器完成的，比如局部变量的分配。动态分配由alloca函数进行分配，但是栈的动态分配和堆是不同的，他的动态分配是由编译器进行释放，无需我们手工实现。

- 分配效率：栈是机器系统提供的数据结构，计算机会在底层对栈提供支持：分配专门的寄存器存放栈的地址，压栈出栈都有专门的指令执行，这就决定了栈的效率比较高。堆则是C/C++函数库提供的，它的机制是很复杂的。

## 简述内存分区情况
1).代码区：存放函数二进制代码

2).数据区：系统运行时申请内存并初始化，系统退出时由系统释放。存放全局变量、静态变量、常量

3).堆区：通过malloc等函数或new等操作符动态申请得到，需程序员手动申请和释放

4).栈区：函数模块内申请，函数结束时由系统自动释放。存放局部变量、函数参数

### 队列和栈有什么区别
队列和栈是两种不同的数据容器。从”数据结构”的角度看，它们都是线性结构，即数据元素之间的关系相同。
队列是一种先进先出的数据结构，它在两端进行操作，一端进行入队列操作，一端进行出列队操作。
栈是一种先进后出的数据结构，它只能在栈顶进行操作，入栈和出栈都在栈顶操作。

## ViewController的didReceiveMemoryWarning怎么被调用：
`[supper didReceiveMemoryWarning];`

## 用预处理指令#define声明一个常数，用以表明1年中有多少秒（忽略闰年问题）

`#define SECONDS_PER_YEAR (60 * 60 * 24 * 365)UL`

我在这想看到几件事情：

\#define 语法的基本知识（例如：不能以分号结束，括号的使用，等等）
懂得预处理器将为你计算常数表达式的值，因此，直接写出你是如何计算一年中有多少秒而不是计算出实际的值，是更清晰而没有代价的。
意识到这个表达式将使一个16位机的整型数溢出-因此要用到长整型符号L,告诉编译器这个常数是的长整型数。
如果你在你的表达式中用到UL（表示无符号长整型），那么你有了一个好的起点。记住，第一印象很重要。

## 写一个”标准"宏MIN ，这个宏输入两个参数并返回较小的一个。
`#define?MIN(A,B)?（（A）?<=?(B)???(A)?:?(B))`
这个测试是为下面的目的而设的：

标识#define在宏中应用的基本知识。这是很重要的，因为直到嵌入(inline)操作符变为标准C的一部分，宏是方便产生嵌入代码的唯一方

法，

对于嵌入式系统来说，为了能达到要求的性能，嵌入代码经常是必须的方法。

三重条件操作符的知识。这个操作符存在C语言中的原因是它使得编译器能产生比 if-then-else 更优化的代码，了解这个用法是很重要的。

懂得在宏中小心地把参数用括号括起来

我也用这个问题开始讨论宏的副作用，例如：当你写下面的代码时会发生什么事？

least?=?MIN(*p++,?b);
结果是：

((*p++)?<=?(b)???(*p++)?:?(*p++))
这个表达式会产生副作用，指针p会作三次++自增操作。

## 关键字const有什么含意？修饰类呢?static的作用,用于类呢?还有extern c的作用
const 意味着"只读"，下面的声明都是什么意思？

```objc
const int a;
int const a;
const int *a;
int * const a;
int const * a const;

```

前两个的作用是一样，a是一个常整型数。
第三个意味着a是一个指向常整型数的指针（也就是，整型数是不可修改的，但指针可以）。
第四个意思a是一个指向整型数的常指针（也就是说，指针指向的整型数是可以修改的，但指针是不可修改的）。
最后一个意味着a是一个指向常整型数的常指针（也就是说，指针指向的整型数是不可修改的，同时指针也是不可修改的）。
> 结论：
关键字const的作用是为给读你代码的人传达非常有用的信息，实际上，声明一个参数为常量是为了告诉了用户这个参数的应用目的。
如果你曾花很多时间清理其它人留下的垃圾，你就会很快学会感谢这点多余的信息。（当然，懂得用const的程序员很少会留下的垃圾让别人来清理的） ?通过给优化器一些附加的信息，使用关键字const也许能产生更紧凑的代码。合理地使用关键字const可以使编译器很自然地保护那些不希望被改变的参数，防止其被无意的代码修改。简而言之，这样可以减少bug的出现。
1. 欲阻止一个变量被改变，可以使用 const 关键字。在定义该 const 变量时，通常需要对它进行初
始化，因为以后就没有机会再去改变它了；
2. 对指针来说，可以指定指针本身为 const，也可以指定指针所指的数据为 const，或二者同时指
定为 const；
3. 在一个函数声明中，const 可以修饰形参，表明它是一个输入参数，在函数内部不能改变其值；
4. 对于类的成员函数，若指定其为 const 类型，则表明其是一个常函数，不能修改类的成员变量；
5. 对于类的成员函数，有时候必须指定其返回值为 const 类型，以使得其返回值不为“左值”。

## 关键字volatile有什么含意?并给出三个不同的例子。
一个定义为 volatile的变量是说这变量可能会被意想不到地改变，这样，编译器就不会去假设这个变量的值了。精确地说就是，优化器在用到这个变量时必须每次都小心地重新读取这个变量的值，而不是使用保存在寄存器里的备份。

下面是volatile变量的几个例子：

并行设备的硬件寄存器（如：状态寄存器）

一个中断服务子程序中会访问到的非自动变量(Non-automatic variables)

多线程应用中被几个任务共享的变量

## 一个参数既可以是const还可以是volatile吗？ 一个指针可以是volatile 吗？解释为什么。
1. 是的。一个例子是只读的状态寄存器。它是volatile因为它可能被意想不到地改变。它是const因为程序不应该试图去修改它。

2. 是的。尽管这并不很常见。一个例子是当一个中服务子程序修该一个指向一个buffer的指针时。

## static 关键字的作用：
1. 函数体内 static 变量的作用范围为该函数体，不同于 auto 变量，该变量的内存只被分配一次，
因此其值在下次调用时仍维持上次的值；
2. 在模块内的 static 全局变量可以被模块内所用函数访问，但不能被模块外其它函数访问；
3. 在模块内的 static 函数只可被这一模块内的其它函数调用，这个函数的使用范围被限制在声明
它的模块内；
4. 在类中的 static 成员变量属于整个类所拥有，对类的所有对象只有一份拷贝；
5. 在类中的 static 成员函数属于整个类所拥有，这个函数不接收 this 指针，因而只能访问类的static 成员变量。

## iOS的系统架构
iOS的系统架构分为（ 核心操作系统层 theCore OS layer ）、（ 核心服务层theCore Services layer ）、（ 媒体层 theMedia layer ）和（ Cocoa 界面服务层 the Cocoa Touch layer ）四个层次。

## cocoa touch框架

iPhone OS 应用程序的基础 Cocoa Touch 框架重用了许多 Mac 系统的成熟模式，但是它更多地专注于触摸的接口和优化。

UIKit 为您提供了在 iPhone OS 上实现图形，事件驱动程序的基本工具，其建立在和 Mac OS X 中一样的 Foundation 框架上，包括文件处理，网络，字符串操作等。

Cocoa Touch 具有和 iPhone 用户接口一致的特殊设计。有了 UIKit，您可以使用 iPhone OS 上的独特的图形接口控件，按钮，以及全屏视图的功能，您还可以使用加速仪和多点触摸手势来控制您的应用。

各色俱全的框架 除了UIKit 外，Cocoa Touch 包含了创建世界一流 iPhone 应用程序需要的所有框架，从三维图形，到专业音效，甚至提供设备访问 API 以控制摄像头，或通过 GPS 获知当前位置。

Cocoa Touch 既包含只需要几行代码就可以完成全部任务的强大的 Objective-C 框架，也在需要时提供基础的 C 语言 API 来直接访问系统。这些框架包括：

Core Animation：通过 Core Animation，您就可以通过一个基于组合独立图层的简单的编程模型来创建丰富的用户体验。

Core Audio：Core Audio 是播放，处理和录制音频的专业技术，能够轻松为您的应用程序添加强大的音频功能。

Core Data：提供了一个面向对象的数据管理解决方案，它易于使用和理解，甚至可处理任何应用或大或小的数据模型。

功能列表：框架分类

下面是 Cocoa Touch 中一小部分可用的框架：

音频和视频：Core Audio ，OpenAL ，Media Library ，AV Foundation

数据管理 ：Core Data ，SQLite

图形和动画 ：Core Animation ，OpenGL ES ，Quartz 2D

网络：Bonjour ，WebKit ，BSD Sockets

用户应用：Address Book ，Core Location ，Map Kit ，Store Kit

## 自动释放池(autoreleasepool)是什么,如何工作
当您向一个对象发送一个autorelease消息时，Cocoa就会将该对象的一个引用放入到最新的自动释放.它仍然是个正当的对象，因此自动释放池定义的作用域内的其它对象可以向它发送消息。当程序执行到作用域结束的位置时，自动释放池就会被释放，池中的所有对象也就被释放。

## Objective-C的优缺点。
objc优点：

1). ?Cateogies

2). ?Posing

3). 动态识别

4).指标计算

5).弹性讯息传递

6).不是一个过度复杂的 C 衍生语言

7).Objective-C 与 C++ 可混合编程

objc缺点:

1).不支援命名空间

2).不支持运算符重载

3).不支持多重继承

4).使用动态运行时类型，所有的方法都是函数调用，所以很多编译时优化方法都用不到。（如内联函数等），性能低劣。

## sprintf,strcpy,memcpy使用上有什么要注意的地方。
1). sprintf是格式化函数。将一段数据通过特定的格式，格式化到一个字符串缓冲区中去。sprintf格式化的函数的长度不可控，有可能格式化后的字符串会超出缓冲区的大小，造成溢出。

2).strcpy是一个字符串拷贝的函数，它的函数原型为strcpy(char *dst, const char *src

将src开始的一段字符串拷贝到dst开始的内存中去，结束的标志符号为 ‘\0'，由于拷贝的长度不是由我们自己控制的，所以这个字符串拷贝很容易出错。

3). memcpy是具备字符串拷贝功能的函数，这是一个内存拷贝函数，它的函数原型为memcpy(char *dst, const char* src, unsigned int len);将长度为len的一段内存，从src拷贝到dst中去，这个函数的长度可控。但是会有内存叠加的问题。

## http和scoket通信的区别
http是客户端用http协议进行请求，发送请求时候需要封装http请求头，并绑定请求的数据，服务器一般有web服务器配合（当然也非绝对）。 http请求方式为客户端主动发起请求，服务器才能给响应，一次请求完毕后则断开连接，以节省资源。服务器不能主动给客户端响应（除非采取http长连接 技术）。iphone主要使用类是NSUrlConnection。

scoket是客户端跟服务器直接使用socket“套接字”进行连接，并没有规定连接后断开，所以客户端和服务器可以保持连接通道，双方 都可以主动发送数据。一般在游戏开发或股票开发这种要求即时性很强并且保持发送数据量比较大的场合使用。主要使用类是CFSocketRef。

## iOS中socket使用
Socket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口（API），通过Socket，我们才能使用TCP/IP协议。

http协议 对应于应用层
tcp协议 对应于传输层
ip协议 对应于网络层
三者本质上没有可比性。 何况HTTP协议是基于TCP连接的。

TCP/IP是传输层协议，主要解决数据如何在网络中传输；而HTTP是应用层协议，主要解决如何包装数据。

我 们在传输数据时，可以只使用传输层（TCP/IP），但是那样的话，由于没有应用层，便无法识别数据内容，如果想要使传输的数据有意义，则必须使用应用层 协议，应用层协议很多，有HTTP、FTP、TELNET等等，也可以自己定义应用层协议。WEB使用HTTP作传输层协议，以封装HTTP文本信息，然 后使用TCP/IP做传输层协议将它发送到网络上。

**SOCKET原理**
1. 套接字（socket）概念
套接字（socket）是通信的基石，是支持TCP/IP协议的网络通信的基本操作单元。它是网络通信过程中端点的抽象表示，包含进行网络通信必须的五种信息：连接使用的协议，本地主机的IP地址，本地进程的协议端口，远地主机的IP地址，远地进程的协议端口。

应 用层通过传输层进行数据通信时，TCP会遇到同时为多个应用程序进程提供并发服务的问题。多个TCP连接或多个应用程序进程可能需要通过同一个 TCP协议端口传输数据。为了区别不同的应用程序进程和连接，许多计算机操作系统为应用程序与TCP／IP协议交互提供了套接字(Socket)接口。应 用层可以和传输层通过Socket接口，区分来自不同应用程序进程或网络连接的通信，实现数据传输的并发服务。

2. 建立socket连接
建立Socket连接至少需要一对套接字，其中一个运行于客户端，称为ClientSocket，另一个运行于服务器端，称为ServerSocket。

套接字之间的连接过程分为三个步骤：服务器监听，客户端请求，连接确认。

服务器监听：服务器端套接字并不定位具体的客户端套接字，而是处于等待连接的状态，实时监控网络状态，等待客户端的连接请求。

客户端请求：指客户端的套接字提出连接请求，要连接的目标是服务器端的套接字。为此，客户端的套接字必须首先描述它要连接的服务器的套接字，指出服务器端套接字的地址和端口号，然后就向服务器端套接字提出连接请求。

连 接确认：当服务器端套接字监听到或者说接收到客户端套接字的连接请求时，就响应客户端套接字的请求，建立一个新的线程，把服务器端套接字的描述发给客户 端，一旦客户端确认了此描述，双方就正式建立连接。而服务器端套接字继续处于监听状态，继续接收其他客户端套接字的连接请求。

3. SOCKET连接与TCP连接
创建Socket连接时，可以指定使用的传输层协议，Socket可以支持不同的传输层协议（TCP或UDP），当使用TCP协议进行连接时，该Socket连接就是一个TCP连接。

4. Socket连接与HTTP连接
由 于通常情况下Socket连接就是TCP连接，因此Socket连接一旦建立，通信双方即可开始相互发送数据内容，直到双方连接断开。但在实际网络应用 中，客户端到服务器之间的通信往往需要穿越多个中间节点，例如路由器、网关、防火墙等，大部分防火墙默认会关闭长时间处于非活跃状态的连接而导致 Socket 连接断连，因此需要通过轮询告诉网络，该连接处于活跃状态。

而HTTP连接使用的是“请求—响应”的方式，不仅在请求时需要先建立连接，而且需要客户端向服务器发出请求后，服务器端才能回复数据。

很 多情况下，需要服务器端主动向客户端推送数据，保持客户端与服务器数据的实时与同步。此时若双方建立的是Socket连接，服务器就可以直接将数据传送给 客户端；若双方建立的是HTTP连接，则服务器需要等到客户端发送一次请求后才能将数据传回给客户端，因此，客户端定时向服务器端发送连接请求，不仅可以 保持在线，同时也是在“询问”服务器是否有新的数据，如果有就将数据传给客户端。

[Socket使用简明教程－ AsyncSocket](http://my.oschina.net/joanfen/blog/287238)

### CFSocket使用有哪几个步骤。
创建 Socket 的上下文；创建 Socket ；配置要访问的服务器信息；封装服务器信息；连接服务器；
### Core Foundation中提供了哪几种操作Socket的方法？
CFNetwork 、 CFSocket 和 BSD Socket
### HTTP协议中，POST和GET的区别是什么？
1).GET 方法

GET 方法提交数据不安全，数据置于请求行，客户端地址栏可见;

GET 方法提交的数据大小有限

GET 方法不可以设置书签

2).POST 方法

POST 方法提交数据安全，数据置于消息主体内，客户端不可见

POST 方法提交的数据大小没有限制

POST 方法可以设置书签

1. GET请求的数据会附在URL之后（就是把数据放置在HTTP协议头中），以?分割URL和传输数据，参数之间以&相连，如：login.action?name=hyddd&password=idontknow&verify=%E4%BD%A0%E5%A5%BD。如果数据是英文字母/数字，原样发送，如果是空格，转换为+，如果是中文/其他字符，则直接把字符串用BASE64加密，得出如：%E4%BD%A0%E5%A5%BD，其中％XX中的XX为该符号以16进制表示的ASCII。
　　POST把提交的数据则放置在是HTTP包的包体中。

2. ”GET方式提交的数据最多只能是1024字节，理论上POST没有限制，可传较大量的数据，IIS4中最大为80KB，IIS5中为100KB”？？！

　　以上这句是我从其他文章转过来的，其实这样说是错误的，不准确的：

　　(1).首先是”GET方式提交的数据最多只能是1024字节”，因为GET是通过URL提交数据，那么GET可提交的数据量就跟URL的长度有直接关系了。而实际上，URL不存在参数上限的问题，HTTP协议规范没有对URL长度进行限制。这个限制是特定的浏览器及服务器对它的限制。IE对URL长度的限制是2083字节(2K+35)。对于其他浏览器，如Netscape、FireFox等，理论上没有长度限制，其限制取决于操作系统的支持。

　　注意这是限制是整个URL长度，而不仅仅是你的参数值数据长度。[见参考资料5]

　　(2).理论上讲，POST是没有大小限制的，HTTP协议规范也没有进行大小限制，说“POST数据量存在80K/100K的大小限制”是不准确的，POST数据是没有限制的，起限制作用的是服务器的处理程序的处理能力。

3.在ASP中，服务端获取GET请求参数用Request.QueryString，获取POST请求参数用Request.Form。在JSP中，用request.getParameter(\”XXXX\”)来获取，虽然jsp中也有request.getQueryString()方法，但使用起来比较麻烦，比如：传一个test.jsp?name=hyddd&password=hyddd，用request.getQueryString()得到的是：name=hyddd&password=hyddd。在PHP中，可以用GET和_POST分别获取GET和POST中的数据，而REQUEST则可以获取GET和POST两种请求中的数据。值得注意的是，JSP中使用request和PHP中使用_REQUEST都会有隐患，这个下次再写个文章总结。

4.POST的安全性要比GET的安全性高。注意：这里所说的安全性和上面GET提到的“安全”不是同个概念。上面“安全”的含义仅仅是不作数据修改，而这里安全的含义是真正的Security的含义，比如：通过GET提交数据，用户名和密码将明文出现在URL上，因为(1)登录页面有可能被浏览器缓存，(2)其他人查看浏览器的历史纪录，那么别人就可以拿到你的账号和密码了，除此之外，使用GET提交数据还可能会造成Cross-site request forgery攻击。

总结一下，Get是向服务器发索取数据的一种请求，而Post是向服务器提交数据的一种请求，在FORM（表单）中，Method默认为”GET”，实质上，GET和POST只是发送机制不同，并不是一个取一个发！

## TCP和UDP的区别
TCP全称是Transmission Control Protocol，中文名为传输控制协议，它可以提供可靠的、面向连接的网络数据传递服务。传输控制协议主要包含下列任务和功能：

* 确保IP数据报的成功传递。

* 对程序发送的大块数据进行分段和重组。

* 确保正确排序及按顺序传递分段的数据。

* 通过计算校验和，进行传输数据的完整性检查。

TCP提供的是面向连接的、可靠的数据流传输，而UDP提供的是非面向连接的、不可靠的数据流传输。

简单的说，TCP注重数据安全，而UDP数据传输快点，但安全性一般

## 你了解svn,cvs等版本控制工具么？
版本控制 svn,cvs 是两种版控制的器,需要配套相关的svn，cvs服务器。

scm是xcode里配置版本控制的地方。版本控制的原理就是a和b同时开发一个项目，a写完当天的代码之后把代码提交给服务器，b要做的时候先从服务器得到最新版本，就可以接着做。 如果a和b都要提交给服务器，并且同时修改了同一个方法，就会产生代码冲突，如果a先提交，那么b提交时，服务器可以提示冲突的代码，b可以清晰的看到，并做出相应的修改或融合后再提交到服务器。

## 静态链接库
此为.a文件，相当于java里的jar包，把一些类编译到一个包中，在不同的工程中如果导入此文件就可以使用里面的类，具体使用依然是#import “ xx.h”。

## fmmpeg框架
音视频编解码框架，内部使用UDP协议针对流媒体开发，内部开辟了六个端口来接受流媒体数据，完成快速接受之目的。

## iPhone OS主要提供了几种播放音频的方法？
SystemSound Services

AVAudioPlayer 类

Audio Queue Services

OpenAL

## 使用AVAudioPlayer类调用哪个框架、使用步骤？
AVFoundation.framework

步骤：配置 AVAudioPlayer 对象；

实现 AVAudioPlayer 类的委托方法；

控制 AVAudioPlayer 类的对象；

监控音量水平；

回放进度和拖拽播放。

## 有哪几种手势通知方法、写清楚方法名？
```objc
-(void)touchesBegan:(NSSet*)touchedwithEvent:(UIEvent*)event;

-(void)touchesMoved:(NSSet*)touched withEvent:(UIEvent*)event;

-(void)touchesEnded:(NSSet*)touchedwithEvent:(UIEvent*)event;

-(void)touchesCanceled:(NSSet*)touchedwithEvent:(UIEvent*)event;
```

## 320框架
 ui框架，导入320工程作为框架包如同添加一个普通框架一样。cover(open) ?flower框架 (2d 仿射技术)，内部核心类是CATransform3D.

## 什么是沙盒模型？哪些操作是属于私有api范畴?
某个iphone工程进行文件操作有此工程对应的指定的位置，不能逾越。

iphone沙箱模型的有四个文件夹documents，tmp，app，Library，永久数据存储一般放documents文件夹，得到模拟器的路径的可使用NSHomeDirectory()方法。Nsuserdefaults保存的文件在tmp文件夹里。

## 在一个对象的方法里面：self.name= “object”；和 name =”object” 有什么不同吗?
self.name =”object”：会调用对象的setName()方法；

name = “object”：会直接把object赋值给当前对象的name属性。

## 请简要说明viewDidLoad和viewDidUnload何时调用
viewDidLoad在view从nib文件初始化时调用，loadView在controller的view为nil时调用。此方法在编程实现view时调用，view控制器默认会注册memory warning notification，当view controller的任何view没有用的时候，viewDidUnload会被调用，在这里实现将retain的view release，如果是retain的IBOutlet view 属性则不要在这里release，IBOutlet会负责release 。

## 控件主要响应3种事件
- 基于触摸的事件
- 基于值的事件
- 基于编辑的事件。

## xib文件的构成分为哪3个图标？都具有什么功能。
File’s Owner 是所有 nib 文件中的每个图标，它表示从磁盘加载 nib 文件的对象；

First Responder 就是用户当前正在与之交互的对象；

View 显示用户界面；完成用户交互；是 UIView 类或其子类。
## 如何高性能的给UIImageView加个圆角？（不准说layer.cornerRadius!）
我觉得应该是使用Quartz2D直接绘制图片,得把这个看看。
步骤：
1. 创建目标大小(cropWidth，cropHeight)的画布。
2. 使用UIImage的drawInRect方法进行绘制的时候，指定rect为(-x，-y，width，height)。
3. 从画布中得到裁剪后的图像。

```objc
- (UIImage*)cropImageWithRect:(CGRect)cropRect
{
    CGRect drawRect = CGRectMake(-cropRect.origin.x , -cropRect.origin.y, self.size.width * self.scale, self.size.height * self.scale);

    UIGraphicsBeginImageContext(cropRect.size);
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextClearRect(context, CGRectMake(0, 0, cropRect.size.width, cropRect.size.height));

    [self drawInRect:drawRect];

    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();

    return image;
}

@end
```

## 使用drawRect有什么影响？（这个可深可浅，你至少得用过。。）
drawRect方法依赖Core Graphics框架来进行自定义的绘制，但这种方法主要的缺点就是它处理touch事件的方式：每次按钮被点击后，都会用setNeddsDisplay进行强制重绘；而且不止一次，每次单点事件触发两次执行。这样的话从性能的角度来说，对CPU和内存来说都是欠佳的。特别是如果在我们的界面上有多个这样的UIButton实例。

## 简述视图控件器的生命周期。
loadView 尽管不直接调用该方法，如多手动创建自己的视图，那么应该覆盖这个方法并将它们赋值给试图控制器的 view 属性。

viewDidLoad 只有在视图控制器将其视图载入到内存之后才调用该方法，这是执行任何其他初始化操作的入口。

viewDidUnload 当试图控制器从内存释放自己的方法的时候调用，用于清楚那些可能已经在试图控制器中创建的对象。

viewVillAppear 当试图将要添加到窗口中并且还不可见的时候或者上层视图移出图层后本视图变成顶级视图时调用该方法，用于执行诸如改变视图方向等的操作。实现该方法时确保调用 [super viewWillAppear:

viewDidAppear 当视图添加到窗口中以后或者上层视图移出图层后本视图变成顶级视图时调用，用于放置那些需要在视图显示后执行的代码。确保调用 [super viewDidAppear：] 。

## 动画有基本类型有哪几种；表视图有哪几种基本样式。
动画有两种基本类型：隐式动画和显式动画。

## Cocoa Touch提供了哪几种Core Animation过渡类型？
Cocoa Touch 提供了 4 种 Core Animation 过渡类型，分别为：交叉淡化、推挤、显示和覆盖。

## UIView与CLayer有什么区别？
1).UIView是iOS系统中界面元素的基础，所有的界面元素都继承自它。它本身完全是由CoreAnimation来实现的 （Mac下似乎不是这样）。它真正的绘图部分，是由一个叫CALayer（Core Animation Layer）的类来管理。 UIView本身，更像是一个CALayer的管理器，访问它的跟绘图和跟坐标有关的属性，例如frame，bounds等 等，实际上内部都是在访问它所包含的CALayer的相关属性。

2).UIView有个layer属性，可以返回它的主CALayer实例，UIView有一个layerClass方法，返回主layer所使用的 类，UIView的子类，可以通过重载这个方法，来让UIView使用不同的CALayer来显示，例如通过
```objc
- (class) layerClass {

         return ([CAEAGLLayer class]);
    }
```
使某个UIView的子类使用GL来进行绘制。

3).UIView 的 CALayer 类似 UIView 的子 View 树形结构，也可以向它的 layer 上添加子layer ，来完成某些特殊的表示。即 CALayer 层是可以嵌套的。
```objc
grayCover = [[CALayer alloc] init];

    grayCover.backgroundColor = [[[UIColor blackColor] colorWithAlphaComponent:0.2] CGColor];

    [self.layer addSubLayer: grayCover];
```
在目标View上敷上一层黑色的透明薄膜。
4).UIView 的 layer 树形在系统内部，被维护着三份 copy 。
- 逻辑树，这里是代码可以操纵的；
- 动画树，是一个中间层，系统就在这一层上更改属性，进行各种渲染操作；
- 显示树，其内容就是当前正被显示在屏幕上得内容。

这三棵树的逻辑结构都是一样的，区别只有各自的属性。

5).动画的运作：对 UIView 的 subLayer （非主 Layer ）属性进行更改，系统将自动进行动画生成，动画持续时间的缺省值似乎是 0.5 秒。

6).坐标系统： CALayer 的坐标系统比 UIView 多了一个 anchorPoint 属性，使用CGPoint 结构表示，值域是 0~1 ，是个比例值。这个点是各种图形变换的坐标原点，同时会更改 layer 的 position 的位置，它的缺省值是 {0.5,0.5} ，即在 layer 的中央。

7).渲染：当更新层，改变不能立即显示在屏幕上。当所有的层都准备好时，可以调用setNeedsDisplay 方法来重绘显示。

8).变换：要在一个层中添加一个 3D 或仿射变换，可以分别设置层的 transform 或affineTransform 属性。

9).变形： Quartz Core 的渲染能力，使二维图像可以被自由操纵，就好像是三维的。图像可以在一个三维坐标系中以任意角度被旋转，缩放和倾斜。 CATransform3D 的一套方法提供了一些魔术般的变换效果。

## Quatrz 2D的绘图功能的三个核心概念是什么并简述其作用。
上下文：主要用于描述图形写入哪里；
路径：是在图层上绘制的内容；
状态：用于保存配置变换的值、填充和轮廓， alpha 值等。

## 解析XML文件有哪几种方式？
以 DOM 方式解析 XML 文件；以 SAX 方式解析 XML 文件；

## tableView 的重用机制
UITableView 通过重用单元格来达到节省内存的目的: 通过为每个单元格指定一个重用标识符(reuseIdentifier),即指定了单元格的种类,以及当单元格滚出屏幕时,允许恢复单元格以便重用.对于不同种类的单元格使用不同的ID,对于简单的表格,一个标识符就够了.

## 描述一个你遇到过的retain cycle例子
block中的循环引用：一个viewController
```objc
@property (nonatomic,strong)HttpRequestHandler * handler;
   @property (nonatomic,strong)NSData          *data;
   _handler = [httpRequestHandler sharedManager];
   [ downloadData:^(id responseData){
       _data = responseData;
   }];
```

self 拥有_handler, _handler 拥有block, block拥有self（因为使用了self的_data属性，block会copy 一份self）
解决方法：
```objc
__weak typedof(self)weakSelf = self
    [ downloadData:^(id responseData){
        weakSelf.data = responseData;
    }];
```

## 为什么其他语言里叫函数调用， objective c里则是给对象发消息（或者谈下对runtime的理解）
先来看看怎么理解发送消息的含义：

曾经觉得Objc特别方便上手，面对着 Cocoa 中大量 API，只知道简单的查文档和调用。还记得初学 Objective-C 时把[receiver message]当成简单的方法调用，而无视了“发送消息”这句话的深刻含义。于是[receiver message]会被编译器转化为：
objc_msgSend(receiver, selector)
如果消息含有参数，则为：
`objc_msgSend(receiver, selector, arg1, arg2, ...)`

如果消息的接收者能够找到对应的selector，那么就相当于直接执行了接收者这个对象的特定方法；否则，消息要么被转发，或是临时向接收者动态添加这个selector对应的实现内容，要么就干脆玩完崩溃掉。

现在可以看出[receiver message]真的不是一个简简单单的方法调用。因为这只是在编译阶段确定了要向接收者发送message这条消息，而receive将要如何响应这条消息，那就要看运行时发生的情况来决定了。

Objective-C 的 Runtime 铸就了它动态语言的特性，这些深层次的知识虽然平时写代码用的少一些，但是却是每个 Objc 程序员需要了解的。

Objc Runtime使得C具有了面向对象能力，在程序运行时创建，检查，修改类、对象和它们的方法。可以使用runtime的一系列方法实现。

顺便附上OC中一个类的数据结构 /usr/include/objc/runtime.h

```objc
struct objc_class {
    Class isa OBJC_ISA_AVAILABILITY; //isa指针指向Meta Class，因为Objc的类的本身也是一个Object，为了处理这个关系，r       untime就创造了Meta Class，当给类发送[NSObject alloc]这样消息时，实际上是把这个消息发给了Class Object

    #if !__OBJC2__
    Class super_class OBJC2_UNAVAILABLE; // 父类
    const char *name OBJC2_UNAVAILABLE; // 类名
    long version OBJC2_UNAVAILABLE; // 类的版本信息，默认为0
    long info OBJC2_UNAVAILABLE; // 类信息，供运行期使用的一些位标识
    long instance_size OBJC2_UNAVAILABLE; // 该类的实例变量大小
    struct objc_ivar_list *ivars OBJC2_UNAVAILABLE; // 该类的成员变量链表
    struct objc_method_list **methodLists OBJC2_UNAVAILABLE; // 方法定义的链表
    struct objc_cache *cache OBJC2_UNAVAILABLE; // 方法缓存，对象接到一个消息会根据isa指针查找消息对象，这时会在method       Lists中遍历，如果cache了，常用的方法调用时就能够提高调用的效率。
    struct objc_protocol_list *protocols OBJC2_UNAVAILABLE; // 协议链表
    #endif

    } OBJC2_UNAVAILABLE;
```

OC中一个类的对象实例的数据结构（/usr/include/objc/objc.h）:
```objc
typedef struct objc_class *Class;

/// Represents an instance of a class.

struct objc_object {

    Class isa  OBJC_ISA_AVAILABILITY;

};

/// A pointer to an instance of a class.

typedef struct objc_object *id;
```

向object发送消息时，Runtime库会根据object的isa指针找到这个实例object所属于的类，然后在类的方法列表以及父类方法列表寻找对应的方法运行。id是一个objc_object结构类型的指针，这个类型的对象能够转换成任何一种对象。

然后再来看看消息发送的函数：objc_msgSend函数

在引言中已经对objc_msgSend进行了一点介绍，看起来像是objc_msgSend返回了数据，其实objc_msgSend从不返回数据而是你的方法被调用后返回了数据。下面详细叙述下消息发送步骤：

检测这个 selector 是不是要忽略的。比如 Mac OS X 开发，有了垃圾回收就不理会 retain,release 这些函数了。
检测这个 target 是不是 nil 对象。ObjC 的特性是允许对一个 nil 对象执行任何一个方法不会 Crash，因为会被忽略掉。
如果上面两个都过了，那就开始查找这个类的 IMP，先从 cache 里面找，完了找得到就跳到对应的函数去执行。
如果 cache 找不到就找一下方法分发表。
如果分发表找不到就到超类的分发表去找，一直找，直到找到NSObject类为止。
如果还找不到就要开始进入动态方法解析了，后面会提到。

后面还有：
动态方法解析resolveThisMethodDynamically
消息转发forwardingTargetForSelector

## SDWebImage里面给UIImageView加载图片的逻辑是什么样的？
### options所有选项：
```objc
//失败后重试
     SDWebImageRetryFailed = 1 << 0,

     //UI交互期间开始下载，导致延迟下载比如UIScrollView减速。
     SDWebImageLowPriority = 1 << 1,

     //只进行内存缓存
     SDWebImageCacheMemoryOnly = 1 << 2,

     //这个标志可以渐进式下载,显示的图像是逐步在下载
     SDWebImageProgressiveDownload = 1 << 3,

     //刷新缓存
     SDWebImageRefreshCached = 1 << 4,

     //后台下载
     SDWebImageContinueInBackground = 1 << 5,

     //NSMutableURLRequest.HTTPShouldHandleCookies = YES;

     SDWebImageHandleCookies = 1 << 6,

     //允许使用无效的SSL证书
     //SDWebImageAllowInvalidSSLCertificates = 1 << 7,

     //优先下载
     SDWebImageHighPriority = 1 << 8,

     //延迟占位符
     SDWebImageDelayPlaceholder = 1 << 9,

     //改变动画形象
     SDWebImageTransformAnimatedImage = 1 << 10,
```

### SDWebImage内部实现过程
1. 入口 setImageWithURL:placeholderImage:options: 会先把 placeholderImage 显示，然后 SDWebImageManager 根据 URL 开始处理图片。
2. 进入 SDWebImageManager-downloadWithURL:delegate:options:userInfo:，交给 SDImageCache 从缓存查找图片是否已经下载        queryDiskCacheForKey:delegate:userInfo:.
3. 先从内存图片缓存查找是否有图片，如果内存中已经有图片缓存，SDImageCacheDelegate 回调 imageCache:didFindImage:forKey:userInfo: 到 SDWebImageManager。
4. SDWebImageManagerDelegate 回调 webImageManager:didFinishWithImage: 到 UIImageView+WebCache 等前端展示图片。
5. 如果内存缓存中没有，生成 NSInvocationOperation 添加到队列开始从硬盘查找图片是否已经缓存。
6. 根据 URLKey 在硬盘缓存目录下尝试读取图片文件。这一步是在 NSOperation 进行的操作，所以回主线程进行结果回调 notifyDelegate:。
7. 如果上一操作从硬盘读取到了图片，将图片添加到内存缓存中（如果空闲内存过小，会先清空内存缓存）。SDImageCacheDelegate 回调 imageCache:didFindImage:forKey:userInfo:。进而回调展示图片。
8. 如果从硬盘缓存目录读取不到图片，说明所有缓存都不存在该图片，需要下载图片，回调 imageCache:didNotFindImageForKey:userInfo:。
9. 共享或重新生成一个下载器 SDWebImageDownloader 开始下载图片。
10. 图片下载由 NSURLConnection 来做，实现相关 delegate 来判断图片下载中、下载完成和下载失败。
11. connection:didReceiveData: 中利用 ImageIO 做了按图片下载进度加载效果。
12. connectionDidFinishLoading: 数据下载完成后交给 SDWebImageDecoder 做图片解码处理。
13. 图片解码处理在一个 NSOperationQueue 完成，不会拖慢主线程 UI。如果有需要对下载的图片进行二次处理，最好也在这里完成，效率会好很多。
14. 在主线程 notifyDelegateOnMainThreadWithInfo: 宣告解码完成，imageDecoder:didFinishDecodingImage:userInfo: 回调给 SDWebImageDownloader。
15. imageDownloader:didFinishWithImage: 回调给 SDWebImageManager 告知图片下载完成。
16. 通知所有的 downloadDelegates 下载完成，回调给需要的地方展示图片。
17. 将图片保存到 SDImageCache 中，内存缓存和硬盘缓存同时保存。写文件到硬盘也在以单独 NSInvocationOperation 完成，避免拖慢主线程。
18. SDImageCache 在初始化的时候会注册一些消息通知，在内存警告或退到后台的时候清理内存图片缓存，应用结束的时候清理过期图片。
19. SDWI 也提供了 UIButton+WebCache 和 MKAnnotationView+WebCache，方便使用。
20. SDWebImagePrefetcher 可以预先下载图片，方便后续使用。

从上面流程可以看出，当你调用setImageWithURL:方法的时候，他会自动去给你干这么多事，当你需要在某一具体时刻做事情的时候，你可以覆盖这些方法。比如在下载某个图片的过程中要响应一个事件，就覆盖这个方法：
```objc
//覆盖方法，指哪打哪，这个方法是下载imagePath2的时候响应
    SDWebImageManager *manager = [SDWebImageManager sharedManager];

    [manager downloadImageWithURL:imagePath2 options:SDWebImageRetryFailed progress:^(NSInteger receivedSize, NSInteger expectedSize) {

        NSLog(@"显示当前进度");

    } completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {

        NSLog(@"下载完成");
    }];
```

### SDWebImage库的作用
通过对UIImageView的类别扩展来实现异步加载替换图片的工作。

主要用到的对象：
1. UIImageView (WebCache)类别，入口封装，实现读取图片完成后的回调
2. SDWebImageManager，对图片进行管理的中转站，记录那些图片正在读取。
向下层读取Cache（调用SDImageCache），或者向网络读取对象（调用SDWebImageDownloader） 。
实现SDImageCache和SDWebImageDownloader的回调。
3. SDImageCache，根据URL的MD5摘要对图片进行存储和读取（实现存在内存中或者存在硬盘上两种实现）
实现图片和内存清理工作。
4. SDWebImageDownloader，根据URL向网络读取数据（实现部分读取和全部读取后再通知回调两种方式）

其他类：
SDWebImageDecoder，异步对图像进行了一次解压⋯⋯
1. SDImageCache是怎么做数据管理的?

SDImageCache分两个部分，一个是内存层面的，一个是硬盘层面的。内存层面的相当是个缓存器，以Key-Value的形式存储图片。当内存不够的时候会清除所有缓存图片。用搜索文件系统的方式做管理，文件替换方式是以时间为单位，剔除时间大于一周的图片文件。当SDWebImageManager向SDImageCache要资源时，先搜索内存层面的数据，如果有直接返回，没有的话去访问磁盘，将图片从磁盘读取出来，然后做Decoder，将图片对象放到内存层面做备份，再返回调用层。

2. 为啥必须做Decoder?
由于UIImage的imageWithData函数是每次画图的时候才将Data解压成ARGB的图像，所以在每次画图的时候，会有一个解压操作，这样效率很低，但是只有瞬时的内存需求。为了提高效率通过SDWebImageDecoder将包装在Data下的资源解压，然后画在另外一张图片上，这样这张新图片就不再需要重复解压了。
这种做法是典型的空间换时间的做法。


## 设计个简单的图片内存缓存器（移除策略是一定要说的）
图片的内存缓存，可以考虑将图片数据保存到一个数据模型中。所以在程序运行时这个模型都存在内存中。
移除策略：释放数据模型对象。

## loadView是干嘛用的？
当你访问一个ViewController的view属性时，如果此时view的值是nil，那么，ViewController就会自动调用loadView这个方法。这个方法就会加载或者创建一个view对象，赋值给view属性。
loadView默认做的事情是：如果此ViewController存在一个对应的nib文件，那么就加载这个nib。否则，就创建一个UIView对象。

如果你用Interface Builder来创建界面，那么不应该重载这个方法。

如果你想自己创建view对象，那么可以重载这个方法。此时你需要自己给view属性赋值。你自定义的方法不应该调用super。如果你需要对view做一些其他的定制操作，在viewDidLoad里面去做。
----
根据上面的文档可以知道，有两种情况：

1. 如果你用了nib文件，重载这个方法就没有太大意义。因为loadView的作用就是加载nib。如果你重载了这个方法不调用super，那么nib文件就不会被加载。如果调用了super，那么view已经加载完了，你需要做的其他事情在viewDidLoad里面做更合适。

2. 如果你没有用nib，这个方法默认就是创建一个空的view对象。如果你想自己控制view对象的创建，例如创建一个特殊尺寸的view，那么可以重载这个方法，自己创建一个UIView对象，然后指定 self.view = myView; 但这种情况也没有必要调用super，因为反正你也不需要在super方法里面创建的view对象。如果调用了super，那么就是浪费了一些资源而已

## viewWillLayoutSubView
横竖屏切换的时候，系统会响应一些函数，其中 viewWillLayoutSubviews 和 viewDidLayoutSubviews。
```objc
- (void)viewWillLayoutSubviews

{

     [self _shouldRotateToOrientation:(UIDeviceOrientation)[UIApplication sharedApplication].statusBarOrientation];

}

-(void)_shouldRotateToOrientation:(UIDeviceOrientation)orientation {
        if (orientation == UIDeviceOrientationPortrait ||orientation ==
                UIDeviceOrientationPortraitUpsideDown) {
          // 竖屏
}
else {
         // 横屏
    }
}
```
通过上述一个函数就知道横竖屏切换的接口了。
注意：viewWillLayoutSubviews只能用在ViewController里面，在view里面没有响应。

## 你实现过多线程的Core Data么？NSPersistentStoreCoordinator，NSManagedObjectContext和NSManagedObject中的哪些需要在线程中创建或者传递？你是用什么样的策略来实现的？
https://onevcat.com/2014/03/common-background-practices/
## Core开头的系列的内容。是否使用过CoreAnimation和CoreGraphics。UI框架和CA，CG框架的联系是什么？分别用CA和CG做过些什么动画或者图像上的内容。（有需要的话还可以涉及Quartz的一些内容）
https://onevcat.com/2013/04/using-blending-in-ios/

## 你实现过一个框架或者库以供别人使用么？如果有，请谈一谈构建框架或者库时候的经验；如果没有，请设想和设计框架的public的API，并指出大概需要如何做、需要注意一些什么方面，来使别人容易地使用你的框架。

## 深浅复制和属性为copy，strong值的变化问题
浅复制：只复制指向对象的指针，而不复制引用对象本身。对于浅复制来说，A和A_copy指向的是同一个内存资源，复制的只不个是一个指针，对象本身资源还是只有一份，那如果我们对A_copy执行了修改操作，那么发现A引用的对象同样被修改了。深复制就好理解了，内存中存在了两份独立对象本身。

在Objective-C中并不是所有的对象都支持Copy，MutableCopy，遵守NSCopying协议的类才可以发送Copy消息，遵守NSMutableCopying协议的类才可以发送MutableCopy消息。
```objc
[immutableObject copy] // 浅拷贝
[immutableObject mutableCopy] //深拷贝
[mutableObject copy] //深拷贝
[mutableObject mutableCopy] //深拷贝
```

属性设为copy,指定此属性的值不可更改，防止可变字符串更改自身的值的时候不会影响到对象属性（如NSString,NSArray,NSDictionary）的值。strong此属性的指会随着变化而变化。copy是内容拷贝，strong是指针拷贝。

## NSTimer创建后，会在哪个线程运行。
用scheduledTimerWithTimeInterval创建的，在哪个线程创建就会被加入哪个线程的RunLoop中就运行在哪个线程。

自己创建的Timer，加入到哪个线程的RunLoop中就运行在哪个线程。

## KVO，NSNotification，delegate及block区别
KVO就是cocoa框架实现的观察者模式，一般同KVC搭配使用，通过KVO可以监测一个值的变化，比如View的高度变化。是一对多的关系，一个值的变化会通知所有的观察者。

NSNotification是通知，也是一对多的使用场景。在某些情况下，KVO和NSNotification是一样的，都是状态变化之后告知对方。NSNotification的特点，就是需要被观察者先主动发出通知，然后观察者注册监听后再来进行响应，比KVO多了发送通知的一步，但是其优点是监听不局限于属性的变化，还可以对多种多样的状态变化进行监听，监听范围广，使用也更灵活。

delegate 是代理，就是我不想做的事情交给别人做。比如狗需要吃饭，就通过delegate通知主人，主人就会给他做饭、盛饭、倒水，这些操作，这些狗都不需要关心，只需要调用delegate（代理人）就可以了，由其他类完成所需要的操作。所以delegate是一对一关系。

block是delegate的另一种形式，是函数式编程的一种形式。使用场景跟delegate一样，相比delegate更灵活，而且代理的实现更直观。

KVO一般的使用场景是数据，需求是数据变化，比如股票价格变化，我们一般使用KVO（观察者模式）。delegate一般的使用场景是行为，需求是需要别人帮我做一件事情，比如买卖股票，我们一般使用delegate。Notification一般是进行全局通知，比如利好消息一出，通知大家去买入。delegate是强关联，就是委托和代理双方互相知道，你委托别人买股票你就需要知道经纪人，经纪人也不要知道自己的顾客。Notification是弱关联，利好消息发出，你不需要知道是谁发的也可以做出相应的反应，同理发消息的人也不需要知道接收的人也可以正常发出消息。

## 如何让计时器(NSTimer)调用一个类方法
计时器只能调用实例方法，但是可以在这个实例方法里面调用静态方法。

使用计时器需要注意，计时器一定要加入RunLoop中，并且选好model才能运行。scheduledTimerWithTimeInterval方法创建一个计时器并加入到RunLoop中所以可以直接使用。

如果计时器的repeats选择YES说明这个计时器会重复执行，一定要在合适的时机调用计时器的invalid。不能在dealloc中调用，因为一旦设置为repeats 为yes，计时器会强持有self，导致dealloc永远不会被调用，这个类就永远无法被释放。比如可以在viewDidDisappear中调用，这样当类需要被回收的时候就可以正常进入dealloc中了。

## 调用一个类的静态方法需不需要release？
静态方法，就是类方法，不需要，类方法对象放在autorelease中

## NSObject的load和initialize方法
**load和initialize的共同特点**
在不考虑开发者主动使用的情况下，系统最多会调用一次
如果父类和子类都被调用，父类的调用一定在子类之前
都是为了应用运行提前创建合适的运行环境
在使用时都不要过重地依赖于这两个方法，除非真正必要

**load和initialize的区别**
**load方法**

调用时机比较早，运行环境有不确定因素。具体说来，在iOS上通常就是App启动时进行加载，但当load调用的时候，并不能保证所有类都加载完成且可用，必要时还要自己负责做auto release处理。对于有依赖关系的两个库中，被依赖的类的load会优先调用。但在一个库之内，调用顺序是不确定的。

对于一个类而言，没有load方法实现就不会调用，不会考虑对NSObject的继承。

一个类的load方法不用写明[super load]，父类就会收到调用，并且在子类之前。

Category的load也会收到调用，但顺序上在主类的load调用之后。

不会直接触发initialize的调用。

**initialize方法相关要点**

initialize的自然调用是在第一次主动使用当前类的时候。

在initialize方法收到调用时，运行环境基本健全。

initialize的运行过程中是能保证线程安全的。

和load不同，即使子类不实现initialize方法，会把父类的实现继承过来调用一遍。注意的是在此之前，父类的方法已经被执行过一次了，同样不需要super调用。

由于initialize的这些特点，使得其应用比load要略微广泛一些。可用来做一些初始化工作，或者单例模式的一种实现方案。

##  能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？
不能向编译后得到的类中增加实例变量；
能向运行时创建的类中添加实例变量；

因为编译后的类已经注册在 runtime 中，类结构体中的 objc_ivar_list 实例变量的链表 和 instance_size 实例变量的内存大小已经确定，同时runtime 会调用 class_setIvarLayout 或 class_setWeakIvarLayout 来处理 strong weak 引用。所以不能向存在的类中添加实例变量；

运行时创建的类是可以添加实例变量，调用 class_addIvar 函数。但是得在调用 objc_allocateClassPair 之后，objc_registerClassPair 之前，原因同上。

## nil/Nil/NULL/NSNull
1. NULL
声明位置在stddef.h文件
对于普通的iOS开发者来说，通常NULL的定义就是：`#  define NULL ((void*)0)`
因此，NULL本质上是：(void*)0
NULL表示C指针为空`charchar *string = NULL;`
2. nil  
声明在objc.h文件
对于普通iOS开发者来说，nil的定义形式为：`#   define nil __DARWIN_NULL`
就是说nil最终是DARWIN_NULL的宏定义，DARWIN_NULL是定义在_types.h中的宏。`#define __DARWIN_NULL ((void *)0)`
也就是说，nil本质上是：(void *)0
用于表示指向Objective-C中对象的指针为空
```objc
NSString *string = nil;  
id anyObject = nil;
```
3. Nil
声明位置在objc.h文件
和上面讲到的nil一样，Nil本质上也是：(void *)0
用于表示Objective-C类（Class）类型的变量值为空
```objc
Class anyClass = Nil;
```
4. NSNull
声明位置在NSNull.h文件
定义
```objc
@interface NSNull : NSObject <NSCopying, NSSecureCoding>  
+ (NSNull *)null;  
@end
```
从定义中可以看出，NSNull是一个Objective-C类，只不过这个类相当特殊，因为它表示的是空值，即什么都不存。它也只有一个单例方法+[NSUll null]。该类通常用于在集合对象中保存一个空的占位对象。

我们通常初始化NSArray对象的形式如下：
```objc
NSArray *arr = [NSArray arrayWithObjects:@"wang",@"zz",nil];
```
当NSArray里遇到nil时，就说明这个数组对象的元素截止了，即NSArray只关注nil之前的对象，nil之后的对象会被抛弃。比如下面的写法：
```objc
NSArray *arr = [NSArray arrayWithObjects:@"wang",@"zz",nil,@"foogry"];
```
这是NSArray中只会保存wang和zz两个字符串，foogry字符串会被抛弃。
这种情况，就可以使用NSNull实现：
```objc
NSArray *arr = [NSArray arrayWithObjects:@"wang",@"zz",[NSNull null],@"foogry"];
```
从前面的介绍可以看出，不管是NULL、nil还是Nil，它们本质上都是一样的，都是(void *)0，只是写法不同。这样做的意义是为了区分不同的数据类型，比如你一看到用到了NULL就知道这是个C指针，看到nil就知道这是个Objective-C对象，看到Nil就知道这是个Class类型的数据。

注意：NULL是C指针指向的值为空；nil是OC对象指针自己本身为空，不是值为空

## 界面卡顿产生的原因和解决方案
> iOS界面处理是在主线程下进行的，系统图形服务通过 CADisplayLink 等机制通知 App，App 主线程开始在 CPU 中计算显示内容，比如视图的创建、布局计算、图片解码、文本绘制等。随后 CPU 会将计算好的内容提交到 GPU 去，由 GPU 进行变换、合成、渲染。随后 GPU 会把渲染结果提交到帧缓冲区去，等待下一次刷新信号到来时显示到屏幕上。显示器通常以固定频率进行刷新，如果在一个刷新时间内，CPU 或者 GPU 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。这就是界面卡顿的原因。CPU 和 GPU 不论哪个阻碍了显示流程，都会造成掉帧现象。
### CPU 资源消耗原因和解决方案
#### 对象创建
对象的创建会分配内存、调整属性、甚至还有读取文件等操作，比较消耗 CPU 资源。尽量用轻量的对象代替重量的对象，可以对性能有所优化。比如 CALayer 比 UIView 要轻量许多，那么不需要响应触摸事件的控件，用 CALayer 显示会更加合适。如果对象不涉及 UI 操作，则尽量放到后台线程去创建，但可惜的是包含有 CALayer 的控件，都只能在主线程创建和操作。通过 Storyboard 创建视图对象时，其资源消耗会比直接通过代码创建对象要大非常多，在性能敏感的界面里，Storyboard 并不是一个好的技术选择。

尽量推迟对象创建的时间，并把对象的创建分散到多个任务中去。尽管这实现起来比较麻烦，并且带来的优势并不多，但如果有能力做，还是要尽量尝试一下。如果对象可以复用，并且复用的代价比释放、创建新对象要小，那么这类对象应当尽量放到一个缓存池里复用。

#### 对象调整
对象的调整也经常是消耗 CPU 资源的地方。这里特别说一下 CALayer：CALayer 内部并没有属性，当调用属性方法时，它内部是通过运行时 resolveInstanceMethod 为对象临时添加一个方法，并把对应属性值保存到内部的一个 Dictionary 里，同时还会通知 delegate、创建动画等等，非常消耗资源。UIView 的关于显示相关的属性（比如 frame/bounds/transform）等实际上都是 CALayer 属性映射来的，所以对 UIView 的这些属性进行调整时，消耗的资源要远大于一般的属性。对此你在应用中，应该尽量减少不必要的属性修改。当视图层次调整时，UIView、CALayer 之间会出现很多方法调用与通知，所以在优化性能时，应该尽量避免调整视图层次、添加和移除视图。

#### 对象销毁
对象的销毁虽然消耗资源不多，但累积起来也是不容忽视的。通常当容器类持有大量对象时，其销毁时的资源消耗就非常明显。同样的，如果对象可以放到后台线程去释放，那就挪到后台线程去。这里有个小 Tip：把对象捕获到 block 中，然后扔到后台队列去随便发送个消息以避免编译器警告，就可以让对象在后台线程销毁了。
```objc
NSArray *tmp = self.array;
self.array = nil;
dispatch_async(queue, ^{
[tmp class];
});

```
#### 布局计算
视图布局的计算是 App 中最为常见的消耗 CPU 资源的地方。如果能在后台线程提前计算好视图布局、并且对视图布局进行缓存，那么这个地方基本就不会产生性能问题了。

不论通过何种技术对视图进行布局，其最终都会落到对 UIView.frame/bounds/center 等属性的调整上。上面也说过，对这些属性的调整非常消耗资源，所以尽量提前计算好布局，在需要时一次性调整好对应属性，而不要多次、频繁的计算和调整这些属性。

#### Autolayout
Autolayout 是苹果本身提倡的技术，在大部分情况下也能很好的提升开发效率，但是 Autolayout 对于复杂视图来说常常会产生严重的性能问题。随着视图数量的增长，Autolayout 带来的 CPU 消耗会呈指数级上升。如果你不想手动调整 frame 等属性，你可以用一些工具方法替代（比如常见的 left/right/top/bottom/width/height 快捷属性），或者使用 ComponentKit、AsyncDisplayKit 等框架。

#### 文本计算
如果一个界面中包含大量文本（比如微博微信朋友圈等），文本的宽高计算会占用很大一部分资源，并且不可避免。如果你对文本显示没有特殊要求，可以参考下 UILabel 内部的实现方式：用 [NSAttributedString boundingRectWithSize:options:context:] 来计算文本宽高，用 -[NSAttributedString drawWithRect:options:context:] 来绘制文本。尽管这两个方法性能不错，但仍旧需要放到后台线程进行以避免阻塞主线程。如果你用 CoreText 绘制文本，那就可以先生成 CoreText 排版对象，然后自己计算了，并且 CoreText 对象还能保留以供稍后绘制使用。

#### 文本渲染
屏幕上能看到的所有文本内容控件，包括 UIWebView，在底层都是通过 CoreText 排版、绘制为 Bitmap 显示的。常见的文本控件 （UILabel、UITextView 等），其排版和绘制都是在主线程进行的，当显示大量文本时，CPU 的压力会非常大。对此解决方案只有一个，那就是自定义文本控件，用 TextKit 或最底层的 CoreText 对文本异步绘制。尽管这实现起来非常麻烦，但其带来的优势也非常大，CoreText 对象创建好后，能直接获取文本的宽高等信息，避免了多次计算（调整 UILabel 大小时算一遍、UILabel 绘制时内部再算一遍）；CoreText 对象占用内存较少，可以缓存下来以备稍后多次渲染。

#### 图片的解码
当你用 UIImage 或 CGImageSource 的那几个方法创建图片时，图片数据并不会立刻解码。图片设置到 UIImageView 或者 CALayer.contents 中去，并且 CALayer 被提交到 GPU 前，CGImage 中的数据才会得到解码。这一步是发生在主线程的，并且不可避免。如果想要绕开这个机制，常见的做法是在后台线程先把图片绘制到 CGBitmapContext 中，然后从 Bitmap 直接创建图片。目前常见的网络图片库都自带这个功能。

#### 图像的绘制
图像的绘制通常是指用那些以 CG 开头的方法把图像绘制到画布中，然后从画布创建图片并显示这样一个过程。这个最常见的地方就是 [UIView drawRect:] 里面了。由于 CoreGraphic 方法通常都是线程安全的，所以图像的绘制可以很容易的放到后台线程进行。一个简单异步绘制的过程大致如下（实际情况会比这个复杂得多，但原理基本一致）：
```objc
- (void)display {
dispatch_async(backgroundQueue, ^{
    CGContextRef ctx = CGBitmapContextCreate(...);
    // draw in context...
    CGImageRef img = CGBitmapContextCreateImage(ctx);
    CFRelease(ctx);
    dispatch_async(mainQueue, ^{
        layer.contents = img;
    });
});
}
```
### GPU资源消耗原因和解决方案
相对于 CPU 来说，GPU 能干的事情比较单一：接收提交的纹理（Texture）和顶点描述（三角形），应用变换（transform）、混合并渲染，然后输出到屏幕上。通常你所能看到的内容，主要也就是纹理（图片）和形状（三角模拟的矢量图形）两类。

#### 纹理的渲染
所有的 Bitmap，包括图片、文本、栅格化的内容，最终都要由内存提交到显存，绑定为 GPU Texture。不论是提交到显存的过程，还是 GPU 调整和渲染 Texture 的过程，都要消耗不少 GPU 资源。当在较短时间显示大量图片时（比如 TableView 存在非常多的图片并且快速滑动时），CPU 占用率很低，GPU 占用非常高，界面仍然会掉帧。避免这种情况的方法只能是尽量减少在短时间内大量图片的显示，尽可能将多张图片合成为一张进行显示。

当图片过大，超过 GPU 的最大纹理尺寸时，图片需要先由 CPU 进行预处理，这对 CPU 和 GPU 都会带来额外的资源消耗。目前来说，iPhone 4S 以上机型，纹理尺寸上限都是 4096x4096，所以，尽量不要让图片和视图的大小超过这个值。

#### 视图的混合 (Composing)
当多个视图（或者说 CALayer）重叠在一起显示时，GPU 会首先把他们混合到一起。如果视图结构过于复杂，混合的过程也会消耗很多 GPU 资源。为了减轻这种情况的 GPU 消耗，应用应当尽量减少视图数量和层次，并在不透明的视图里标明 opaque 属性以避免无用的 Alpha 通道合成。当然，这也可以用上面的方法，把多个视图预先渲染为一张图片来显示。

#### 图形的生成
CALayer 的 border、圆角、阴影、遮罩（mask），CASharpLayer 的矢量图形显示，通常会触发离屏渲染（offscreen rendering），而离屏渲染通常发生在 GPU 中。当一个列表视图中出现大量圆角的 CALayer，并且快速滑动时，可以观察到 GPU 资源已经占满，而 CPU 资源消耗很少。这时界面仍然能正常滑动，但平均帧数会降到很低。为了避免这种情况，可以尝试开启 CALayer.shouldRasterize 属性，但这会把原本离屏渲染的操作转嫁到 CPU 上去。对于只需要圆角的某些场合，也可以用一张已经绘制好的圆角图片覆盖到原本视图上面来模拟相同的视觉效果。最彻底的解决办法，就是把需要显示的图形在后台线程绘制为图片，避免使用圆角、阴影、遮罩等属性。

## 如何追踪app崩溃率，如何解决线上闪退
当iOS设备上的App应用闪退时，操作系统会生成一个crash日志，保存在设备上。crash日志上有很多有用的信息，比如每个正在执行线程的完整堆栈跟踪信息和内存映像，这样就能够通过解析这些信息进而定位crash发生时的代码逻辑，从而找到App闪退的原因。通常来说，crash产生来源于两种问题：违反iOS系统规则导致的crash和App代码逻辑BUG导致的crash，下面分别对他们进行分析。

### 违反iOS系统规则产生crash的三种类型

(1) 内存报警闪退
当iOS检测到内存过低时，它的VM系统会发出低内存警告通知，尝试回收一些内存；如果情况没有得到足够的改善，iOS会终止后台应用以回收更多内存；最后，如果内存还是不足，那么正在运行的应用可能会被终止掉。在Debug模式下，可以主动将客户端执行的动作逻辑写入一个log文件中，这样程序童鞋可以将内存预警的逻辑写入该log文件，当发生如下截图中的内存报警时，就是提醒当前客户端性能内存吃紧，可以通过Instruments工具中的Allocations 和 Leaks模块库来发现内存分配问题和内存泄漏问题。

(2) 响应超时
当应用程序对一些特定的事件（比如启动、挂起、恢复、结束）响应不及时，苹果的Watchdog机制会把应用程序干掉，并生成一份相应的crash日志。这些事件与下列UIApplicationDelegate方法相对应，当遇到Watchdog日志时，可以检查上图中的几个方法是否有比较重的阻塞UI的动作。　
```objc
application:didFinishLaunchingWithOptions:　
applicationWillResignActive:
applicationDidEnterBackground:　
applicationWillEnterForeground:
applicationDidBecomeActive:
applicationWillTerminate:
```
(3) 用户强制退出
一看到“用户强制退出”，首先可能想到的双击Home键，然后关闭应用程序。不过这种场景一般是不会产生crash日志的，因为双击Home键后，所有的应用程序都处于后台状态，而iOS随时都有可能关闭后台进程，当应用阻塞界面并停止响应时这种场景才会产生crash日志。这里指的“用户强制退出”场景，是稍微比较复杂点的操作：先按住电源键，直到出现“滑动关机”的界面时，再按住Home键，这时候当前应用程序会被终止掉，并且产生一份相应事件的crash日志。

### 应用逻辑的Bug
大多数闪退崩溃日志的产生都是因为应用中的Bug，这种Bug的错误种类有很多，比如　　
```objc
SEGV：（Segmentation Violation，段违例），无效内存地址，比如空指针，未初始化指针，栈溢出等；
  SIGABRT：收到Abort信号，可能自身调用abort()或者收到外部发送过来的信号；
  SIGBUS：总线错误。与SIGSEGV不同的是，SIGSEGV访问的是无效地址（比如虚存映射不到物理内存），而SIGBUS访问的是有效地址，但总线访问异常（比如地址对齐问题）；
  SIGILL：尝试执行非法的指令，可能不被识别或者没有权限；
  SIGFPE：Floating Point Error，数学计算相关问题（可能不限于浮点计算），比如除零操作；
  SIGPIPE：管道另一端没有进程接手数据；
```
常见的崩溃原因基本都是代码逻辑问题或资源问题，比如数组越界，访问野指针或者资源不存在，或资源大小写错误等。

### crash的收集

如果是在windows上你可以通过itools或pp助手等辅助工具查看系统产生的历史crash日志，然后再根据app来查看。如果是在Mac 系统上，只需要打开xcode->windows->devices，选择device logs进行查看，如下图，这些crash文件都可以导出来，然后再单独对这个crash文件做处理分析。

市场上已有的商业软件提供crash收集服务，这些软件基本都提供了日志存储，日志符号化解析和服务端可视化管理等服务：

Crashlytics (www.crashlytics.com)
Crittercism (www.crittercism.com)
Bugsense (www.bugsense.com)　　
HockeyApp (www.hockeyapp.net)　　
Flurry(www.flurry.com)

开源的软件也可以拿来收集crash日志，比如Razor,QuincyKit（git链接）等，这些软件收集crash的原理其实大同小异，都是根据系统产生的crash日志进行了一次提取或封装，然后将封装后的crash文件上传到对应的服务端进行解析处理。很多商业软件都采用了Plcrashreporter这个开源工具来上传和解析crash，比如HockeyApp,Flurry和crittercism等。

由于自己的crash信息太长，找了一张示例：　　
1)crash标识是应用进程产生crash时的一些标识信息，它描述了该crash的唯一标识（E838FEFB-ECF6-498C-8B35-D40F0F9FEAE4），所发生的硬件设备类型（iphone3,1代表iphone4），以及App进程相关的信息等；
2）基本信息描述的是crash发生的时间和系统版本；　　
3）异常类型描述的是crash发生时抛出的异常类型和错误码；　　
4）线程回溯描述了crash发生时所有线程的回溯信息，每个线程在每一帧对应的函数调用信息（这里由于空间限制没有全部列出）；　　
5）二进制映像是指crash发生时已加载的二进制文件。以上就是一份crash日志包含的所有信息，接下来就需要根据这些信息去解析定位导致crash发生的代码逻辑， 这就需要用到符号化解析的过程（洋名叫：symbolication)。

## 什么是事件响应链，点击屏幕时是如何互动的，事件的传递。
对于IOS设备用户来说，他们操作设备的方式主要有三种：触摸屏幕、晃动设备、通过遥控设施控制设备。对应的事件类型有以下三种：

1. 触屏事件（Touch Event）
2. 运动事件（Motion Event）
3.远端控制事件（Remote-Control Event）

**响应者链（Responder Chain）**
响应者对象（Responder Object），指的是有响应和处理事件能力的对象。响应者链就是由一系列的响应者对象构成的一个层次结构。

UIResponder是所有响应对象的基类，在UIResponder类中定义了处理上述各种事件的接口。我们熟悉的UIApplication、 UIViewController、UIWindow和所有继承自UIView的UIKit类都直接或间接的继承自UIResponder，所以它们的实例都是可以构成响应者链的响应者对象。

响应者链有以下特点：
1、响应者链通常是由视图（UIView）构成的；
2、一个视图的下一个响应者是它视图控制器（UIViewController）（如果有的话），然后再转给它的父视图（Super View）；
3、视图控制器（如果有的话）的下一个响应者为其管理的视图的父视图；
4、单例的窗口（UIWindow）的内容视图将指向窗口本身作为它的下一个响应者
需要指出的是，Cocoa Touch应用不像Cocoa应用，它只有一个UIWindow对象，因此整个响应者链要简单一点；
5、单例的应用（UIApplication）是一个响应者链的终点，它的下一个响应者指向nil，以结束整个循环。

**点击屏幕时是如何互动的**
iOS系统检测到手指触摸(Touch)操作时会将其打包成一个UIEvent对象，并放入当前活动Application的事件队列，单例的UIApplication会从事件队列中取出触摸事件并传递给单例的UIWindow来处理，UIWindow对象首先会使用hitTest:withEvent:方法寻找此次Touch操作初始点所在的视图(View)，即需要将触摸事件传递给其处理的视图，这个过程称之为hit-test view。

UIWindow实例对象会首先在它的内容视图上调用hitTest:withEvent:，此方法会在其视图层级结构中的每个视图上调用pointInside:withEvent:（该方法用来判断点击事件发生的位置是否处于当前视图范围内，以确定用户是不是点击了当前视图），如果pointInside:withEvent:返回YES，则继续逐级调用，直到找到touch操作发生的位置，这个视图也就是要找的hit-test view。

hitTest:withEvent:方法的处理流程如下:首先调用当前视图的pointInside:withEvent:方法判断触摸点是否在当前视图内；若返回NO,则hitTest:withEvent:返回nil;若返回YES,则向当前视图的所有子视图(subviews)发送hitTest:withEvent:消息，所有子视图的遍历顺序是从最顶层视图一直到到最底层视图，即从subviews数组的末尾向前遍历，直到有子视图返回非空对象或者全部子视图遍历完毕；若第一次有子视图返回非空对象，则hitTest:withEvent:方法返回此对象，处理结束；如所有子视图都返回非，则hitTest:withEvent:方法返回自身(self)。

事件的传递和响应分两个链：

传递链：由系统向离用户最近的view传递。UIKit –> active app’s event queue –> window –> root view –>……–>lowest view
响应链：由离用户最近的view向系统传递。initial view –> super view –> …..–> view controller –> window –> Application

## Run Loop是什么，使用的目的，何时使用和关注点
Run Loop是一让线程能随时处理事件但不退出的机制。RunLoop 实际上是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 "接受消息->等待->处理" 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。

OSX/iOS 系统中，提供了两个这样的对象：NSRunLoop 和 CFRunLoopRef。CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。

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
重点是UIApplicationMain() 函数，这个方法会为main thread 设置一个NSRunLoop 对象，这就解释了本文开始说的为什么我们的应用可以在无人操作的时候休息，需要让它干活的时候又能立马响应。

对其它线程来说，run loop默认是没有启动的，如果你需要更多的线程交互则可以手动配置和启动，如果线程只是去执行一个长时间的已确定的任务则不需要。在任何一个Cocoa程序的线程中，都可以通过：
```objc
NSRunLoop   *runloop = [NSRunLoop currentRunLoop];
```
来获取到当前线程的run loop。

一个run loop就是一个事件处理循环，用来不停的监听和处理输入事件并将其分配到对应的目标上进行处理。

NSRunLoop是一种更加高明的消息处理模式，他就高明在对消息处理过程进行了更好的抽象和封装，这样才能是的你不用处理一些很琐碎很低层次的具体消息的处理，在NSRunLoop中每一个消息就被打包在input source或者是timer source中了。使用run loop可以使你的线程在有工作的时候工作，没有工作的时候休眠，这可以大大节省系统资源。
**什么时候使用runloop**
仅当在为你的程序创建辅助线程的时候，你才需要显式运行一个run loop。Run loop是程序主线程基础设施的关键部分。所以，Cocoa和Carbon程序提供了代码运行主程序的循环并自动启动run loop。IOS程序中UIApplication的run方法（或Mac OS X中的NSApplication）作为程序启动步骤的一部分，它在程序正常启动的时候就会启动程序的主循环。类似的，RunApplicationEventLoop函数为Carbon程序启动主循环。如果你使用xcode提供的模板创建你的程序，那你永远不需要自己去显式的调用这些例程。

对于辅助线程，你需要判断一个run loop是否是必须的。如果是必须的，那么你要自己配置并启动它。你不需要在任何情况下都去启动一个线程的run loop。比如，你使用线程来处理一个预先定义的长时间运行的任务时，你应该避免启动run loop。Run loop在你要和线程有更多的交互时才需要，比如以下情况：
- 使用端口或自定义输入源来和其他线程通信
- 使用线程的定时器
- Cocoa中使用任何performSelector…的方法
- 使线程周期性工作

**关注点**
1. Cocoa中的NSRunLoop类并不是线程安全的
我们不能再一个线程中去操作另外一个线程的run loop对象，那很可能会造成意想不到的后果。不过幸运的是CoreFundation中的不透明类CFRunLoopRef是线程安全的，而且两种类型的run loop完全可以混合使用。Cocoa中的NSRunLoop类可以通过实例方法：
```objc
- (CFRunLoopRef)getCFRunLoop;
```
获取对应的CFRunLoopRef类，来达到线程安全的目的。
2. Run loop的管理并不完全是自动的。
我们仍必须设计线程代码以在适当的时候启动run loop并正确响应输入事件，当然前提是线程中需要用到run loop。而且，我们还需要使用while/for语句来驱动run loop能够循环运行，下面的代码就成功驱动了一个run loop：
```objc
BOOL isRunning = NO;
do {
     isRunning = [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDatedistantFuture]];
} while (isRunning);
```
3. Run loop同时也负责autorelease pool的创建和释放
在使用手动的内存管理方式的项目中，会经常用到很多自动释放的对象，如果这些对象不能够被即时释放掉，会造成内存占用量急剧增大。Run loop就为我们做了这样的工作，每当一个运行循环结束的时候，它都会释放一次autorelease pool，同时pool中的所有自动释放类型变量都会被释放掉。
##  ARC和MRC
Objective-c中提供了两种内存管理机制MRC（MannulReference Counting）和ARC(Automatic Reference Counting)，分别提供对内存的手动和自动管理，来满足不同的需求。Xcode 4.1及其以前版本没有ARC。

在MRC的内存管理模式下，与对变量的管理相关的方法有：retain,release和autorelease。retain和release方法操作的是引用记数，当引用记数为零时，便自动释放内存。并且可以用NSAutoreleasePool对象，对加入自动释放池（autorelease调用）的变量进行管理，当drain时回收内存。
  1. retain，该方法的作用是将内存数据的所有权附给另一指针变量，引用数加1，即retainCount+= 1;
  2. release，该方法是释放指针变量对内存数据的所有权，引用数减1，即retainCount-= 1;
  3. autorelease，该方法是将该对象内存的管理放到autoreleasepool中。

在ARC中与内存管理有关的标识符，可以分为变量标识符和属性标识符，对于变量默认为__strong，而对于属性默认为unsafe_unretained。也存在autoreleasepool。

其中assign/retain/copy与MRC下property的标识符意义相同，strong类似与retain,assign类似于unsafe_unretained，strong/weak/unsafe_unretained与ARC下变量标识符意义相同，只是一个用于属性的标识，一个用于变量的标识(带两个下划短线__)。所列出的其他的标识符与MRC下意义相同。

## 大量数据表的优化方案
1. 对查询进行优化，要尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。

2. 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描，如：

 `select id from t where num is null`
最好不要给数据库留NULL，尽可能的使用 NOT NULL填充数据库.

备注、描述、评论之类的可以设置为 NULL，其他的，最好不要使用NULL。

不要以为 NULL 不需要空间，比如：char(100) 型，在字段建立时，空间就固定了， 不管是否插入值（NULL也包含在内），都是占用 100个字符的空间的，如果是varchar这样的变长字段， null 不占用空间。

可以在num上设置默认值0，确保表中num列没有null值，然后这样查询：

select id from t where num=0
3. 应尽量避免在 where 子句中使用 != 或 <> 操作符，否则将引擎放弃使用索引而进行全表扫描。

4. 应尽量避免在 where 子句中使用 or 来连接条件，如果一个字段有索引，一个字段没有索引，将导致引擎放弃使用索引而进行全表扫描，如：

 select id from t where num=10 or Name='admin'
可以这样查询：

 select id from t where num=10 union all select id from t where Name='admin'
5. in 和 not in 也要慎用，否则会导致全表扫描，如：

select id from t where num in (1,2,3)
对于连续的数值，能用 between 就不要用 in 了：

 select id from t where num between 1 and 3
很多时候用 exists 代替 in 是一个好的选择：

 select num from a where num in (select num from b)
用下面的语句替换：

 select num from a where exists (select 1 from b where num=a.num)
6. 下面的查询也将导致全表扫描：

select id from t where name like ‘%abc%’
若要提高效率，可以考虑全文检索。

7. 如果在 where 子句中使用参数，也会导致全表扫描。因为SQL只有在运行时才会解析局部变量，但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时进行选择。然 而，如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。如下面语句将进行全表扫描：

select id from t where num=@num
可以改为强制查询使用索引：

select id from t with (index(索引名)) where num=@num
应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。如：

select id from t where num/2=100
应改为:

select id from t where num=100*2
8. 应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：

 select id from t where substring(name,1,3)=’abc’ -–name以abc开头的id
 select id from t where datediff(day,createdate,’2015-11-30′)=0 -–‘2015-11-30’ --生成的id
应改为:

select id from t where name like'abc%'
select id from t where createdate>='2005-11-30' and createdate<'2005-12-1'
9. 不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。

10. 在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使用，并且应尽可能的让字段顺序与索引顺序相一致。

11. 不要写一些没有意义的查询，如需要生成一个空表结构：

select col1,col2 into #t from t where1=0
这类代码不会返回任何结果集，但是会消耗系统资源的，应改成这样：

create table #t(…)
12. Update 语句，如果只更改1、2个字段，不要Update全部字段，否则频繁调用会引起明显的性能消耗，同时带来大量日志。

13. 对于多张大数据量（这里几百条就算大了）的表JOIN，要先分页再JOIN，否则逻辑读会很高，性能很差。

14. select count(*) from table；这样不带任何条件的count会引起全表扫描，并且没有任何业务意义，是一定要杜绝的。

15. 索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有 必要。

16. 应尽可能的避免更新 clustered 索引数据列，因为 clustered 索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新 clustered 索引数据列，那么需要考虑是否应将该索引建为 clustered 索引。

17. 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连 接时会逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。

18. 尽可能的使用 varchar/nvarchar 代替 char/nchar ，因为首先变长字段存储空间小，可以节省存储空间，其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。

19. 任何地方都不要使用

  select * from t
用具体的字段列表代替“*”，不要返回用不到的任何字段。

20. 尽量使用表变量来代替临时表。如果表变量包含大量数据，请注意索引非常有限（只有主键索引）。

21. 避免频繁创建和删除临时表，以减少系统表资源的消耗。临时表并不是不可使用，适当地使用它们可以使某些例程更有效，例如，当需要重复引用大型表或常用表中的某个数据集时。但是，对于一次性事件， 最好使用导出表。

22. 在新建临时表时，如果一次性插入数据量很大，那么可以使用 select into 代替 create table，避免造成大量 log ，以提高速度；如果数据量不大，为了缓和系统表的资源，应先create table，然后insert。

23. 如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先 truncate table ，然后 drop table ，这样可以避免系统表的较长时间锁定。

24. 尽量避免使用游标，因为游标的效率较差，如果游标操作的数据超过1万行，那么就应该考虑改写。

25. 使用基于游标的方法或临时表方法之前，应先寻找基于集的解决方案来解决问题，基于集的方法通常更有效。

26. 与临时表一样，游标并不是不可使用。对小型数据集使用 FAST_FORWARD 游标通常要优于其他逐行处理方法，尤其是在必须引用几个表才能获得所需的数据时。在结果集中包括“合计”的例程通常要比使用游标执行的速度快。如果开发时 间允许，基于游标的方法和基于集的方法都可以尝试一下，看哪一种方法的效果更好。

27. 在所有的存储过程和触发器的开始处设置 SET NOCOUNT ON ，在结束时设置 SET NOCOUNT OFF 。无需在执行存储过程和触发器的每个语句后向客户端发送 DONE_IN_PROC 消息。

28. 尽量避免大事务操作，提高系统并发能力。

29. 尽量避免向客户端返回大数据量，若数据量过大，应该考虑相应需求是否合理。

实际案例分析：拆分大的 DELETE 或INSERT 语句，批量提交SQL语句

如果你需要在一个在线的网站上去执行一个大的 DELETE 或 INSERT 查询，你需要非常小心，要避免你的操作让你的整个网站停止相应。因为这两个操作是会锁表的，表一锁住了，别的操作都进不来了。

Apache 会有很多的子进程或线程。所以，其工作起来相当有效率，而我们的服务器也不希望有太多的子进程，线程和数据库链接，这是极大的占服务器资源的事情，尤其是内存。

如果你把你的表锁上一段时间，比如30秒钟，那么对于一个有很高访问量的站点来说，这30秒所积累的访问进程/线程，数据库链接，打开的文件数，可能不仅仅会让你的WEB服务崩溃，还可能会让你的整台服务器马上挂了。

所以，如果你有一个大的处理，你一定把其拆分，使用 LIMIT oracle(rownum),sqlserver(top)条件是一个好的方法。下面是一个mysql示例：

## Restful架构
REST是一种架构风格，其核心是面向资源，REST专门针对网络应用设计和开发方式，以降低开发的复杂性，提高系统的可伸缩性。REST提出设计概念和准则为：
```objc
1. 网络上的所有事物都可以被抽象为资源(resource)
2. 每一个资源都有唯一的资源标识(resource identifier)，对资源的操作不会改变这些标识
3. 所有的操作都是无状态的
```
REST简化开发，其架构遵循CRUD原则，该原则告诉我们对于资源(包括网络资源)只需要四种行为：创建，获取，更新和删除就可以完成相关的操作和处理。您可以通过统一资源标识符（Universal Resource Identifier，URI）来识别和定位资源，并且针对这些资源而执行的操作是通过 HTTP 规范定义的。其核心操作只有GET,PUT,POST,DELETE。

由于REST强制所有的操作都必须是stateless的，这就没有上下文的约束，如果做分布式，集群都不需要考虑上下文和会话保持的问题。极大的提高系统的可伸缩性。

RESTful架构：
　　（1）每一个URI代表一种资源；
　　（2）客户端和服务器之间，传递这种资源的某种表现层；
　　（3）客户端通过四个HTTP动词，对服务器端资源进行操作，实现"表现层状态转化"。

## 写一个单例模式
```objc
+ (AccountManager *)sharedManager
{
    static AccountManager *sharedAccountManagerInstance = nil;
    static dispatch_once_t predicate;
    dispatch_once(&predicate, ^{
            sharedAccountManagerInstance = [[self alloc] init];
    });
return sharedAccountManagerInstance;
}
```
## iOS Life Cycle
**应用程序的状态**
**Not running未运行**：程序没启动。
**Inactive未激活**：程序在前台运行，不过没有接收到事件。在没有事件处理情况下程序通常停留在这个状态。
**Active激活**：程序在前台运行而且接收到了事件。这也是前台的一个正常的模式。
**Backgroud后台**：程序在后台而且能执行代码，大多数程序进入这个状态后会在在这个状态上停留一会。时间到之后会进入挂起状态(Suspended)。有的程序经过特殊的请求后可以长期处于Backgroud状态。
**Suspended挂起**：程序在后台不能执行代码。系统会自动把程序变成这个状态而且不会发出通知。当挂起时，程序还是停留在内存中的，当系统内存低时，系统就把挂起的程序清除掉，为前台程序提供更多的内存。

iOS的入口在main.m文件：
```objc
int main(int argc, char *argv[])
{
@autoreleasepool {
    return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
}
}
```
main函数的两个参数，iOS中没有用到，包括这两个参数是为了与标准ANSI C保持一致。 UIApplicationMain函数，前两个和main函数一样，重点是后两个。

后两个参数分别表示程序的主要类(principal class)和代理类(delegate class)。如果主要类(principal class)为nil，将从Info.plist中获取，如果Info.plist中不存在对应的key，则默认为UIApplication；如果代理类(delegate class)将在新建工程时创建。
根据UIApplicationMain函数，程序将进入AppDelegate.m，这个文件是xcode新建工程时自动生成的。下面看一下AppDelegate.m文件，这个关乎着应用程序的生命周期。

1. application didFinishLaunchingWithOptions：当应用程序启动时执行，应用程序启动入口，只在应用程序启动时执行一次。若用户直接启动，lauchOptions内无数据,若通过其他方式启动应用，lauchOptions包含对应方式的内容。
2. applicationWillResignActive：在应用程序将要由活动状态切换到非活动状态时候，要执行的委托调用，如 按下 home 按钮，返回主屏幕，或全屏之间切换应用程序等。
3. applicationDidEnterBackground：在应用程序已进入后台程序时，要执行的委托调用。
4. applicationWillEnterForeground：在应用程序将要进入前台时(被激活)，要执行的委托调用，刚好与applicationWillResignActive 方法相对应。
5. applicationDidBecomeActive：在应用程序已被激活后，要执行的委托调用，刚好与applicationDidEnterBackground 方法相对应。
6. applicationWillTerminate：在应用程序要完全推出的时候，要执行的委托调用，这个需要要设置UIApplicationExitsOnSuspend的键值。

**初次启动**：
iOS_didFinishLaunchingWithOptions
iOS_applicationDidBecomeActive
**按下home键**：
iOS_applicationWillResignActive
iOS_applicationDidEnterBackground
**点击程序图标进入**：
iOS_applicationWillEnterForeground
iOS_applicationDidBecomeActive

当应用程序进入后台时,应该保存用户数据或状态信息，所有没写到磁盘的文件或信息，在进入后台时，最后都写到磁盘去，因为程序可能在后台被杀死。释放尽可能释放的内存。
```objc
- (void)applicationDidEnterBackground:(UIApplication *)application
```
方法有大概5秒的时间让你完成这些任务。如果超过时间还有未完成的任务，你的程序就会被终止而且从内存中清除。

如果还需要长时间的运行任务，可以在该方法中调用
```objc
[application beginBackgroundTaskWithExpirationHandler:^{

    NSLog(@"begin Background Task With Expiration Handler");

}];

```
**程序终止**
程序只要符合以下情况之一，只要进入后台或挂起状态就会终止：

1. OS4.0以前的系统
2. app是基于iOS4.0之前系统开发的。
3. 设备不支持多任务
4. 在Info.plist文件中，程序包含了 UIApplicationExitsOnSuspend 键。

系统常常是为其他app启动时由于内存不足而回收内存最后需要终止应用程序，但有时也会是由于app很长时间才响应而终止。如果app当时运行在后台并且没有暂停，系统会在应用程序终止之前调用app的代理的方法 - (void)applicationWillTerminate:(UIApplication *)application，这样可以让你可以做一些清理工作。你可以保存一些数据或app的状态。这个方法也有5秒钟的限制。超时后方法会返回程序从内存中清除。

注意：用户可以手工关闭应用程序。

## 远程推送
当服务端远程向APNS推送至一台离线的设备时，苹果服务器Qos组件会自动保留一份最新的通知，等设备上线后，Qos将把推送发送到目标设备上

远程推送的基本过程
1. 客户端的app需要将用户的UDID和app的bundleID发送给apns服务器,进行注册,apns将加密后的device Token返回给app
2. app获得device Token后,上传到公司服务器
3. 当需要推送通知时,公司服务器会将推送内容和device Token一起发给apns服务器
4. apns再将推送内容送到客户端上

创建证书的流程：
1. 打开钥匙串，生成CertificateSigningRequest.certSigningRequest文件
2. 将CertificateSigningRequest.certSigningRequest上传进developer，导出.cer文件
3. 利用CSR导出P12文件
4. 需要准备下设备token值（无空格）
5. 使用OpenSSL合成服务器所使用的推送证书

本地app代码参考
1.注册远程通知
```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions//中注册远程通知
{
[[UIApplication sharedApplication] registerForRemoteNotificationTypes:(UIRemoteNotificationTypeAlert | UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeSound)];
}
```
2. 实现几个代理方法：
```objc
//获取deviceToken令牌  
-(void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken  
{  
//获取设备的deviceToken唯一编号  
NSLog(@"deviceToken=%@",deviceToken);  
NSString *realDeviceToken=[NSString stringWithFormat:@"%@",deviceToken];  
//去除<>  
realDeviceToken = [realDeviceToken stringByReplacingOccurrencesOfString:@"<" withString:@""];  
realDeviceToken = [realDeviceToken stringByReplacingOccurrencesOfString:@">" withString:@""];  
NSLog(@"realDeviceToken=%@",realDeviceToken);  
[[NSUserDefaults standardUserDefaults] setValue:realDeviceToken forKey:@"DeviceToken"];  //要发送给服务器
}  

 //获取令牌出错  
-(void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error  
{  
//注册远程通知设备出错  
NSLog(@"RegisterForRemoteNotification error=%@",error);  
}  
//在应用在前台时受到消息调用  
-(void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo  
{  
 //打印推送的消息  
NSLog(@"%@",[[userInfo objectForKey:@"aps"] objectForKey:@"alert"]):  
}
```
一般我们是使用开发版本的Provisioning做推送测试,如果没有问题,再使用发布版本证书的时候一般也应该是没有问题的。为了以防万一,我们可以在越狱的手机上安装我们的使用发布版证书的ipa文件(最好使用debug版本,并打印出获取到的deviceToken),安装成功后在;XCode->Window->Organizer-找到对应的设备查看console找到打印的deviceToken。

在后台的推送程序中使用发布版制作的证书并使用该deviceToken做推送服务.
使用开发和发布证书获取到的deviceToken是不一样的。

## 支付宝SDK使用
使用支付宝进行一个完整的支付功能，大致有以下步骤：向支付宝申请, 与支付宝签约，获得商户ID（partner）和账号ID（seller）和私钥(privateKey)。下载支付宝SDK，生成订单信息,签名加密调用支付宝客户端，由支付宝客户端跟支付宝安全服务器打交道。支付完毕后,支付宝客户端会自动跳回到原来的应用程序，在原来的应用程序中显示支付结果给用户看。
**集成之后可能遇到的问题**
1. 集成SDK编译时找不到 openssl/asn1.h 文件
解决方案：Build Settings --> Search Paths --> Header Search paths : $(SRCROOT)/支付宝集成/Classes/Alipay
2. 链接时：找不到 SystemConfiguration.framework 这个库
解决方案：打开支付宝客户端进行支付(用户没有安装支付宝客户端,直接在应用程序中添加一个WebView,通过网页让用户进行支付)
// 注意:如果是通过网页支付完成,那么会回调该block:callback
```objc
 [[AlipaySDK defaultService] payOrder:orderString fromScheme:@"jingdong" callback:^(NSDictionary *resultDic) { }];
```
在AppDelegate.m
```objc
// 当通过别的应用程序,将该应用程序打开时,会调用该方法
- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<NSString *,id> *)options{ // 当用户通过支付宝客户端进行支付时,会回调该block:standbyCallback
[[AlipaySDK defaultService] processOrderWithPaymentResult:url standbyCallback:^(NSDictionary *resultDic) { NSLog(@"result = %@",resultDic); }]; return YES;}
```
## iOS的锁屏和解锁

**idleTimer**
idleTimer 是iOS内置的时间监测机制，当在一段时间内未操作即进入锁屏状态。但有些应用程序是不需要锁住屏幕的，比如游戏，视频这类应用。 可以通过设置UIApplication的idleTimerDisabled属性来指定iOS是否锁屏。
```objc
// 禁用休闲时钟
[[UIApplication sharedApplication] setIdleTimerDisabled: YES];
```
也可以使用这种语法
```objc
[UIApplication sharedApplication].idleTimerDisabled = YES;
```
但是，这个命令只能禁用自动锁屏，如果点击了锁屏按钮，仍然会进入锁屏的。有一点例外的是，AVPlayer不用设置idleTimerDisabled=YES，也能屏幕常亮，播放完成后过一分钟就自动关闭屏幕。有兴趣的可以自己尝试一下。

**锁屏和解锁通知**
iPhone的锁屏监测分为两种方式监听：一种是程序在前台，另一种程序在后台。 程序在前台，这种比较简单。直接使用Darwin层的通知就可以了：
> Darwin是由苹果电脑于2000年所释出的一个开放原始码操作系统。Darwin 是MacOSX 操作环境的操作系统成份。苹果电脑于2000年把Darwin 释出给开放原始码社群。现在的Darwin皆可以在苹果电脑的PowerPC 架构和X86 架构下执行，而后者的架构只有有限的驱动程序支援。

```objc
#import <notify.h>
#define NotificationLock CFSTR("com.apple.springboard.lockcomplete")
#define NotificationChange CFSTR("com.apple.springboard.lockstate")
#define NotificationPwdUI CFSTR("com.apple.springboard.hasBlankedScreen")

static void screenLockStateChanged(CFNotificationCenterRef center,void* observer,CFStringRef name,const void* object,CFDictionaryRef userInfo)
{
NSString* lockstate = (__bridge NSString*)name;
if ([lockstate isEqualToString:(__bridge  NSString*)NotificationLock]) {
    NSLog(@"locked.");
} else {
    NSLog(@"lock state changed.");
}
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), NULL, screenLockStateChanged, NotificationLock, NULL, CFNotificationSuspensionBehaviorDeliverImmediately);
CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), NULL, screenLockStateChanged, NotificationChange, NULL, CFNotificationSuspensionBehaviorDeliverImmediately);
return YES;
}
```
notify.h的具体内容可以移步开发文档。这种方法，程序在前台是可以拿到的，在后台情况下就无能为力了。

第二种是程序退后台后，这时再锁屏就收不到上面的那个通知了，需要另外一种方式, 以循环的方式一直来检测是否是锁屏状态，会消耗性能并可能被苹果挂起，需要合理设置循环时间。
```objc
static void setScreenStateCb()
{
uint64_t locked;

__block int token = 0;
notify_register_dispatch("com.apple.springboard.lockstate",&token,dispatch_get_main_queue(),^(int t){
});
notify_get_state(token, &locked);
NSLog(@"%d",(int)locked);
}

- (void)applicationDidEnterBackground:(UIApplication *)application
{
while (YES) {
    setScreenStateCb();
    sleep(5); // 循环5s
}
}

```

**UIApplication**
上面我们使用了UIApplication的IdleTimerDisabled方法，下面就大概了解下UIApplication吧。

UIApplication，每个程序只能有一个，系统使用的是单例模式，用[UIApplication sharedApplication]来得到一个实例。这个单例实例是在系统启动时由main函数里面的UIApplicationMain方法生成，实现的是UIApplicationDelegate的Protocol，也就是AppDelegate的一个实例。每次通过[UIApplication sharedApplication]调用的就是它。UIApplication保存一个UIWindow对象序列，用来快速恢复views。

UIApplication在程序里的作用很多，大致如下所示：
```objc
一、远程提醒，就是push notification注册；
二、可以连接到UIUndoManager；在Cocoa中使用NSUndoManager可以很方便的完成撤销操作。NSUndoManager会记录下修改、撤销操作的消息。这个机制使用两个NSInvocation对象栈。当进行操作时，控制器会添加一个该操作的逆操作的invocation到Undo栈中。当进行Undo操作时，Undo操作的逆操作会倍添加到Redo栈中，就这样利用Undo和Redo两个堆栈巧妙的实现撤销操作。需要注意的是，堆栈中存放的都是NSInvocation实例。
三、检查能否打开某个URL，并且打开URL；这个功能可以配合应用的自定义URL功能，来检测是否安装了某个应用。使用的是[[UIApplication sharedApplication] canOpenURL:url]方法。如果返回YES，可执行[[UIApplication sharedApplication] openURL:url];
四、注册Local Notification；
五、在后台运行以及从后台转为前台时的操作；
六、防止屏幕睡眠：即上面的[[UIApplication sharedApplication] setIdleTimerDisabled: YES];
七、手动调整status bar的位置和状态，如设置为竖屏、横屏等；
八、设置badge number，就是图标右上角的数字；
九、每当应用联网时，在状态栏上会显示联网小菊花。UIApplication可以设置是否出现。
UIApplication *app = [UIApplication sharedApplication];
app.networkActivityIndicatorVisible =!app.networkActivityIndicatorVisible;//转动
app.networkActivityIndicatorVisible = app.networkActivityIndicatorVisible;//不转动
```
UIUndoManager示例
```objc
- (void) one
{
position = position + 10;
[[undoManager prepareWithInvocationTarget:self] two];
[self showTheChangesToThePostion];
}

- (void) two
{
position = position - 10;
[[undoManager prepareWithInvocationTarget:self] one];
[self showTheChangesToThePostion];
}
```
prepareWithInvocationTarget:方法记录了target并返回UndoManager，然后UndoManager重载了forwardInvocation方法，也就将two方法的Invocation添加到undo栈中了。

## 什么是OC
OC语言在c语言的基础上，增加了一层最小的面向对象语法，完全兼容C语言，在OC代码中，可以混用c，甚至是c++代码。可以使用OC开发mac osx平台和iOS平台的应用程序。拓展名：c语言.c OC语言.m 兼容C++.mm。

为了与c语言的关键字区分开来,基本上所有的关键字都是以@开头。基本类型：5种，增加了布尔类型，BOOL类型与其他类型的用法一致，BOOL类型的本质是char类型的，定义如下：
```objc
 Typedef signed char BOOL
```
宏定义:
```objc
#define YES  (BOOL)1
#define NO   (BOOL)0
```
布尔类型的输出一般当做整数来用。
程序编译连接过程为：源文件（.m）---(编译)---->目标文件（.0）-----（链接）---->可执行文件（.out）。

每个对象内部都默认有一个isa指针指向这个对象所使用的类。isa是对象中的隐藏指针，指向创建这个对象的类。OC做为一门面向对象语言，具有面向对象的语言特性，如封装、继承、多态。也具有静态语言的特性(如C++)，又有动态语言的效率(动态绑定、动态加载等)。

Apple公司领导着Objective-C语言的发展与维护，包括Objective-C运行时，Cocoa/Cocoa-Touch框架以及Objective-C语言的编译器。

## 什么是面向对象
OC语言是面向对象的，c语言是面向过程的，面向对象和面向过程只是解决问题的两种思考方式，面向过程关注的是解决问题涉及的步骤，面向对象关注的是设计能够实现解决问题所需功能的类。

面向对象的编程方法具有四个基本特征：抽象，封装，继承和多态。

抽象是忽略一个主题中与当前目标无关的那些方面，以便更充分地注意与当前目标有关的方面。抽象包括两个方面，一是过程抽象，二是数据抽象。过程抽象是指任何一个明确定义功能的操作都可被使用者看作单个的实体看待，尽管这个操作实际上可能由一系列更低级的操作来完成。数据抽象定义了数据类型和施加于该类型对象上的操作，并限定了对象的值只能通过使用这些操作修改和观察。

继承是一种联结类的层次模型，并且允许和鼓励类的重用，它提供了一种明确表述共性的方法。新类继承了原始类的特性，新类称为原始类的派生类（子类），而原始类称为新类的基类（父类）。派生类可以从它的基类那里继承方法和实例变量，并且类可以修改或增加新的方法使之更适合特殊的需要。继承性很好的解决了软件的可重用性问题。

封装是把过程和数据包围起来，对数据的访问只能通过已定义的界面。面向对象计算始于这个基本概念，即现实世界可以被描绘成一系列完全自治、封装的对象，这些对象通过一个受保护的接口访问其他对象。一旦定义了一个对象的特性，则有必要决定这些特性的可见性，封装保证了模块具有较好的独立性，使得程序维护修改较为容易。对应用程序的修改仅限于类的内部，因而可以将应用程序修改带来的影响减少到最低限度。

多态性是指允许不同类的对象对同一消息作出响应。多态性包括参数化多态性和包含多态性。多态性语言具有灵活、抽象、行为共享、代码共享的优势，很好的解决了应用程序函数同名问题。**多态是依赖于接口的**。

但是，在C++使用OOP的编程方式在一些场合未能提供最高性能。现在内存存取成为计算机性能的重要瓶颈，这个问题在C++设计OOP编程范式的实现方式之初并未能考虑周全。现时的OOP编程有可能不缓存友好（cache friendly），导致有时候并不能发挥硬件最佳性能。大概就是过度封装，多态增加cache miss的可能性，数据存取时导致载入缓存的浪费等。

## OC和传统的面向对象语言有什么区别和优势
OC中方法的实现只能写在@implementation··@end中，对象方法的声明只能写在@interface···@end中间；对象方法都以-号开头，类方法都以+号开头；对象方法只能由对象来调用，类方法只能由类来调用，不能当做函数一样调用。函数属于整个文件，可以写在文件中的任何位置，包括@implementation··@end中，但写在@interface···@end会无法识别，函数的声明可以在main函数内部也可以在main函数外部。对象方法归类\对象所有；函数调用不依赖于对象；函数内部不能直接通过成员变量名访问对象的成员变量。

Objective-C的运行时是动态的，它能让你在运行时为类添加方法或者去除方法以及使用反射。 OC的动态特性表现为了三个方面：动态类型、动态绑定、动态加载。之所以叫做动态，是因为必须到运行时(run time)才会做一些事情。

动态类型，说简单点就是id类型。动态类型是跟静态类型相对的。像内置的明确的基本类型都属于静态类型(int、NSString等)。静态类型在编译的时候就能被识别出来。所以，若程序发生了类型不对应，编译器就会发出警告。而动态类型就编译器编译的时候是不能被识别的，要等到运行时(run time)，即程序运行的时候才会根据语境来识别。所以这里面就有两个概念要分清：编译时跟运行时。

动态绑定(dynamic binding)需要用到@selector/SEL。先来看看“函数”，对于其他一些静态语言，比如c++,一般在编译的时候就已经将要调用的函数的函数签名都告诉编译器了。静态的，不能改变。而在OC中，其实是没有函数的概念的，我们叫“消息机制”，所谓的函数调用就是给对象发送一条消息。这时，动态绑定的特性就来了。OC可以先跳过编译，到运行的时候才动态地添加函数调用，在运行时才决定要调用什么方法，需要传什么参数进去。这就是动态绑定，要实现他就必须用SEL变量绑定一个方法。最终形成的这个SEL变量就代表一个方法的引用。这里要注意一点：**SEL并不是C里面的函数指针**，虽然很像，但真心不是函数指针。SEL变量只是一个整数，他是该方法的ID。以前的函数调用，是根据函数名，也就是字符串去查找函数体。但现在，我们是根据一个ID整数来查找方法，整数的查找字自然要比字符串的查找快得多！所以，动态绑定的特定不仅方便，而且效率更高。

动态加载就是根据需求动态地加载资源，在运行时加载新类。在运行时创建一个新类,只需要3步:
1. 为 class pair分配存储空间 ,使用 objc_allocateClassPair函数
2. 增加需要的方法使用class_addMethod函数,增加实例变量用class_addIvar
3. 用objc_registerClassPair函数注册这个类,以便它能被别人使用。

>使用这些函数请引#import <objc/runtime.h>

## HTTP协议及HTTPS，能否保持长连接等
HTTP协议是客户端最常用到的协议了，HTTP连接使用的是“请求—响应”的方式，不仅在请求时需要先建立连接，而且需要客户端向服务器发出请求后，服务器端才能回复数据。HTTPS是以安全为目标的HTTP通道，是HTTP的安全版。 在HTTP下加入SSL层。 HTTPS存在不同于HTTP的默认端口及一个加密/身份验证层（在HTTP与TCP之间）。HTTP协议以明文方式发送内容，不提供任何方式的数据加密，如果攻击者截取了Web浏览器和网站服务器之间的传输报文，就可以直接读懂其中的信息，因此HTTP协议不适合传输一些敏感信息。http的连接很简单，是无状态的，HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议。

Request和Response的格式：
```
// 请求
GET / HTTP/1.1

Host:xxx.xxxx.com

User-Agent: Mozilla/5.0 (Windows; U; Windows NT 6.0; en-US; rv:1.9.0.10) Gecko/2016042316 Firefox/3.0.10

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8

Accept-Language: en-us,en;q=0.5

Accept-Encoding: gzip,deflate

Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7

Keep-Alive: 300

Connection: keep-alive

If-Modified-Since: Mon, 25 May 2016 03:19:18 GMT


//响应
HTTP/1.1 200 OK

Cache-Control: private, max-age=30

Content-Type: text/html; charset=utf-8

Content-Encoding: gzip

Expires: Mon, 25 May 2016 03:20:33 GMT

Last-Modified: Mon, 25 May 2016 03:20:03 GMT

Vary: Accept-Encoding

Server: Microsoft-IIS/7.0

X-AspNet-Version: 2.0.50727

X-Powered-By: ASP.NET

Date: Mon, 25 May 2016 03:20:02 GMT

Content-Length: 12173

消息体的内容（略）
```
HTTP/1.1的默认模式使用带流水线的持久连接。这种情况下，HTTP客户每碰到一个引用就立即发出一个请求，因而HTTP客户可以一个接一个紧挨着发出各个引用对象的请求。服务器收到这些请求后，也可以一个接一个紧挨着发出各个对象。如果所有的请求和响应都是紧挨着发送的，那么所有引用到的对象一共只经历1个RTT的延迟(而不是像不带流水线的版本那样，每个引用到的对象都各有1个RTT的延迟)。另外，带流水线的持久连接中服务器空等请求的时间比较少。与非持久连接相比，持久连接(不论是否带流水线)除降低了1个RTT的响应延迟外，缓启动延迟也比较小。其原因在于既然各个对象使用同一个TCP连接，服务器发出第一个对象后就不必再以一开始的缓慢速率发送后续对象。相反，服务器可以按照第一个对象发送完毕时的速率开始发送下一个对象。

参考书目：《HTTP权威指南》
