# objc

## Objective-C对象消息名关键词

我们在写Objective－C的代码时，在进行某个动作（action）时，会发送一些相关联的消息。经常会遇到以下的一些关键词：

- should 决定某个动作是否要发生，如果返回NO，则不会执行这个动作，也不会有will和did消息（下面将会说明）了。如shouldAutorotateToInterfaceOrientation:
- will 通常在某个动作发生之前，如viewWillAppear、viewWillDisappear等
- did 通常在某个动作发生之后，如viewDidAppear，didAddSubView等

> 所以它们的调用顺序依次是：should->will->action->did 写代码时一定要搞清这个顺序，否则很容易出现逻辑错误。

## OC中load方法和initialize方法的异同

**对于load方法，官方的文档说明如下：**

Invoked whenever a class or category is added to the Objective-C runtime; implement this method to perform class-specific behavior upon loading. The load message is sent to classes and categories that are both dynamically loaded and statically linked, but only if the newly loaded class or category implements a method that can respond.

The order of initialization is as follows:

- All initializers in any framework you link to.
- All +load methods in your image.
- All C++ static initializers and C/C++ **attribute**(constructor) functions in your image.
- All initializers in frameworks that link to you. In addition:

- A class's +load method is called after all of its superclasses' +load methods.

- A category +load method is called after the class's own +load method. In a custom implementation of load you can therefore safely message other unrelated classes from the same image, but any load methods implemented by those classes may not have run yet.

文档也说清楚了，**对于load方法，只要文件被引用就会被调用。load方法调用顺序是父类的load方法 优先调用于子类的load方法，而本类的load方法优先于category调用**。 **对于+initialize方法，官方的文档说明如下：** Initializes the class before it receives its first message.

The runtime sends initialize to each class in a program just before the class, or any class that inherits from it, is sent its first message from within the program. The runtime sends the initialize message to classes in a thread-safe manner. Superclasses receive this message before their subclasses. The superclass implementation may be called multiple times if subclasses do not implement initialize--the runtime will call the inherited implementation--or if subclasses explicitly call [super initialize]. If you want to protect yourself from being run multiple times, you can structure your implementation along these lines:

```objective-c
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

Because initialize is called in a **thread-safe** manner and the order of initialize being called on different classes is not guaranteed, it's important to do the minimum amount of work necessary in initialize methods.

Specifically, any code that takes locks that might be required by other classes in their initialize methods is liable to lead to **deadlocks**.

Therefore you should not rely on initialize for complex initialization, and should instead limit it to straightforward, class local initialization. initialize is invoked **only once per class**. If you want to perform independent initialization for the class and for categories of the class, you should implement load methods.

**文档也很明确的说明了：**文件被引用并不代表initialize就会被调用，只有类或者子类中第一次有函数调用时， 都会调用initialize。initialize是线程安全的，我们不能在initialize方法中加锁，这有可能导致死锁。 我们也不应该在函数中实现复杂的代码。initialize只会被调用一次。

**+load和+initialize共同点：**

- 在不考虑开发者主动使用的情况下，系统最多会调用一次
- 如果父类和子类都被调用，父类的调用一定在子类之前
- 这两个方法不适合做复杂的操作，应该是足够简单
- 在使用时都不要过重地依赖于这两个方法，除非真正必要。在使用时一定要注意防止死锁！
- 都不需要调用[super load]、[super initialize]

**+load和+initialize不同点：**

- load方法没有自动释放池，如果做数据处理，需要释放内存，则开发者得自己添加autoreleasepool来管理内存的释放。
- 和load不同，即使子类不实现initialize方法，也会把父类的实现继承过来调用一遍。注意的是在此之前， 父类的方法已经被执行过一次了，同样不需要super调用。

  ## Fast Enumeration 的实现原理

  [Objective-C Fast Enumeration 的实现原理](http://blog.leichunfeng.com/blog/2016/06/20/objective-c-fast-enumeration-implementation-principle/)

## Object-C有私有方法吗？私有变量呢？

objective-c – 类里面的方法只有两种, 静态方法和实例方法. 这似乎就不是完整的面向对象了, 按照OO的原则就是一个对象只暴露有用的东西. 如果没有了私有方法的话, 对于一些小范围的代码重用就不那么顺手了. 在类里面声名一个私有方法

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

## 对block的理解

Block分为三种，分别是全局block、栈block和堆block。ARC之后，我们并不需要手动copy到堆上， 通常都已经交给编译器来完成。 1). 使用block和使用delegate完成委托模式有什么优点?

首先要了解什么是委托模式，委托模式在iOS中大量应用，其在设计模式中是适配器模式中的对象适配器， Objective-C中使用id类型指向一切对象，使委托模式更为简洁。了解委托模式的细节：

iOS设计模式---委托模式

使用block实现委托模式，其优点是回调的block代码块定义在委托对象函数内部，使代码更为紧凑;

适配对象不再需要实现具体某个protocol，代码更为简洁。

2). 多线程与block

GCD与Block

使用 dispatch_async 系列方法，可以以指定的方式执行block

GCD编程实例

dispatch_async的完整定义

void?dispatch_async( dispatch_queue_t?queue, dispatch_block_t?block); 功能：在指定的队列里提交一个异步执行的block，不阻塞当前线程

通过queue来控制block执行的线程。主线程执行前文定义的 finishBlock对象

dispatch_async(dispatch_get_main_queue(),^(void){finishBlock();});

## 为什么其他语言里叫函数调用， objective c里则是给对象发消息（或者谈下对runtime的理解）

先来看看怎么理解发送消息的含义：

曾经觉得Objc特别方便上手，面对着 Cocoa 中大量 API，只知道简单的查文档和调用。还记得初学 Objective-C 时把[receiver message]当成简单的方法调用，而无视了"发送消息"这句话的深刻含义。 于是[receiver message]会被编译器转化为： objc_msgSend(receiver, selector) 如果消息含有参数，则为： `objc_msgSend(receiver, selector, arg1, arg2, ...)`

如果消息的接收者能够找到对应的selector，那么就相当于直接执行了接收者这个对象的特定方法； 否则，消息要么被转发，或是临时向接收者动态添加这个selector对应的实现内容，要么就干脆玩完崩溃掉。

现在可以看出[receiver message]真的不是一个简简单单的方法调用。因为这只是在编译阶段确定了 要向接收者发送message这条消息，而receive将要如何响应这条消息，那就要看运行时发生的情况来决定了。

Objective-C 的 Runtime 铸就了它动态语言的特性，这些深层次的知识虽然平时写代码用的少一些， 但是却是每个 Objc 程序员需要了解的。

Objc Runtime使得C具有了面向对象能力，在程序运行时创建，检查，修改类、对象和它们的方法。 可以使用runtime的一系列方法实现。

顺便附上OC中一个类的数据结构 /usr/include/objc/runtime.h

```objc
struct objc_class {
    Class isa OBJC_ISA_AVAILABILITY; //isa指针指向Meta Class，
    因为Objc的类的本身也是一个Object，为了处理这个关系，runtime就创造了Meta Class，
    当给类发送[NSObject alloc]这样消息时，实际上是把这个消息发给了Class Object

    #if !__OBJC2__
    Class super_class OBJC2_UNAVAILABLE; // 父类
    const char *name OBJC2_UNAVAILABLE; // 类名
    long version OBJC2_UNAVAILABLE; // 类的版本信息，默认为0
    long info OBJC2_UNAVAILABLE; // 类信息，供运行期使用的一些位标识
    long instance_size OBJC2_UNAVAILABLE; // 该类的实例变量大小
    struct objc_ivar_list *ivars OBJC2_UNAVAILABLE; // 该类的成员变量链表
    struct objc_method_list **methodLists OBJC2_UNAVAILABLE; // 方法定义的链表
    struct objc_cache *cache OBJC2_UNAVAILABLE; // 方法缓存，对象接到一个消息会根据isa
    指针查找消息对象，这时会在method       Lists中遍历，如果cache了，常用的方法调用时就能够提高调用的效率。
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

向object发送消息时，Runtime库会根据object的isa指针找到这个实例object所属于的类， 然后在类的方法列表以及父类方法列表寻找对应的方法运行。id是一个objc_object结构类型的指针， 这个类型的对象能够转换成任何一种对象。

然后再来看看消息发送的函数：objc_msgSend函数

在引言中已经对objc_msgSend进行了一点介绍，看起来像是objc_msgSend返回了数据，其实objc_msgSend 从不返回数据而是你的方法被调用后返回了数据。下面详细叙述下消息发送步骤：

检测这个 selector 是不是要忽略的。比如 Mac OS X 开发，有了垃圾回收就不理会 retain,release 这些函数了。 检测这个 target 是不是 nil 对象。ObjC 的特性是允许对一个 nil 对象执行任何一个方法不会 Crash，因为会被忽略掉。 如果上面两个都过了，那就开始查找这个类的 IMP，先从 cache 里面找，完了找得到就跳到对应的函数去执行。 如果 cache 找不到就找一下方法分发表。 如果分发表找不到就到超类的分发表去找，一直找，直到找到NSObject类为止。 如果还找不到就要开始进入动态方法解析了，后面会提到。

后面还有： 动态方法解析resolveThisMethodDynamically 消息转发forwardingTargetForSelector

## 对runtime的理解

1. 消息是如何转发的？ 动态解析过程大致是这样的：通过resolveInstanceMethod允许开发者决定是否动态添加方法，若返回NO， 就直接进入doesNotRecognizeSelector，流程结束，否则需要通过class_addMethod动态添加方法 并返回YES并进入下一步。forwardingTargetForSelector是第二步，允许开发者决定将由哪个对象响应这个selector， 如果返回nil，则直接进入doesNotRecognizeSelector，流程结束，否则需要返回一个对象，但不能是self。 进入第三步指定方法签名methodSignatureForSelector，若返回nil， 则直接进入doesNotRecognizeSelector且流程结束，否则指定签名，并进入下一步forwardInvocation。 forwardInvocation允许开发者修改响应者、方法实现等。若没有实现forwardInvocation， 则直接进入doesNotRecognizeSelector，流程结束。

2. 方法调用会被缓存吗？如何缓存过，又是如何查找的呢？ 方法是会缓存进来了，不然下次再调用又要重新查一次，效率是不高的。采用散列（哈希）的方式来缓存， 查询的效率是比较高的，因此内部会采用散列缓存起来。

3. 对象的内存是如何布局的？ 成员变量（包括父类）都保存在对象本身的存储空间内；本类的实例方法保存在类对象中， 本类的类方法保存在元类对象中；父类的实例方法保存在各级super class中， 父类的类方法保存在各级super meta class中。

4. runtime有哪些应用场景？

5. 给category添加属性

6. Method-Swizzling hook方法，然后交换方法实现来达到调用系统方法之前先做一些额外的处理

7. 埋点处理

8. 字典与模型互转

9. 模型自动获取所有属性并转换成SQL语句操作数据库

## oc是动态运行时语言是什么意思?

多态。 主要是将数据类型的确定由编译时，推迟到了运行时。 这个问题其实浅涉及到两个概念，运行时和多态。 简单来说，运行时机制使我们直到运行时才去决定一个对象的类别，以及调用该类别对象指定方法。 多态：不同对象以自己的方式响应相同的消息的能力叫做多态。意思就是假设生物类(life)都用有一个相同的方法-eat; 那人类属于生物，猪也属于生物，都继承了life后，实现各自的eat，但是调用是我们只需调用各自的eat方法。 也就是不同的对象以自己的方式响应了相同的消息(响应了eat这个选择器)。

### 关于多态性

多态，子类指针可以赋值给父类。 这个题目其实可以出到一切面向对象语言中， 因此关于多态，继承和封装基本最好都有个自我意识的理解，也并非一定要把书上资料上写的能背出来

## __block在arc和非arc下含义一样吗？

是不一样的。 在MRC中**block variable在block中使用是不会retain的 但是ARC中**block则会Retain。 取而代之的是用**weak或是**unsafe_unretained來更精确的描述weak reference的目的 其中前者只能在iOS5之后可以使用，但是比较好 (该对象release之后，此pointer会自动设成成nil) 而后者是ARC的环境下为了兼容4.x的解決方案。

```objc
__block MyClass* temp = …;    // MRC环境下使用
__weak MyClass* temp = …;    // ARC但只支援iOS5.0以上的版本
__unsafe_retained MyClass* temp = …;  //ARC且可以兼容4.x以后的版本
```

## block 实现原理

Objective-C是对C语言的扩展，block的实现是基于指针和函数指针。

从计算语言的发展，最早的goto，高级语言的指针，到面向对象语言的block，从机器的思维，一步步接近人的思维，以方便开发人员更为高效、直接的描述出现实的逻辑(需求)。

使用实例

cocoaTouch框架下动画效果的Block的调用

使用typed声明block

typedef?void(^didFinishBlock)?(NSObject?*ob); 这就声明了一个didFinishBlock类型的block， 然后便可用

@property?(nonatomic,copy)?didFinishBlock?finishBlock; 声明一个blokc对象，注意对象属性设置为copy，接到block 参数时，便会自动复制一份。

__block是一种特殊类型，

使用该关键字声明的局部变量，可以被block所改变，并且其在原函数中的值会被改变。

## 对MVC和MVVM的理解 你还熟悉什么设计模式？

MVC是出现比较早的架构设计模式，而且到现在已经是很成熟了。出现MVVM的原因是MVC中的V越来越复杂， 于是才有人想要给V瘦身。

> 设计模式：并不是一种新技术，而是一种编码经验，使用比如java中的接口，iphone中的协议， 继承关系等基本手段，用比较成熟的逻辑去处理某一种类型的事情，总结为所谓设计模式。 面向对象编程中，java已经归纳了23种设计模式。

- mvc设计模式 ：模型，视图，控制器，可以将整个应用程序在思想上分成三大块，对应是的数据的存储或处理， 前台的显示，业务逻辑的控制。 Iphone本身的设计思想就是遵循mvc设计模式。其不属于23种设计模式范畴。

- 代理模式：代理模式给某一个对象提供一个代理对象，并由代理对象控制对源对象的引用. 比如一个工厂生产了产品，并不想直接卖给用户，而是搞了很多代理商，用户可以直接找代理商买东西， 代理商从工厂进货.常见的如QQ的自动回复就属于代理拦截，代理模式在iphone中得到广泛应用.

- 单例模式：说白了就是一个类不通过alloc方式创建对象，而是用一个静态方法返回这个类的对象。 系统只需要拥有一个的全局对象，这样有利于我们协调系统整体的行为， 比如想获得[UIApplication sharedApplication];任何地方调用都可以得到 UIApplication的对象， 这个对象是全局唯一的。

- 观察者模式： 当一个物体发生变化时，会通知所有观察这个物体的观察者让其做出反应。 实现起来无非就是把所有观察者的对象给这个物体，当这个物体的发生改变， 就会调用遍历所有观察者的对象调用观察者的方法从而达到通知观察者的目的。

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

- 为单例对象实现一个静态实例，并初始化，然后设置成nil。

- 实现一个实例构造方法检查上面声明的静态实例是否为nil，如果是则新建并返回一个本类的实例，

- 重写allocWithZone方法，用来保证其他人直接使用alloc和init试图获得一个新实力的时候不产生一个新实例，

- 适当实现allocWitheZone，copyWithZone，release和autorelease。

## 如何使用Xcode设计通用应用?

使用MVC模式设计应用，其中Model层完成脱离界面，即在Model层，其是可运行在任何设备上， 在controller层，根据iPhone与iPad(独有UISplitViewController)的不同特点选择不同的viewController对象。 在View层，可根据现实要求，来设计，其中以xib文件设计时，其设置其为universal。

## 对ARC的理解

ARC是编译器帮我们完成的，我们不再手动添加retain、relase、autorelease， 而且在运行期还会帮助我们优化。但是ARC并不是万能的，它并不能自我理解循环引用问题， 依然需要我们手动解决循环引用的问题。

ARC管理都会放到自动释放池中，如果我们需要做一些循环操作，生成大量的临时变量， 我们还是需要加一下autoreleasepool，以及时地释放内存。

ARC下对于属性修饰符不同，其内存管理策略也不一样：

- strong：强引用，引用计数加1
- weak：弱引用，引用计数没有加1
- copy：强引用，引用计数加1

ARC下还是有可能出现内存泄露的，内存得不到释放，特别是使用block的时候，一定要学会分析是否形成循环引用。

## __weak

当一个__weak 类型的指针指向的对象被释放时,该指针会自动被置成nil.

```objc
id __weak obj1 = obj;
```

会转化为

```objc
id obj1;  
objc_initWeak(&obj1, obj);  
objc_destoryWeak(&obj1);
```

即编译器会通过objc_initWeak函数初始化__weak修饰的变量，当变量的作用域结束后会通过objc_destoryWeak函数释放该变量。objc_initWeak函数实际干的活是：

```objc
objc1 = 0;  
objc_storeWeak(&obj1, obj);
```

这里是先将指针objc1置成0，再调用objc_storeWeak函数使得obj1指向obj对象。 接下来的objc_destoryWeak函数的实际操作如下：

```objc
objc_storeWeak(&obj1, 0);
```

也就是说，让obj1指针指向的内容变成空。

**__weak实现原理**

实际上，objc_storeWeak函数会把第二个参数的对象的地址作为key，并将第一个参数（__weak关键字修饰的指针的地址）作为值，注册到weak表中。如果第二个参数为0（说明对应的对象被释放了），则将weak表中将整个key-value键值对删除，这就是__weak关键字的核心思想！

weak表和引用计数表类似，都是通过hash表实现的。如果使用weak表，将被释放的对象地址作为key去检索，就能很高效的获取对应的指向该对象的类型为__weak的指针变量的地址。同时很容易理解，一个对象可能有多个__weak指针指向，因此一个对象地址key可能对应多个值。

在调用对象的release方法时，会在其中一步调用objc_clear_deallocating函数，该函数会执行以下操作：以当前对象的地址作为key，从weak表中获取对应的值----指向该对象的__weak类型的指针变量；将取到的所有指针变量的值赋值为nil；从weak表中删除该key对应的整条记录。

如果大量使用附有__weak修饰符的变量会消耗响应的CPU资源，因此，应该尽量少使用__weak修饰符.

## 野指针是什么，iOS开发中什么情况下会有野指针？

所谓野指针，是指指向内存已经被释放的内存区的指针。

当进入播放页面时马上又返回上一个页面，偶尔出现闪退，原因就是出现了野指针（访问了已释放的对象内存区。 当进入播放页面时，就会立刻去解析视频数据，内部是FFMPEG操作，当快速返回上一个页面时， FFMPEG还在操作中，导致访问了已释放的对象。 使用block时，也会出现野指针。

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

## Object-c的类可以多重继承么?可以实现多个接口么?Category是什么?

重写一个类的方式用继承好还是分类好?为什么? Object-c的类不可以多重继承;可以实现多个接口，通过实现多个接口可以完成C++的多重继承;Category是类别， 一般情况用分类好，用Category去重写类的方法，仅对本Category有效，不会影响到其他类与原有类的关系。

## #import 跟#include 又什么区别，@class呢, #import<> 跟 #import""又什么区别

# import是Objective-C导入头文件的关键字，#include是C/C++导入头文件的关键字, 使用#import头文件会自动只导入一次，不会重复导入，相当于#include和#pragma once; @class告诉编译器某个类的声明，当执行时，才去查看类的实现文件，可以解决头文件的相互包含;

# import<>用来包含系统的头文件，#import""用来包含用户头文件。

## Object-C有多继承吗？没有的话用什么代替？cocoa 中所有的类都是NSObject 的子类

多继承在这里是用protocol 委托代理 来实现的

你不用去考虑繁琐的多继承 ,虚基类的概念.

ood的多态特性 在 obj-c 中通过委托来实现.

## 对于语句NSString*obj = [[NSData alloc] init]; obj在编译时和运行时分别时什么类型的对象?

编译时是NSString的类型;运行时是NSData类型的对象

## 常见的object-c的数据类型有那些， 和C的基本数据类型有什么区别?如：NSInteger和int

object-c的数据类型有NSString，NSNumber，NSArray，NSMutableArray，NSData等等， 这些都是class，创建后便是对象，而C语言的基本数据类型int，只是一定字节的内存空间， 用于存放数值;NSInteger是基本数据类型，并不是NSNumber的子类，当然也不是NSObject的子类。 NSInteger是基本数据类型Int或者Long的别名(NSInteger的定义typedef long NSInteger)， 它的区别在于，NSInteger会根据系统是32位还是64位来决定是本身是int还是Long。

## id 声明的对象有什么特性?

Id 声明的对象具有运行时的特性，即可以指向任意类型的objcetive-c的对象;

## Objective-C如何对内存管理的,说说你的看法和解决方法?

Objective-C的内存管理主要有三种方式ARC(自动内存计数)、手动内存计数、内存池。

1. (Garbage Collection)自动内存计数：这种方式和java类似，在你的程序的执行过程中。 始终有一个高人在背后准确地帮你收拾垃圾，你不用考虑它什么时候开始工作，怎样工作。你只需要明白， 我申请了一段内存空间，当我不再使用从而这段内存成为垃圾的时候，我就彻底的把它忘记掉， 反正那个高人会帮我收拾垃圾。遗憾的是，那个高人需要消耗一定的资源，在携带设备里面， 资源是紧俏商品所以iPhone不支持这个功能。所以"Garbage Collection"不是本入门指南的范围， 对"Garbage Collection"内部机制感兴趣的同学可以参考一些其他的资料， 不过说老实话"Garbage Collection"不大适合适初学者研究。

解决: 通过alloc – initial方式创建的, 创建后引用计数+1, 此后每retain一次引用计数+1, 那么在程序中做相应次数的release就好了.

1. (Reference Counted)手动内存计数：就是说，从一段内存被申请之后， 就存在一个变量用于保存这段内存被使用的次数，我们暂时把它称为计数器，当计数器变为0的时候， 那么就是释放这段内存的时候。比如说，当在程序A里面一段内存被成功申请完成之后， 那么这个计数器就从0变成1(我们把这个过程叫做alloc)，然后程序B也需要使用这个内存， 那么计数器就从1变成了2(我们把这个过程叫做retain)。紧接着程序A不再需要这段内存了， 那么程序A就把这个计数器减1(我们把这个过程叫做release);程序B也不再需要这段内存的时候， 那么也把计数器减1(这个过程还是release)。当系统(也就是Foundation)发现这个计数器变 成员了0， 那么就会调用内存回收程序把这段内存回收(我们把这个过程叫做dealloc)。 顺便提一句，如果没有Foundation，那么维护计数器，释放内存等等工作需要你手工来完成。

解决:一般是由类的静态方法创建的, 函数名中不会出现alloc或init字样, 如[NSString string]和[NSArray arrayWithObject:], 创建后引用计数+0, 在函数出栈后释放, 即相当于一个栈上的局部变量. 当然也可以通过retain延长对象的生存期.

1. (NSAutoRealeasePool)内存池：可以通过创建和释放内存池控制内存申请和回收的时机.

解决:是由autorelease加入系统内存池, 内存池是可以嵌套的, 每个内存池都需要有一个创建释放对, 就像main函数中写的一样. 使用也很简单, 比如[[[NSString alloc]initialWithFormat:@"Hey you!"] autorelease], 即将一个NSString对象加入到最内层的系统内存池, 当我们释放这个内存池时, 其中的对象都会被释放.

## 使用nonatomic一定是线程安全的吗？

不是的。 atomic原子操作，系统会为setter方法加锁。 具体使用 @synchronized(self){//code } nonatomic不会为setter方法加锁。 atomic：线程安全，需要消耗大量系统资源来为属性加锁 nonatomic：非线程安全，适合内存较小的移动设备

## 原子(atomic)跟非原子(non-atomic)属性有什么区别?

1. atomic提供多线程安全。是防止在写未完成的时候被另外一个线程读取，造成数据错误
2. non-atomic:在自己管理内存的环境中，解析的访问器保留并自动释放返回的值， 如果指定了 nonatomic ，那么访问器只是简单地返回这个值。

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

str的retainCount创建+1，retain+1，加入数组自动+1 3 retain+1，release-1，release-1 2 数组删除所有对象，所有数组内的对象自动-1 1

## 内存管理的几条原则是什么?按照默认法则.那些关键字生成的对象需要手动释放?在和property结合的时候怎样有效的避免内存泄露?

谁申请，谁释放 遵循Cocoa Touch的使用原则; 内存管理主要要避免"过早释放"和"内存泄漏"，对于"过早释放"需要注意@property设置特性时， 一定要用对特性关键字，对于"内存泄漏"，一定要申请了要负责释放，要细心。 关键字alloc 或new 生成的对象需要手动释放; 设置正确的property属性，对于retain需要在合适的地方释放，

## Object C中创建线程的方法是什么?如果在主线程中执行代码，

方法是什么?如果想延时执行代码、方法又是什么? 线程创建有三种方法：使用NSThread创建、使用GCD的dispatch、使用子类化的NSOperation, 然后将其加入NSOperationQueue;在主线程执行代码，方法是`performSelectorOnMainThread`， 如果想延时执行代码可以用`performSelector:onThread:withObject:waitUntilDone:`

## Category

Category用于向已经存在的类添加方法从而达到扩展已有类的目的，在很多情形下Category也是比创建子类更优的选择。 Category用于大型类有效分解。新添加的方法会被被扩展的类的所有子类自动继承。 Category也可以用于替代这个已有类中某个方法的实体，从而达到修复BUG的目的。 如此就不能去调用已有类中原有的那个被替换掉方法实体了。需要注意的是，当准备有Category来替换某一个方法的时候， 一定要保证实现原来方法的所有功能，否则这种替代就是没有意义而且会引起新的BUG。

Category的方法不一定非要在@implementation中实现，也可以在其他位置实现， 但是当调用Category的方法时，依据继承树没有找到该方法的实现，程序则会崩溃。Category理论上不能添加变量， 但是可以使用@dynamic 来弥补这种不足。 ```objc @implementation NSObject (Category) @dynamic variable;

- (id) variable { return objc_getAssociatedObject(self, externVariableKey); }
- (void)setVariable:(id) variable { objc_setAssociatedObject(self, externVariableKey, variable, OBJC_ASSOCIATION_RETAIN_NONATOMIC); }

````
和子类不同的是，Category不能用于向被扩展类添加实例变量。Category通常作为一种组织框架代码的工具来使用。
如果需要添加一个新的变量，则需添加子类。如果只是添加一个新的方法，用Category是比较好的选择。

## 在Category中实现属性
做开发时我们常常会需要在已经实现了的类中增加一些方法，这时候我们一般会用Category的方式来做。但是这样做我们也只能扩展一些方法，而有时候我们更多的是想给它增加一个属性。由于类已经是编译好的了，就不能静态的增加成员了，这样我们就需要自己来实现getter和setter方法了，在这些方法中动态的读写属性变量来实现属性。一种比较简单的做法是使用Objective-C运行时的这两个方法：
```objc
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
id objc_getAssociatedObject(id object, const void *key);
```
这两个方法可以让一个对象和另一个对象关联，就是说一个对象可以保持对另一个对象的引用，并获取那个对象。有了这些，就能实现属性功能了。
policy可以设置为以下这些值：
```objc
enum {
    OBJC_ASSOCIATION_ASSIGN = 0,
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,
    OBJC_ASSOCIATION_RETAIN = 01401,
    OBJC_ASSOCIATION_COPY = 01403
};
```
这些值跟属性定义中的nonatomic，copy，retain等关键字的功能类似。
下面是一个属性自定义getter和setter的例子：
```objc
NSString const * kExposeController = @"exposeController";

- (UIViewController *)exposeController {
    return (UIViewController *)objc_getAssociatedObject(self, kExposeController);
}

- (void)setExposeController:(UIViewController *)exposeController {
    objc_setAssociatedObject(self, kExposeController, exposeController, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```
可以看出使用objc_setAssociatedObject和objc_getAssociatedObject函数可以很方便的实现属性的getter和setter。
**一个很方便的宏**
为此，我特意写了一个Synthesize宏，可以提供@synthesize类似的功能。可以支持两种最常用的属性：非原子retain和assign属性（如果需要其他类型的属性可自行修改）。
```objc
#import <objc/runtime.h>
#define SYNTHESIZE_CATEGORY_OBJ_PROPERTY(propertyGetter, propertySetter)                                                             
- (id) propertyGetter {                                                                                                             
    return objc_getAssociatedObject(self, @selector( propertyGetter ));                                                             
}                                                                                                                                   
- (void) propertySetter (id)obj{                                                                                                    
    objc_setAssociatedObject(self, @selector( propertyGetter ), obj, OBJC_ASSOCIATION_RETAIN_NONATOMIC);                            
}


#define SYNTHESIZE_CATEGORY_VALUE_PROPERTY(valueType, propertyGetter, propertySetter)                                                
- (valueType) propertyGetter {                                                                                                      
    valueType ret = {0};                                                                                                                  
    [objc_getAssociatedObject(self, @selector( propertyGetter )) getValue:&ret];                                                    
    return ret;                                                                                                                     
}                                                                                                                                   
- (void) propertySetter (valueType)value{                                                                                           
    NSValue *valueObj = [NSValue valueWithBytes:&value objCType:@encode(valueType)];                                                
    objc_setAssociatedObject(self, @selector( propertyGetter ), valueObj, OBJC_ASSOCIATION_RETAIN_NONATOMIC);                       
}
```
用这个宏只需要指定相关属性的类型，getter和setter就可以快速的实现一个属性。比如在UIAlert的Category实现一个非原子retain属性userInfo，以及一个assign的类型为CGRect的customArea属性：

UIAlertView+Ex.h
```objc
@interface UIAlertView (Ex)
@property(nonatomic, retain) id userInfo;
@property(nonatomic) CGRect customArea;
@end
```
UIAlertView+Ex.m
```objc
@implementation UIAlertView (Ex)
SYNTHESIZE_CATEGORY_OBJ_PROPERTY(userInfo, setUserInfo:)
SYNTHESIZE_CATEGORY_VALUE_PROPERTY(CGRect, customArea, setCustomArea:)
@end
```
**runtime对category的加载过程**
下面是runtime中category的结构：
```objc
struct _category_t {
const char *name; // 类的名字
struct _class_t *cls; // 要扩展的类对象，编译期间这个值是不会有的，
在app被runtime加载时才会根据name对应到类对象
const struct _method_list_t *instance_methods; // 实例方法
const struct _method_list_t *class_methods; // 类方法
const struct _protocol_list_t *protocols; // 这个category实现的protocol，
比较不常用在category里面实现协议，但是确实支持的
const struct _prop_list_t *properties; // 这个category所有的property，
这也是category里面可以定义属性的原因，不过这个property不会@synthesize实例变量，
一般有需求添加实例变量属性时会采用objc_setAssociatedObject和objc_getAssociatedObject方法绑定方法绑定，
不过这种方法生成的与一个普通的实例变量完全是两码事。
};
````

category动态扩展了原来类的方法，在调用者看来好像原来类本来就有这些方法似的， 不论有没有import category 的.h，都可以成功调用category的方法，都影响不到category的加载流程， import只是帮助了编译检查和链接过程。runtime加载完成后，category的原始信息在类结构里将不会存在。

objc runtime的加载入口是一个叫_objc_init的方法，在library加载前由libSystem dyld调用， 进行初始化操作。调用map_images方法将文件中的image map到内存。调用_read_images方法初始化map后的image， 这里面干了很多的事情，像load所有的类、协议和category，著名的+ load方法就是这一步调用的。 category的初始化，循环调用了_getObjc2CategoryList方法。

在调用完_getObjc2CategoryList后，runtime终于开始了category的处理，首先分成两拨， 一拨是实例对象相关的调用addUnattachedCategoryForClass，一拨是类对象相关的调用addUnattachedCategoryForClass， 然后会调到attachCategoryMethods方法，这个方法把一个类所有的category_list的所有方法取出来组成一个method_list_t ， 这里是倒序添加的，也就是说，新生成的category的方法会先于旧的category的方法插入。

生成了所有method的list之后，调用attachMethodLists将所有方法前序添加进类的方法的数组中， 也就是说，如果原来类的方法是a,b,c，类别的方法是1,2,3，那么插入之后的方法将会是1,2,3,a,b,c， 也就是说，原来类的方法被category的方法覆盖了，但被覆盖的方法确实还在那里。

```objc
static void attachCategoryMethods(class_t *cls, category_list *cats,
                  BOOL *inoutVtablesAffected)
{
if (!cats) return;
if (PrintReplacedMethods) printReplacements(cls, cats);

BOOL isMeta = isMetaClass(cls);
method_list_t **mlists = (method_list_t **)
    _malloc_internal(cats->count * sizeof(*mlists));

// Count backwards through cats to get newest categories first
int mcount = 0;
int i = cats->count;
BOOL fromBundle = NO;
while (i--) {
    method_list_t *mlist = cat_method_list(cats->list[i].cat, isMeta);
    if (mlist) {
        mlists[mcount++] = mlist;
        fromBundle |= cats->list[i].fromBundle;
    }
}

attachMethodLists(cls, mlists, mcount, NO, fromBundle, inoutVtablesAffected);

_free_internal(mlists);

}
```

这也即是我们上面说的Category修复Bug的原理。 **Extension** Extension非常像是没有命名的类别。扩展只是用来定义类的私有方法的，实现要在原始的.m里面。 还以用来改变原始属性的一些性质。一般的时候，Extension都是放在.m文件中@implementation的上方。 Extension中的方法必须在@implementation中实现，否则编译会报错。Category没有源代码的类添加方法， 格式：定义一对.h和.m。Extension作用于管理类的所有方法，格式：把代码写到原始类的.m文件中。

## 类别(category)的作用?继承和类别在实现中有何区别?

category 可以在不获悉，不改变原来代码的情况下往里面添加新的方法，只能添加，不能删除修改， 并且如果类别和原来类中的方法产生名称冲突，则类别将覆盖原来的方法，因为类别具有更高的优先级。

类别主要有3个作用：

1. 将类的实现分散到多个不同文件或多个不同框架中。
2. 创建对私有方法的前向引用。
3. 向对象添加非正式协议。 继承可以增加，修改或者删除方法，并且可以增加属性。

## 类别和类扩展的区别

  category和extensions的不同在于 后者可以添加属性。另外后者添加的方法是必须要实现的。 extensions可以认为是一个私有的Category。

## oc中的协议和java中的接口概念有何不同?

OC中的代理有2层含义，官方定义为 formal和informal protocol。前者和Java接口一样。 informal protocol中的方法属于设计模式考虑范畴，不是必须实现的，但是如果有实现，就会改变类的属性。 其实关于正式协议，类别和非正式协议我很早前学习的时候大致看过，也写在了学习教程里 "非正式协议概念其实就是类别的另一种表达方式"这里有一些你可能希望实现的方法，你可以使用他们更好的完成工作"。 这个意思是，这些是可选的。比如我门要一个更好的方法，我们就会申明一个这样的类别去实现。然后你在后期可以直接使用这些更好的方法。 这么看，总觉得类别这玩意儿有点像协议的可选协议。" 现在来看，其实protocal已经开始对两者都统一和规范起来操作，因为资料中说"非正式协议使用interface修饰"， 现在我们看到协议中两个修饰词："必须实现(@requied)"和"可选实现(@optional)"。

## KVC & KVO

KVC:键 – 值编码是一种间接访问对象的属性使用字符串来标识属性，而不是通过调用存取方法，直接或通过实例变量访问的机制。

很多情况下可以简化程序代码。apple文档其实给了一个很好的例子。

KVO:键值观察机制，他提供了观察某一属性变化的方法，极大的简化了代码。

具体用看到嗯哼用到过的一个地方是对于按钮点击变化状态的的监控。

比如我自定义的一个button

```objc
[self addObserver:self forKeyPath:@"highlighted" options:0 context:nil];
#pragma mark?KVO
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object
change:(NSDictionary *)change context:(void *)context
{
if ([keyPath isEqualToString:@"highlighted"]) {
[self setNeedsDisplay];
}
}
```

对于系统是根据keypath去取的到相应的值发生改变，理论上来说是和kvc机制的道理是一样的。

对于kvc机制如何通过key寻找到value：

"当通过KVC调用对象时，比如：[self valueForKey:@"someKey"]时，程序会自动试图通过几种不同的 方式解析这个调用。首先查找对象是否带有 someKey 这个方法，如果没找到，会继续查找对象是否带 有someKey这个实例变量(iVar)，如果还没有找到，程序会继续试图调用 -(id) valueForUndefinedKey:这个方法。 如果这个方法还是没有被实现的话，程序会抛出一个NSUndefinedKeyException异常错误。

Key-Value Coding查找方法的时候，不仅仅会查找someKey这个方法，还会查找getsomeKey这个方法， 前面加一个get，或者_someKey以及_getsomeKey这几种形式。同时， 查找实例变量的时候也会不仅仅查找someKey这个变量，也会查找_someKey这个变量是否存在。

## 代理(Delegate)的作用?

> 代理的目的是改变或传递控制链。

- 允许一个类在某些特定时刻通知到其他类，而不需要获取到那些类的指针。可以减少框架复杂度。
- 代理可以理解为java中的回调监听机制的一种类似。

## 通知和协议的不同之处?

协议有控制链(has-a)的关系，通知没有。 delegate针对one-to-one关系，用于sender接受到reciever的某个功能反馈值。

notification针对one-to-one/many/none,reciver,用于通知多个object某个事件。

## 什么是推送(push)消息?

推送通知更是一种技术。 简单点就是客户端获取资源的一种手段。 普通情况下，都是客户端主动的pull。 推送则是服务器端主动push。

## Respond chain

事件响应链。包括点击事件，画面刷新事件等。在视图栈内从上至下，或者从下之上传播。 可以说点事件的分发，传递以及处理。具体可以去看下touch事件这块。 可以从责任链模式，来讲通过事件响应链处理，其拥有的扩展性

## OC的垃圾回收机制?

OC2.0有Garbage collection，但是iOS平台不提供。 一般我们了解的objective-c对于内存管理都是手动操作的，但是也有自动释放池。 但是差了大部分资料，貌似不要和arc机制搞混就好了。

## NSOperation queue?

存放NSOperation的集合类。 操作和操作队列，基本可以看成java中的线程和线程池的概念。用于处理ios多线程开发的问题。 网上部分资料提到一点是，虽然是queue，但是却并不是带有队列的概念，放入的操作并非是按照严格的先进现出。 这边又有个疑点是，对于队列来说，先进先出的概念是Afunc添加进队列，Bfunc紧跟着也进入队列，Afunc先执行这个是必然的， 但是Bfunc是等Afunc完全操作完以后，B才开始启动并且执行，因此队列的概念离乱上有点违背了多线程处理这个概念。 但是转念一想其实可以参考银行的取票和叫号系统。 因此对于A比B先排队取票但是B率先执行完操作，我们亦然可以感性认为这还是一个队列。 但是后来看到一票关于这操作队列话题的文章，其中有一句提到 "因为两个操作提交的时间间隔很近，线程池中的线程，谁先启动是不定的。" 瞬间觉得这个queue名字有点忽悠人了，还不如pool~ 综合一点，我们知道他可以比较大的用处在于可以帮组多线程编程就好了。

## Lazy load

最好也最简单的一个列子就是tableView中图片的加载显示了。 一个延时载，避免内存过高，一个异步加载，避免线程堵塞。

## 是否在一个视图控制器中嵌入两个tableview控制器?

一个视图控制只提供了一个View视图，理论上一个tableViewController也不能放吧， 只能说可以嵌入一个tableview视图。当然，题目本身也有歧义，如果不是我们定性思维认为的UIViewController， 而是宏观的表示视图控制者，那我们倒是可以把其看成一个视图控制者，它可以控制多个视图控制器，比如TabbarController那样的感觉。

## 一个tableView是否可以关联两个不同的数据源?你会怎么处理?

首先我们从代码来看，数据源如何关联上的，其实是在数据源关联的代理方法里实现的。 因此我们并不关心如何去关联他，他怎么关联上，方法只是让我返回根据自己的需要去设置如相关的数据源。 因此，我觉得可以设置多个数据源啊，但是有个问题是，你这是想干嘛呢?想让列表如何显示，不同的数据源分区块显示?

## 什么时候使用NSMutableArray，什么时候使用NSArray?

当数组在程序运行时，需要不断变化的，使用NSMutableArray，当数组在初始化后，便不再改变的，使用NSArray。 需要指出的是，使用NSArray只表明的是该数组在运行时不发生改变，即不能往NSAarry的数组里新增和删除元素， 但不表明其数组內的元素的内容不能发生改变。NSArray是线程安全的，NSMutableArray不是线程安全的， 多线程使用到NSMutableArray需要注意。

## 给出委托方法的实例，并且说出UITableVIew的Data Source方法

CocoaTouch框架中用到了大量委托，其中UITableViewDelegate就是委托机制的典型应用， 是一个典型的使用委托来实现适配器模式，其中UITableViewDelegate协议是目标，tableview是适配器， 实现UITableViewDelegate协议，并将自身设置为talbeview的delegate的对象，是被适配器，一般情况下该对象是UITableViewController。

UITableVIew的Data Source方法有- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section;

- (UITableViewCell _)tableView:(UITableView_ )tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;

## 如果我们不创建内存池，是否有内存池提供给我们?

界面线程维护着自己的内存池，用户自己创建的数据线程，则需要创建该线程的内存池

## 什么时候需要在程序中创建内存池?

用户自己创建的数据线程，则需要创建该线程的内存池

## 类NSObject的哪些方法经常被使用?

NSObject是Objetive-C的基类，其由NSObject类及一系列协议构成。 其中类方法alloc、class、 description 对象方法init、dealloc、– performSelector:withObject:afterDelay:等经常被使用

## 什么是简便构造方法?

简便构造方法一般由CocoaTouch框架提供，如NSNumber的 `+ numberWithBool:` `+ numberWithChar:` `+ numberWithDouble:` `+ numberWithFloat:` `+ numberWithInt:` Foundation下大部分类均有简便构造方法，我们可以通过简便构造方法，获得系统给我们创建好的对象，并且不需要手动释放。

## UIView的动画效果有那些?

UIViewAnimationOptionCurveEaseInOut UIViewAnimationOptionCurveEaseIn UIViewAnimationOptionCurveEaseOut UIViewAnimationOptionTransitionFlipFromLeft UIViewAnimationOptionTransitionFlipFromRight UIViewAnimationOptionTransitionCurlUp UIViewAnimationOptionTransitionCurlDown

## 在iPhone应用中如何保存数据?

1. 通过web服务，保存在服务器上
2. 通过NSCoder固化机制，将对象保存在文件中
3. 通过SQlite或CoreData保存在文件数据库中

## ios 平台怎么做数据的持久化?coredata 和sqlite有无必然联系？coredata是一个关系型数据库吗？

iOS 中可以有四种持久化数据的方式：属性列表(plist)、对象归档、 SQLite3 和 Core Data； core data 可以使你以图形界面的方式快速的定义 app 的数据模型，同时在你的代码中容易获取到它。 coredata 提供了基础结构去处理常用的功能，例如保存，恢复，撤销和重做，允许你在 app 中继续创建新的任务。 在使用 core data 的时候，你不用安装额外的数据库系统，因为 core data 使用内置的 sqlite 数据库。 core data 将你 app 的模型层放入到一组定义在内存中的数据对象。 coredata 会追踪这些对象的改变， 同时可以根据需要做相反的改变，例如用户执行撤销命令。当 core data 在对你 app 数据的改变进行保存的时候， core data 会把这些数据归档，并永久性保存。 mac os x 中sqlite 库，它是一个轻量级功能强大的关系数据引擎， 也很容易嵌入到应用程序。可以在多个平台使用， sqlite 是一个轻量级的嵌入式 sql 数据库编程。 与 core data 框架不同的是， sqlite 是使用程序式的， sql 的主要的 API 来直接操作数据表。 Core Data 不是一个关系型数据库，也不是关系型数据库管理系统 (RDBMS) 。虽然 Core Dta 支持SQLite 作为一种存储类型，但它不能使用任意的 SQLite 数据库。 Core Data 在使用的过程种自己创建这个数据库。 Core Data 支持对一、对多的关系。

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

## 简单介绍下NSURLConnection类及+ sendSynchronousRequest:returningResponse:error:

与– initWithRequest:delegate:两个方法的区别? NSURLConnection主要用于网络访问，其中+ sendSynchronousRequest:returningResponse:error: 是同步访问数据，即当前线程会阻塞，并等待request的返回的response，而– initWithRequest:delegate: 使用的是异步加载，当其完成网络访问后，会通过delegate回到主线程，并其委托的对象。

## ViewController的didReceiveMemoryWarning怎么被调用：

`[supper didReceiveMemoryWarning];`

## 用预处理指令#define声明一个常数，用以表明1年中有多少秒（忽略闰年问题）

`#define SECONDS_PER_YEAR (60 * 60 * 24 * 365)UL`

我在这想看到几件事情：

# define 语法的基本知识（例如：不能以分号结束，括号的使用，等等） 懂得预处理器将为你计算常数表达式的值，因此，直接写出你是如何计算一年中有多少秒而不是计算出实际的值， 是更清晰而没有代价的。 意识到这个表达式将使一个16位机的整型数溢出-因此要用到长整型符号L,告诉编译器这个常数是的长整型数。 如果你在你的表达式中用到UL（表示无符号长整型），那么你有了一个好的起点。记住，第一印象很重要。

## 写一个"标准"宏MIN ，这个宏输入两个参数并返回较小的一个。

`#define?MIN(A,B)?（（A）?<=?(B)???(A)?:?(B))` 这个测试是为下面的目的而设的：

标识#define在宏中应用的基本知识。这是很重要的，因为直到嵌入(inline)操作符变为标准C的一部分， 宏是方便产生嵌入代码的唯一方

法，

对于嵌入式系统来说，为了能达到要求的性能，嵌入代码经常是必须的方法。

三重条件操作符的知识。这个操作符存在C语言中的原因是它使得编译器能产生比 if-then-else 更优化的代码， 了解这个用法是很重要的。

懂得在宏中小心地把参数用括号括起来

我也用这个问题开始讨论宏的副作用，例如：当你写下面的代码时会发生什么事？

least?=?MIN(*p++,?b); 结果是：

((_p++)?<=?(b)???(_p++)?:?(*p++)) 这个表达式会产生副作用，指针p会作三次++自增操作。

## 关键字const有什么含意？修饰类呢?

const 意味着"只读"，下面的声明都是什么意思？

```objc
const int a;
int const a;
const int *a;
int * const a;
int const * const a;
```

前两个的作用是一样，a是一个常整型数。 第三个意味着a是一个指向常整型数的指针（也就是，整型数是不可修改的，但指针可以）。 第四个意思a是一个指向整型数的常指针（也就是说，指针指向的整型数是可以修改的，但指针是不可修改的）。 最后一个意味着a是一个指向常整型数的常指针（也就是说，指针指向的整型数是不可修改的，同时指针也是不可修改的）。

> 结论： 关键字const的作用是为给读你代码的人传达非常有用的信息，实际上， 声明一个参数为常量是为了告诉了用户这个参数的应用目的。 合理地使用关键字const可以使编译器很自然地保护那些不希望被改变的参数， 防止其被无意的代码修改。简而言之，这样可以减少bug的出现。

> 1. 欲阻止一个变量被改变,可以使用const关键字.在定义该const变量时,通常需要对它进行初 始化,因为以后就没有机会再去改变它了;
> 2. 对指针来说,可以指定指针本身为const,也可以指定指针所指的数据为const,或二者同时指 定为 const;
> 3. 在一个函数声明中,const可以修饰形参,表明它是一个输入参数,在函数内部不能改变其值;
> 4. 对于类的成员函数,若指定其为const类型,则表明其是一个常函数,不能修改类的成员变量;
> 5. 对于类的成员函数,有时候必须指定其返回值为const类型,以使得其返回值不为"左值".

## static

1. 函数体内 static 变量的作用范围为该函数体，不同于 auto 变量，该变量的内存只被分配一次， 因此其值在下次调用时仍维持上次的值；
2. 在模块内的 static 全局变量可以被模块内所用函数访问，但不能被模块外其它函数访问；
3. 在模块内的 static 函数只可被这一模块内的其它函数调用，这个函数的使用范围被限制在声明它的模块内；
4. 在类中的 static 成员变量属于整个类所拥有，对类的所有对象只有一份拷贝；
5. 在类中的 static 成员函数属于整个类所拥有，这个函数不接收 this 指针，因而只能访问类的static 成员变量。

## extern"C" 的作用：

1. 被extern "C"限定的函数或变量是extern类型的; extern是C/C++语言中表明函数和全局变量作用范围（可见性的关键字,该关键字告诉编译器, 其声明的函数和变量可以在本模块或其它模块中使用。
2. 被extern"C"修饰的变量和函数是按照C语言方式编译和连接的; extern"C"的惯用法:

  1. 在 C++中引用C语言中的函数和变量，在包含C语言头文件假设为 cExample.h时，需进行下列处理：extern "C" { #include "cExample.h" }而在C语言的头文件中，对其外部函数只能指定为extern类型，C语言中不支持extern "C"声明，在.c 文件中包含了extern "C"时会出现编译语法错误。
  2. 在C中引用C++语言中的函数和变量时，C++的头文件需添加extern"C"，但是在C语言中不能直接引用声明了extern "C"的该头文件，应该仅将C文件中将C++中定义的extern"C"函数声明为extern类型。

## 关键字volatile有什么含意?并给出三个不同的例子。

一个定义为volatile的变量是说这变量可能会被意想不到地改变，这样，编译器就不会去假设这个变量的值了。精确地说就是，优化器在用到这个变量时必须每次都小心地重新读取这个变量的值，而不是使用保存在寄存器里的备份。就像大家更熟悉的const一样，volatile是一个类型修饰符。它是被设计用来修饰被不同线程访问和修改的变量。如果不加入volatile，基本上会导致这样的结果：要么无法编写多线程程序，要么编译器失去大量优化的机会。

Volatile变量具有 synchronized 的可见性特性，但是不具备原子特性。这就是说线程能够自动发现 volatile变量的最新值。Volatile变量可用于提供线程安全，但是只能应用于非常有限的一组用例：多个变量之间或者某个变量的当前值与修改后值之间没有约束。因此，单独使用 volatile 还不足以实现计数器、互斥锁或任何具有与多个变量相关的不变式（Invariants）的类（例如 "start <=end"）。

出于简易性或可伸缩性的考虑，您可能倾向于使用 volatile变量而不是锁。当使用 volatile变量而非锁时，某些习惯用法更加易于编码和阅读。此外，volatile变量不会像锁那样造成线程阻塞，因此也很少造成可伸缩性问题。在某些情况下，如果读操作远远大于写操作，volatile变量还可以提供优于锁的性能优势。

```objc
volatile int i=10;
int j = i;
...
int k = i;
```

volatile 告诉编译器i是随时可能发生变化的，每次使用它的时候必须从i的地址中读取，因而编译器生成的可执行码会重新从i的地址读取数据放在k中。编译器在产生release版可执行码时会进行编译优化，加volatile关键字的变量有关的运算，将不进行编译优化。而优化做法是，由于编译器发现两次从i读数据的代码之间的代码没有对i进行过操作，它会自动把上次读的数据放在k中。而不是重新从i里面读。这样以来，如果i是一个寄存器变量或者表示一个端口数据就容易出错，所以说volatile可以保证对特殊地址的稳定访问，不会出错。

```objc
int square(volatile int *ptr) { return *ptr * *ptr; }
```

这段代码的目的是用来返指针ptr指向值的平方，但是，由于ptr指向一个volatile型参数，编译器将产生类似下面的代码：

```objc
int square(volatile int *ptr) {
int a,b;
a = *ptr;
b = *ptr;
return a * b;
}
```

由于*ptr的值可能被意想不到地该变，因此a和b可能是不同的。结果，这段代码可能返不是你所期望的平方值！正确的代码如下：

```objc
long square(volatile int *ptr) { int a; a = *ptr; return a * a; }
```

下面是volatile变量的几个例子：

- 并行设备的硬件寄存器（如：状态寄存器）
- 一个中断服务子程序中会访问到的非自动变量(Non-automatic variables)
- 多线程应用中被几个任务共享的变量

> 1. 一个参数既可以是const还可以是volatile吗？答案是是的。一个例子是只读的状态寄存器。它是volatile因为它可能被意想不到地改变。它是const因为程序不应该试图去修改它。
> 2. 一个指针可以是volatile 吗？答案是是的。尽管这并不很常见。一个例子是当一个中服务子程序修该一个指向一个buffer的指针时。

**在编写多线程程序中使用volatile的关键点：**

1. 将所有的共享对象声明为volatile；
2. 不要将volatile直接作用于基本类型；
3. 当定义了共享类的时候，用volatile成员函数来保证线程安全； 在多线程中，我们可以利用锁的机制来保护好资源临界区。在临界区的外面操作共享变量则需要volatile，在临界区的里面则non-volatile了。

## static 关键字的作用：

1. 函数体内 static 变量的作用范围为该函数体，不同于 auto 变量，该变量的内存只被分配一次， 因此其值在下次调用时仍维持上次的值；
2. 在模块内的 static 全局变量可以被模块内所用函数访问，但不能被模块外其它函数访问；
3. 在模块内的 static 函数只可被这一模块内的其它函数调用，这个函数的使用范围被限制在声明 它的模块内；
4. 在类中的 static 成员变量属于整个类所拥有，对类的所有对象只有一份拷贝；
5. 在类中的 static 成员函数属于整个类所拥有，这个函数不接收 this 指针，因而只能访问类的static 成员变量。

## iOS的系统架构

iOS的系统架构分为（ 核心操作系统层 theCore OS layer ）、（ 核心服务层theCore Services layer ）、 （ 媒体层 theMedia layer ）和（ Cocoa 界面服务层 the Cocoa Touch layer ）四个层次。

## cocoa touch框架

iPhone OS 应用程序的基础 Cocoa Touch 框架重用了许多 Mac 系统的成熟模式， 但是它更多地专注于触摸的接口和优化。

UIKit 为您提供了在 iPhone OS 上实现图形，事件驱动程序的基本工具，其建立在和 Mac OS X 中一样的 Foundation 框架上，包括文件处理，网络，字符串操作等。

Cocoa Touch 具有和 iPhone 用户接口一致的特殊设计。有了 UIKit，您可以使用 iPhone OS 上的独特的图形接口控件，按钮，以及全屏视图的功能，您还可以使用加速仪和多点触摸手势来控制您的应用。

各色俱全的框架 除了UIKit 外，Cocoa Touch 包含了创建世界一流 iPhone 应用程序需要的所有框架， 从三维图形，到专业音效，甚至提供设备访问 API 以控制摄像头，或通过 GPS 获知当前位置。

Cocoa Touch 既包含只需要几行代码就可以完成全部任务的强大的 Objective-C 框架， 也在需要时提供基础的 C 语言 API 来直接访问系统。这些框架包括：

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

当您向一个对象发送一个autorelease消息时，Cocoa就会将该对象的一个引用放入到最新的自动释放. 它仍然是个正当的对象，因此自动释放池定义的作用域内的其它对象可以向它发送消息。 当程序执行到作用域结束的位置时，自动释放池就会被释放，池中的所有对象也就被释放。

## Objective-C的优缺点。

objc优点：

1. Cateogies
2. ?Posing
3. 动态识别
4. 指标计算
5. 弹性讯息传递
6. 不是一个过度复杂的 C 衍生语言
7. Objective-C 与 C++ 可混合编程 objc缺点:
8. 不支援命名空间
9. 不支持运算符重载
10. 不支持多重继承
11. 使用动态运行时类型，所有的方法都是函数调用，所以很多编译时优化方法都用不到。（如内联函数等），性能低劣。

## sprintf,strcpy,memcpy使用上有什么要注意的地方。

1. sprintf是格式化函数。将一段数据通过特定的格式，格式化到一个字符串缓冲区中去。 sprintf格式化的函数的长度不可控，有可能格式化后的字符串会超出缓冲区的大小，造成溢出。
2. strcpy是一个字符串拷贝的函数，它的函数原型为strcpy(char _dst, const char_ src 将src开始的一段字符串拷贝到dst开始的内存中去，结束的标志符号为 '\0'，由于拷贝的长度不是由我们自己控制的， 所以这个字符串拷贝很容易出错。
3. memcpy是具备字符串拷贝功能的函数，这是一个内存拷贝函数，它的函数原型 为memcpy(char _dst, const char_ src, unsigned int len);将长度为len的一段内存， 从src拷贝到dst中去，这个函数的长度可控。但是会有内存叠加的问题。

## 你了解svn,cvs等版本控制工具么？

版本控制 svn,cvs 是两种版控制的器,需要配套相关的svn，cvs服务器。

scm是xcode里配置版本控制的地方。版本控制的原理就是a和b同时开发一个项目，a写完当天的代码之后把代码提交给服务器， b要做的时候先从服务器得到最新版本，就可以接着做。 如果a和b都要提交给服务器，并且同时修改了同一个方法， 就会产生代码冲突，如果a先提交，那么b提交时，服务器可以提示冲突的代码，b可以清晰的看到，并做出相应的修改或融合后再提交到服务器。

## 静态链接库

此为.a文件，相当于java里的jar包，把一些类编译到一个包中，在不同的工程中如果导入此文件就可以使用里面的类， 具体使用依然是#import " xx.h"。

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

ui框架，导入320工程作为框架包如同添加一个普通框架一样。cover(open) ?flower框架 (2d 仿射技术)， 内部核心类是CATransform3D.

## 什么是沙盒模型？哪些操作是属于私有api范畴?

某个iphone工程进行文件操作有此工程对应的指定的位置，不能逾越。

iphone沙箱模型的有四个文件夹documents，tmp，app，Library，永久数据存储一般放documents文件夹， 得到模拟器的路径的可使用NSHomeDirectory()方法。Nsuserdefaults保存的文件在tmp文件夹里。

## 在一个对象的方法里面：self.name= "object"；和 name ="object" 有什么不同吗?

self.name ="object"：会调用对象的setName()方法；

name = "object"：会直接把object赋值给当前对象的name属性。

## 请简要说明viewDidLoad和viewDidUnload何时调用

viewDidLoad在view从nib文件初始化时调用，loadView在controller的view为nil时调用。 此方法在编程实现view时调用，view控制器默认会注册memory warning notification， 当view controller的任何view没有用的时候，viewDidUnload会被调用，在这里实现将retain的view release， 如果是retain的IBOutlet view 属性则不要在这里release，IBOutlet会负责release 。

## 控件主要响应3种事件

- 基于触摸的事件
- 基于值的事件
- 基于编辑的事件。

## xib文件的构成分为哪3个图标？都具有什么功能。

File's Owner 是所有 nib 文件中的每个图标，它表示从磁盘加载 nib 文件的对象；

First Responder 就是用户当前正在与之交互的对象；

View 显示用户界面；完成用户交互；是 UIView 类或其子类。

## 如何高性能的给UIImageView加个圆角？（不准说layer.cornerRadius!）

我觉得应该是使用Quartz2D直接绘制图片,得把这个看看。 步骤：

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

drawRect方法依赖Core Graphics框架来进行自定义的绘制，但这种方法主要的缺点就是它处理touch事件的方式： 每次按钮被点击后，都会用setNeddsDisplay进行强制重绘；而且不止一次，每次单点事件触发两次执行。 这样的话从性能的角度来说，对CPU和内存来说都是欠佳的。特别是如果在我们的界面上有多个这样的UIButton实例。

## 简述视图控件器的生命周期。

loadView 尽管不直接调用该方法，如多手动创建自己的视图，那么应该覆盖这个方法并将它们赋值给试图控制器的 view 属性。

viewDidLoad 只有在视图控制器将其视图载入到内存之后才调用该方法，这是执行任何其他初始化操作的入口。

viewDidUnload 当试图控制器从内存释放自己的方法的时候调用，用于清楚那些可能已经在试图控制器中创建的对象。

viewVillAppear 当试图将要添加到窗口中并且还不可见的时候或者上层视图移出图层后本视图变成顶级视图时调用该方法， 用于执行诸如改变视图方向等的操作。实现该方法时确保调用 [super viewWillAppear:

viewDidAppear 当视图添加到窗口中以后或者上层视图移出图层后本视图变成顶级视图时调用， 用于放置那些需要在视图显示后执行的代码。确保调用 [super viewDidAppear：] 。

## 动画有基本类型有哪几种；表视图有哪几种基本样式。

动画有两种基本类型：隐式动画和显式动画。

## Cocoa Touch提供了哪几种Core Animation过渡类型？

Cocoa Touch 提供了 4 种 Core Animation 过渡类型，分别为：交叉淡化、推挤、显示和覆盖。

## UIView与CLayer有什么区别？

1).UIView是iOS系统中界面元素的基础，所有的界面元素都继承自它。 它本身完全是由CoreAnimation来实现的 （Mac下似乎不是这样）。它真正的绘图部分， 是由一个叫CALayer（Core Animation Layer）的类来管理。 UIView本身，更像是一个CALayer的管理器， 访问它的跟绘图和跟坐标有关的属性，例如frame，bounds等 等，实际上内部都是在访问它所包含的CALayer的相关属性。

2).UIView有个layer属性，可以返回它的主CALayer实例，UIView有一个layerClass方法， 返回主layer所使用的 类，UIView的子类，可以通过重载这个方法，来让UIView使用不同的CALayer来显示，例如通过

```objc
- (class) layerClass {

         return ([CAEAGLLayer class]);
    }
```

使某个UIView的子类使用GL来进行绘制。

3).UIView 的 CALayer 类似 UIView 的子 View 树形结构，也可以向它的 layer 上添加子layer ， 来完成某些特殊的表示。即 CALayer 层是可以嵌套的。

```objc
grayCover = [[CALayer alloc] init];

    grayCover.backgroundColor = [[[UIColor blackColor] colorWithAlphaComponent:0.2] CGColor];

    [self.layer addSubLayer: grayCover];
```

在目标View上敷上一层黑色的透明薄膜。 4).UIView 的 layer 树形在系统内部，被维护着三份 copy 。

- 逻辑树，这里是代码可以操纵的；
- 动画树，是一个中间层，系统就在这一层上更改属性，进行各种渲染操作；
- 显示树，其内容就是当前正被显示在屏幕上得内容。

这三棵树的逻辑结构都是一样的，区别只有各自的属性。

5).动画的运作：对 UIView 的 subLayer （非主 Layer ）属性进行更改，系统将自动进行动画生成， 动画持续时间的缺省值似乎是 0.5 秒。

6).坐标系统： CALayer 的坐标系统比 UIView 多了一个 anchorPoint 属性，使用CGPoint 结构表示， 值域是 0~1 ，是个比例值。这个点是各种图形变换的坐标原点，同时会更改 layer 的 position 的位置， 它的缺省值是 {0.5,0.5} ，即在 layer 的中央。

7).渲染：当更新层，改变不能立即显示在屏幕上。当所有的层都准备好时，可以调用setNeedsDisplay 方法来重绘显示。

8).变换：要在一个层中添加一个 3D 或仿射变换，可以分别设置层的 transform 或affineTransform 属性。

9).变形： Quartz Core 的渲染能力，使二维图像可以被自由操纵，就好像是三维的。 图像可以在一个三维坐标系中以任意角度被旋转，缩放和倾斜。 CATransform3D 的一套方法提供了一些魔术般的变换效果。

## Quatrz 2D的绘图功能的三个核心概念是什么并简述其作用。

上下文：主要用于描述图形写入哪里； 路径：是在图层上绘制的内容； 状态：用于保存配置变换的值、填充和轮廓， alpha 值等。

## 解析XML文件有哪几种方式？

以 DOM 方式解析 XML 文件；以 SAX 方式解析 XML 文件；

## tableView 的重用机制

UITableView 通过重用单元格来达到节省内存的目的: 通过为每个单元格指定一个重用标识符(reuseIdentifier), 即指定了单元格的种类,以及当单元格滚出屏幕时,允许恢复单元格以便重用.对于不同种类的单元格使用不同的ID,对于简单的表格,一个标识符就够了.

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

self 拥有_handler, _handler 拥有block, block拥有self（因为使用了self的_data属性，block会copy 一份self） 解决方法：

```objc
__weak typedof(self)weakSelf = self
    [ downloadData:^(id responseData){
        weakSelf.data = responseData;
    }];
```

## 设计个简单的图片内存缓存器（移除策略是一定要说的）

图片的内存缓存，可以考虑将图片数据保存到一个数据模型中。所以在程序运行时这个模型都存在内存中。 移除策略：释放数据模型对象。

## loadView是干嘛用的？

当你访问一个ViewController的view属性时，如果此时view的值是nil，那么， ViewController就会自动调用loadView这个方法。这个方法就会加载或者创建一个view对象，赋值给view属性。 loadView默认做的事情是：如果此ViewController存在一个对应的nib文件，那么就加载这个nib。 否则，就创建一个UIView对象。

如果你用Interface Builder来创建界面，那么不应该重载这个方法。

如果你想自己创建view对象，那么可以重载这个方法。此时你需要自己给view属性赋值。你自定义的方法不应该调用super。

## 如果你需要对view做一些其他的定制操作，在viewDidLoad里面去做。

根据上面的文档可以知道，有两种情况：

1. 如果你用了nib文件，重载这个方法就没有太大意义。因为loadView的作用就是加载nib。 如果你重载了这个方法不调用super，那么nib文件就不会被加载。如果调用了super，那么view已经加载完了， 你需要做的其他事情在viewDidLoad里面做更合适。

2. 如果你没有用nib，这个方法默认就是创建一个空的view对象。如果你想自己控制view对象的创建， 例如创建一个特殊尺寸的view，那么可以重载这个方法，自己创建一个UIView对象，然后指定 self.view = myView; 但这种情况也没有必要调用super，因为反正你也不需要在super方法里面创建的view对象。如果调用了super， 那么就是浪费了一些资源而已

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

通过上述一个函数就知道横竖屏切换的接口了。 注意：viewWillLayoutSubviews只能用在ViewController里面，在view里面没有响应。

## 你实现过多线程的Core Data么？NSPersistentStoreCoordinator，NSManagedObjectContext

和NSManagedObject中的哪些需要在线程中创建或者传递？你是用什么样的策略来实现的？ <https://onevcat.com/2014/03/common-background-practices/>

## Core开头的系列的内容。是否使用过CoreAnimation和CoreGraphics。UI框架和CA，CG框架的联系是什么？

分别用CA和CG做过些什么动画或者图像上的内容。（有需要的话还可以涉及Quartz的一些内容） <https://onevcat.com/2013/04/using-blending-in-ios/>

## 你实现过一个框架或者库以供别人使用么？如果有，请谈一谈构建框架或者库时候的经验；如果没有，

请设想和设计框架的public的API，并指出大概需要如何做、需要注意一些什么方面，来使别人容易地使用你的框架。

## 深浅复制和属性为copy，strong值的变化问题

浅复制：只复制指向对象的指针，而不复制引用对象本身。对于浅复制来说，A和A_copy指向的是同一个内存资源， 复制的只不个是一个指针，对象本身资源还是只有一份，那如果我们对A_copy执行了修改操作，那么发现A引用的对象同样被修改了。深复制就好理解了，内存中存在了两份独立对象本身。

在Objective-C中并不是所有的对象都支持Copy，MutableCopy，遵守NSCopying协议的类才可以发送Copy消息， 遵守NSMutableCopying协议的类才可以发送MutableCopy消息。

```objc
[immutableObject copy] // 浅拷贝
[immutableObject mutableCopy] //深拷贝
[mutableObject copy] //深拷贝
[mutableObject mutableCopy] //深拷贝
```

属性设为copy,指定此属性的值不可更改，防止可变字符串更改自身的值的时候不会影响到对象属性 （如NSString,NSArray,NSDictionary）的值。strong此属性的指会随着变化而变化。copy是内容拷贝，strong是指针拷贝。

## NSTimer创建后，会在哪个线程运行。

用scheduledTimerWithTimeInterval创建的，在哪个线程创建就会被加入哪个线程的RunLoop中就运行在哪个线程。

自己创建的Timer，加入到哪个线程的RunLoop中就运行在哪个线程。

## KVO，NSNotification，delegate及block区别

KVO就是cocoa框架实现的观察者模式，一般同KVC搭配使用，通过KVO可以监测一个值的变化， 比如View的高度变化。是一对多的关系，一个值的变化会通知所有的观察者。

NSNotification是通知，也是一对多的使用场景。在某些情况下，KVO和NSNotification是一样的， 都是状态变化之后告知对方。NSNotification的特点，就是需要被观察者先主动发出通知， 然后观察者注册监听后再来进行响应，比KVO多了发送通知的一步，但是其优点是监听不局限于属性的变化， 还可以对多种多样的状态变化进行监听，监听范围广，使用也更灵活。

delegate 是代理，就是我不想做的事情交给别人做。比如狗需要吃饭，就通过delegate通知主人， 主人就会给他做饭、盛饭、倒水，这些操作，这些狗都不需要关心，只需要调用delegate（代理人）就可以了， 由其他类完成所需要的操作。所以delegate是一对一关系。

block是delegate的另一种形式，是函数式编程的一种形式。使用场景跟delegate一样，相比delegate更灵活， 而且代理的实现更直观。

KVO一般的使用场景是数据，需求是数据变化，比如股票价格变化，我们一般使用KVO（观察者模式）。 delegate一般的使用场景是行为，需求是需要别人帮我做一件事情，比如买卖股票，我们一般使用delegate。 Notification一般是进行全局通知，比如利好消息一出，通知大家去买入。delegate是强关联， 就是委托和代理双方互相知道，你委托别人买股票你就需要知道经纪人，经纪人也不要知道自己的顾客。 Notification是弱关联，利好消息发出，你不需要知道是谁发的也可以做出相应的反应， 同理发消息的人也不需要知道接收的人也可以正常发出消息。

## 如何让计时器(NSTimer)调用一个类方法

计时器只能调用实例方法，但是可以在这个实例方法里面调用静态方法。

使用计时器需要注意，计时器一定要加入RunLoop中，并且选好model才能运行。scheduledTimerWithTimeInterval 方法创建一个计时器并加入到RunLoop中所以可以直接使用。

如果计时器的repeats选择YES说明这个计时器会重复执行，一定要在合适的时机调用计时器的invalid。 不能在dealloc中调用，因为一旦设置为repeats 为yes，计时器会强持有self，导致dealloc永远不会被调用， 这个类就永远无法被释放。比如可以在viewDidDisappear中调用，这样当类需要被回收的时候就可以正常进入dealloc中了。

## 调用一个类的静态方法需不需要release？

静态方法，就是类方法，不需要，类方法对象放在autorelease中

## NSObject的load和initialize方法

**load和initialize的共同特点** 在不考虑开发者主动使用的情况下，系统最多会调用一次 如果父类和子类都被调用，父类的调用一定在子类之前 都是为了应用运行提前创建合适的运行环境 在使用时都不要过重地依赖于这两个方法，除非真正必要

**load和initialize的区别** **load方法**

调用时机比较早，运行环境有不确定因素。具体说来，在iOS上通常就是App启动时进行加载， 但当load调用的时候，并不能保证所有类都加载完成且可用，必要时还要自己负责做auto release处理。 对于有依赖关系的两个库中，被依赖的类的load会优先调用。但在一个库之内，调用顺序是不确定的。

对于一个类而言，没有load方法实现就不会调用，不会考虑对NSObject的继承。

一个类的load方法不用写明[super load]，父类就会收到调用，并且在子类之前。

Category的load也会收到调用，但顺序上在主类的load调用之后。

不会直接触发initialize的调用。

**initialize方法相关要点**

initialize的自然调用是在第一次主动使用当前类的时候。

在initialize方法收到调用时，运行环境基本健全。

initialize的运行过程中是能保证线程安全的。

和load不同，即使子类不实现initialize方法，会把父类的实现继承过来调用一遍。注意的是在此之前， 父类的方法已经被执行过一次了，同样不需要super调用。

由于initialize的这些特点，使得其应用比load要略微广泛一些。可用来做一些初始化工作， 或者单例模式的一种实现方案。

## 能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？

不能向编译后得到的类中增加实例变量； 能向运行时创建的类中添加实例变量；

因为编译后的类已经注册在 runtime 中，类结构体中的 objc_ivar_list 实例变量的链表 和 instance_size 实例变量的内存大小已经确定，同时runtime 会调用 class_setIvarLayout 或 class_setWeakIvarLayout 来处理 strong weak 引用。所以不能向存在的类中添加实例变量；

运行时创建的类是可以添加实例变量，调用 class_addIvar 函数。但是得在调用 objc_allocateClassPair 之后， objc_registerClassPair 之前，原因同上。

## nil/Nil/NULL/NSNull

1. NULL 声明位置在stddef.h文件 对于普通的iOS开发者来说，通常NULL的定义就是：`# define NULL ((void*)0)` 因此，NULL本质上是：(void_)0 NULL表示C指针为空`charchar_ string = NULL;`
2. nil<br>
  声明在objc.h文件 对于普通iOS开发者来说，nil的定义形式为：`# define nil __DARWIN_NULL` 就是说nil最终是DARWIN_NULL的宏定义，DARWIN_NULL是定义在_types.h中的宏。`#define __DARWIN_NULL ((void *)0)` 也就是说，nil本质上是：(void *)0 用于表示指向Objective-C中对象的指针为空

  ```objc
  NSString *string = nil;  
  id anyObject = nil;
  ```

3. Nil 声明位置在objc.h文件 和上面讲到的nil一样，Nil本质上也是：(void *)0 用于表示Objective-C类（Class）类型的变量值为空

  ```objc
  Class anyClass = Nil;
  ```

4. NSNull 声明位置在NSNull.h文件 定义 ```objc @interface NSNull : NSObject

  <nscopying, nssecurecoding="">
  </nscopying,>

5. (NSNull *)null;<br>
  @end ``` 从定义中可以看出，NSNull是一个Objective-C类，只不过这个类相当特殊，因为它表示的是空值， 即什么都不存。它也只有一个单例方法+[NSUll null]。该类通常用于在集合对象中保存一个空的占位对象。

我们通常初始化NSArray对象的形式如下：

```objc
NSArray *arr = [NSArray arrayWithObjects:@"wang",@"zz",nil];
```

当NSArray里遇到nil时，就说明这个数组对象的元素截止了，即NSArray只关注nil之前的对象， nil之后的对象会被抛弃。比如下面的写法：

```objc
NSArray *arr = [NSArray arrayWithObjects:@"wang",@"zz",nil,@"foogry"];
```

这是NSArray中只会保存wang和zz两个字符串，foogry字符串会被抛弃。 这种情况，就可以使用NSNull实现：

```objc
NSArray *arr = [NSArray arrayWithObjects:@"wang",@"zz",[NSNull null],@"foogry"];
```

从前面的介绍可以看出，不管是NULL、nil还是Nil，它们本质上都是一样的，都是(void *)0，只是写法不同。 这样做的意义是为了区分不同的数据类型，比如你一看到用到了NULL就知道这是个C指针， 看到nil就知道这是个Objective-C对象，看到Nil就知道这是个Class类型的数据。

注意：NULL是C指针指向的值为空；nil是OC对象指针自己本身为空，不是值为空

## 什么是事件响应链，点击屏幕时是如何互动的，事件的传递。

对于IOS设备用户来说，他们操作设备的方式主要有三种：触摸屏幕、晃动设备、通过遥控设施控制设备。 对应的事件类型有以下三种：

1. 触屏事件（Touch Event）
2. 运动事件（Motion Event） 3.远端控制事件（Remote-Control Event）

**响应者链（Responder Chain）** 响应者对象（Responder Object），指的是有响应和处理事件能力的对象。响应者链就是由一系列的响应者对象构成的一个层次结构。

UIResponder是所有响应对象的基类，在UIResponder类中定义了处理上述各种事件的接口。 我们熟悉的UIApplication、 UIViewController、UIWindow和所有继承自UIView的UIKit类 都直接或间接的继承自UIResponder，所以它们的实例都是可以构成响应者链的响应者对象。

响应者链有以下特点： 1、响应者链通常是由视图（UIView）构成的； 2、一个视图的下一个响应者是它视图控制器（UIViewController）（如果有的话），然后再转给它的父视图（Super View）； 3、视图控制器（如果有的话）的下一个响应者为其管理的视图的父视图； 4、单例的窗口（UIWindow）的内容视图将指向窗口本身作为它的下一个响应者 需要指出的是，Cocoa Touch应用不像Cocoa应用，它只有一个UIWindow对象，因此整个响应者链要简单一点； 5、单例的应用（UIApplication）是一个响应者链的终点，它的下一个响应者指向nil，以结束整个循环。

**点击屏幕时是如何互动的** iOS系统检测到手指触摸(Touch)操作时会将其打包成一个UIEvent对象，并放入当前活动Application的事件队列， 单例的UIApplication会从事件队列中取出触摸事件并传递给单例的UIWindow来处理，UIWindow对象首 先会使用hitTest:withEvent:方法寻找此次Touch操作初始点所在的视图(View)， 即需要将触摸事件传递给其处理的视图，这个过程称之为hit-test view。

UIWindow实例对象会首先在它的内容视图上调用hitTest:withEvent:，此方法会在其视图层级结构中的 每个视图上调用pointInside:withEvent:（该方法用来判断点击事件发生的位置是否处于当前视图范围内， 以确定用户是不是点击了当前视图），如果pointInside:withEvent:返回YES，则继续逐级调用， 直到找到touch操作发生的位置，这个视图也就是要找的hit-test view。

hitTest:withEvent:方法的处理流程如下:首先调用当前视图的pointInside:withEvent:方法判断 触摸点是否在当前视图内；若返回NO,则hitTest:withEvent:返回nil;若返回YES,则向当前视图的 所有子视图(subviews)发送hitTest:withEvent:消息，所有子视图的遍历顺序是从最顶层视图一直到到最底层视图， 即从subviews数组的末尾向前遍历，直到有子视图返回非空对象或者全部子视图遍历完毕； 若第一次有子视图返回非空对象，则hitTest:withEvent:方法返回此对象，处理结束；如所有子视图都返回非， 则hitTest:withEvent:方法返回自身(self)。

事件的传递和响应分两个链：

传递链：由系统向离用户最近的view传递。UIKit –> active app's event queue –> window –> root view –>......–>lowest view 响应链：由离用户最近的view向系统传递。initial view –> super view –> .....–> view controller –> window –> Application

## ARC和MRC

Objective-c中提供了两种内存管理机制MRC（MannulReference Counting）和ARC(Automatic Reference Counting)， 分别提供对内存的手动和自动管理，来满足不同的需求。Xcode 4.1及其以前版本没有ARC。

在MRC的内存管理模式下，与对变量的管理相关的方法有：retain,release和autorelease。 retain和release方法操作的是引用记数，当引用记数为零时，便自动释放内存。 并且可以用NSAutoreleasePool对象，对加入自动释放池（autorelease调用）的变量进行管理，当drain时回收内存。

1. retain，该方法的作用是将内存数据的所有权附给另一指针变量，引用数加1，即retainCount+= 1;
2. release，该方法是释放指针变量对内存数据的所有权，引用数减1，即retainCount-= 1;
3. autorelease，该方法是将该对象内存的管理放到autoreleasepool中。

在ARC中与内存管理有关的标识符，可以分为变量标识符和属性标识符，对于变量默认为__strong， 而对于属性默认为unsafe_unretained。也存在autoreleasepool。

其中assign/retain/copy与MRC下property的标识符意义相同，strong类似与retain,assign类似于unsafe_unretained，strong/weak/unsafe_unretained与ARC下变量标识符意义相同，只是一个用于属性的标识， 一个用于变量的标识(带两个下划短线__)。所列出的其他的标识符与MRC下意义相同。

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

**应用程序的状态** **Not running未运行**：程序没启动。 **Inactive未激活**：程序在前台运行，不过没有接收到事件。在没有事件处理情况下程序通常停留在这个状态。 **Active激活**：程序在前台运行而且接收到了事件。这也是前台的一个正常的模式。 **Backgroud后台**：程序在后台而且能执行代码，大多数程序进入这个状态后会在在这个状态上停留一会。 时间到之后会进入挂起状态(Suspended)。有的程序经过特殊的请求后可以长期处于Backgroud状态。 **Suspended挂起**：程序在后台不能执行代码。系统会自动把程序变成这个状态而且不会发出通知。 当挂起时，程序还是停留在内存中的，当系统内存低时，系统就把挂起的程序清除掉，为前台程序提供更多的内存。

iOS的入口在main.m文件：

```objc
int main(int argc, char *argv[])
{
@autoreleasepool {
    return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
}
}
```

main函数的两个参数，iOS中没有用到，包括这两个参数是为了与标准ANSI C保持一致。 UIApplicationMain函数， 前两个和main函数一样，重点是后两个。

后两个参数分别表示程序的主要类(principal class)和代理类(delegate class)。 如果主要类(principal class)为nil，将从Info.plist中获取，如果Info.plist中不存在对应的key， 则默认为UIApplication；如果代理类(delegate class)将在新建工程时创建。 根据UIApplicationMain函数，程序将进入AppDelegate.m，这个文件是xcode新建工程时自动生成的。 下面看一下AppDelegate.m文件，这个关乎着应用程序的生命周期。

1. application didFinishLaunchingWithOptions：当应用程序启动时执行，应用程序启动入口， 只在应用程序启动时执行一次。若用户直接启动，lauchOptions内无数据,若通过其他方式启动应用， lauchOptions包含对应方式的内容。
2. applicationWillResignActive：在应用程序将要由活动状态切换到非活动状态时候， 要执行的委托调用，如 按下 home 按钮，返回主屏幕，或全屏之间切换应用程序等。
3. applicationDidEnterBackground：在应用程序已进入后台程序时，要执行的委托调用。
4. applicationWillEnterForeground：在应用程序将要进入前台时(被激活)，要执行的委托调用， 刚好与applicationWillResignActive 方法相对应。
5. applicationDidBecomeActive：在应用程序已被激活后，要执行的委托调用， 刚好与applicationDidEnterBackground 方法相对应。
6. applicationWillTerminate：在应用程序要完全推出的时候，要执行的委托调用， 这个需要要设置UIApplicationExitsOnSuspend的键值。

**初次启动**： iOS_didFinishLaunchingWithOptions iOS_applicationDidBecomeActive **按下home键**： iOS_applicationWillResignActive iOS_applicationDidEnterBackground **点击程序图标进入**： iOS_applicationWillEnterForeground iOS_applicationDidBecomeActive

当应用程序进入后台时,应该保存用户数据或状态信息，所有没写到磁盘的文件或信息，在进入后台时， 最后都写到磁盘去，因为程序可能在后台被杀死。释放尽可能释放的内存。

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

**程序终止** 程序只要符合以下情况之一，只要进入后台或挂起状态就会终止：

1. OS4.0以前的系统
2. app是基于iOS4.0之前系统开发的。
3. 设备不支持多任务
4. 在Info.plist文件中，程序包含了 UIApplicationExitsOnSuspend 键。

系统常常是为其他app启动时由于内存不足而回收内存最后需要终止应用程序， 但有时也会是由于app很长时间才响应而终止。如果app当时运行在后台并且没有暂停， 系统会在应用程序终止之前调用app的代理的方法 - (void)applicationWillTerminate:(UIApplication *)application， 这样可以让你可以做一些清理工作。你可以保存一些数据或app的状态。这个方法也有5秒钟的限制。 超时后方法会返回程序从内存中清除。

注意：用户可以手工关闭应用程序。

## 远程推送

当服务端远程向APNS推送至一台离线的设备时，苹果服务器Qos组件会自动保留一份最新的通知， 等设备上线后，Qos将把推送发送到目标设备上

远程推送的基本过程

1. 客户端的app需要将用户的UDID和app的bundleID发送给apns服务器,进行注册, apns将加密后的device Token返回给app
2. app获得device Token后,上传到公司服务器
3. 当需要推送通知时,公司服务器会将推送内容和device Token一起发给apns服务器
4. apns再将推送内容送到客户端上

创建证书的流程：

1. 打开钥匙串，生成CertificateSigningRequest.certSigningRequest文件
2. 将CertificateSigningRequest.certSigningRequest上传进developer，导出.cer文件
3. 利用CSR导出P12文件
4. 需要准备下设备token值（无空格）
5. 使用OpenSSL合成服务器所使用的推送证书

本地app代码参考 1.注册远程通知

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions//中注册远程通知
{
[[UIApplication sharedApplication] registerForRemoteNotificationTypes:(UIRemoteNotificationTypeAlert | UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeSound)];
}
```

1. 实现几个代理方法：

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

  一般我们是使用开发版本的Provisioning做推送测试,如果没有问题,再使用发布版本证书的时候一般也应该是没有问题的。 为了以防万一,我们可以在越狱的手机上安装我们的使用发布版证书的ipa文件(最好使用debug版本, 并打印出获取到的deviceToken),安装成功后在;XCode->Window->Organizer-找到对应的设 备查看console找到打印的deviceToken。

在后台的推送程序中使用发布版制作的证书并使用该deviceToken做推送服务. 使用开发和发布证书获取到的deviceToken是不一样的。

## iOS的锁屏和解锁

**idleTimer** idleTimer 是iOS内置的时间监测机制，当在一段时间内未操作即进入锁屏状态。但有些应用程序是不需要锁住屏幕的， 比如游戏，视频这类应用。 可以通过设置UIApplication的idleTimerDisabled属性来指定iOS是否锁屏。

```objc
// 禁用休闲时钟
[[UIApplication sharedApplication] setIdleTimerDisabled: YES];
```

也可以使用这种语法

```objc
[UIApplication sharedApplication].idleTimerDisabled = YES;
```

但是，这个命令只能禁用自动锁屏，如果点击了锁屏按钮，仍然会进入锁屏的。有一点例外的是， AVPlayer不用设置idleTimerDisabled=YES，也能屏幕常亮，播放完成后过一分钟就自动关闭屏幕。 有兴趣的可以自己尝试一下。

**锁屏和解锁通知** iPhone的锁屏监测分为两种方式监听：一种是程序在前台，另一种程序在后台。 程序在前台，这种比较简单。 直接使用Darwin层的通知就可以了：

> Darwin是由苹果电脑于2000年所释出的一个开放原始码操作系统。Darwin 是MacOSX 操作环境的操作系统成份。 苹果电脑于2000年把Darwin 释出给开放原始码社群。现在的Darwin皆可以在苹果电脑的PowerPC 架构和X86 架构下执行， 而后者的架构只有有限的驱动程序支援。

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

第二种是程序退后台后，这时再锁屏就收不到上面的那个通知了，需要另外一种方式, 以循环的方式一直来检测是否是锁屏状态， 会消耗性能并可能被苹果挂起，需要合理设置循环时间。

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

**UIApplication** 上面我们使用了UIApplication的IdleTimerDisabled方法，下面就大概了解下UIApplication吧。

UIApplication，每个程序只能有一个，系统使用的是单例模式，用[UIApplication sharedApplication]来得到一个实例。 这个单例实例是在系统启动时由main函数里面的UIApplicationMain方法生成，实现的是UIApplicationDelegate的Protocol， 也就是AppDelegate的一个实例。每次通过[UIApplication sharedApplication]调用的就是它。 UIApplication保存一个UIWindow对象序列，用来快速恢复views。

UIApplication在程序里的作用很多，大致如下所示：

```objc
一、远程提醒，就是push notification注册；
二、可以连接到UIUndoManager；在Cocoa中使用NSUndoManager可以很方便的完成撤销操作。
NSUndoManager会记录下修改、撤销操作的消息。这个机制使用两个NSInvocation对象栈。当进行操作时，
控制器会添加一个该操作的逆操作的invocation到Undo栈中。当进行Undo操作时，
Undo操作的逆操作会倍添加到Redo栈中，就这样利用Undo和Redo两个堆栈巧妙的实现撤销操作。需要注意的是，
堆栈中存放的都是NSInvocation实例。
三、检查能否打开某个URL，并且打开URL；这个功能可以配合应用的自定义URL功能，来检测是否安装了某个应用。
使用的是[[UIApplication sharedApplication] canOpenURL:url]方法。如果返回YES，
可执行[[UIApplication sharedApplication] openURL:url];
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

prepareWithInvocationTarget:方法记录了target并返回UndoManager， 然后UndoManager重载了forwardInvocation方法，也就将two方法的Invocation添加到undo栈中了。

## 什么是OC

OC语言在c语言的基础上，增加了一层最小的面向对象语法，完全兼容C语言，在OC代码中，可以混用c， 甚至是c++代码。可以使用OC开发mac osx平台和iOS平台的应用程序。拓展名：c语言.c OC语言.m 兼容C++.mm。

为了与c语言的关键字区分开来,基本上所有的关键字都是以@开头。基本类型：5种，增加了布尔类型， BOOL类型与其他类型的用法一致，BOOL类型的本质是char类型的，定义如下：

```objc
 Typedef signed char BOOL
```

宏定义:

```objc
#define YES  (BOOL)1
#define NO   (BOOL)0
```

布尔类型的输出一般当做整数来用。 程序编译连接过程为：源文件（.m）---(编译)---->目标文件（.0）-----（链接）---->可执行文件（.out）。

每个对象内部都默认有一个isa指针指向这个对象所使用的类。isa是对象中的隐藏指针，指向创建这个对象的类。 OC做为一门面向对象语言，具有面向对象的语言特性，如封装、继承、多态。也具有静态语言的特性(如C++)， 又有动态语言的效率(动态绑定、动态加载等)。

Apple公司领导着Objective-C语言的发展与维护，包括Objective-C运行时， Cocoa/Cocoa-Touch框架以及Objective-C语言的编译器。

## 什么是面向对象

OC语言是面向对象的，c语言是面向过程的，面向对象和面向过程只是解决问题的两种思考方式， 面向过程关注的是解决问题涉及的步骤，面向对象关注的是设计能够实现解决问题所需功能的类。

面向对象的编程方法具有四个基本特征：抽象，封装，继承和多态。

抽象是忽略一个主题中与当前目标无关的那些方面，以便更充分地注意与当前目标有关的方面。 抽象包括两个方面，一是过程抽象，二是数据抽象。过程抽象是指任何一个明确定义功能的操作都可被使用者看作单个的实体看待， 尽管这个操作实际上可能由一系列更低级的操作来完成。数据抽象定义了数据类型和施加于该类型对象上的操作， 并限定了对象的值只能通过使用这些操作修改和观察。

继承是一种联结类的层次模型，并且允许和鼓励类的重用，它提供了一种明确表述共性的方法。 新类继承了原始类的特性，新类称为原始类的派生类（子类），而原始类称为新类的基类（父类）。 派生类可以从它的基类那里继承方法和实例变量，并且类可以修改或增加新的方法使之更适合特殊的需要。 继承性很好的解决了软件的可重用性问题。

封装是把过程和数据包围起来，对数据的访问只能通过已定义的界面。面向对象计算始于这个基本概念， 即现实世界可以被描绘成一系列完全自治、封装的对象，这些对象通过一个受保护的接口访问其他对象。 一旦定义了一个对象的特性，则有必要决定这些特性的可见性，封装保证了模块具有较好的独立性， 使得程序维护修改较为容易。对应用程序的修改仅限于类的内部，因而可以将应用程序修改带来的影响减少到最低限度。

多态性是指允许不同类的对象对同一消息作出响应。多态性包括参数化多态性和包含多态性。 多态性语言具有灵活、抽象、行为共享、代码共享的优势，很好的解决了应用程序函数同名问题。**多态是依赖于接口的**。

但是，在C++使用OOP的编程方式在一些场合未能提供最高性能。现在内存存取成为计算机性能的重要瓶颈， 这个问题在C++设计OOP编程范式的实现方式之初并未能考虑周全。现时的OOP编程有可能不缓存友好（cache friendly）， 导致有时候并不能发挥硬件最佳性能。大概就是过度封装，多态增加cache miss的可能性，数据存取时导致载入缓存的浪费等。

## OC和传统的面向对象语言有什么区别和优势

OC中方法的实现只能写在@implementation··@end中，对象方法的声明只能写在@interface···@end中间； 对象方法都以-号开头，类方法都以+号开头；对象方法只能由对象来调用，类方法只能由类来调用， 不能当做函数一样调用。函数属于整个文件，可以写在文件中的任何位置，包括@implementation··@end中， 但写在@interface···@end会无法识别，函数的声明可以在main函数内部也可以在main函数外部。 对象方法归类\对象所有；函数调用不依赖于对象；函数内部不能直接通过成员变量名访问对象的成员变量。

Objective-C的运行时是动态的，它能让你在运行时为类添加方法或者去除方法以及使用反射。 OC的动态特性表现为了三个方面：动态类型、动态绑定、动态加载。之所以叫做动态， 是因为必须到运行时(run time)才会做一些事情。

动态类型，说简单点就是id类型。动态类型是跟静态类型相对的。像内置的明确的基本类型都属于静态类型(int、NSString等)。 静态类型在编译的时候就能被识别出来。所以，若程序发生了类型不对应，编译器就会发出警告。 而动态类型就编译器编译的时候是不能被识别的，要等到运行时(run time)，即程序运行的时候才会根据语境来识别。 所以这里面就有两个概念要分清：编译时跟运行时。

动态绑定(dynamic binding)需要用到@selector/SEL。先来看看"函数"，对于其他一些静态语言， 比如c++,一般在编译的时候就已经将要调用的函数的函数签名都告诉编译器了。静态的，不能改变。而在OC中， 其实是没有函数的概念的，我们叫"消息机制"，所谓的函数调用就是给对象发送一条消息。这时， 动态绑定的特性就来了。OC可以先跳过编译，到运行的时候才动态地添加函数调用，在运行时才决定要调用什么方法， 需要传什么参数进去。这就是动态绑定，要实现他就必须用SEL变量绑定一个方法。 最终形成的这个SEL变量就代表一个方法的引用。这里要注意一点：**SEL并不是C里面的函数指针**，虽然很像， 但真心不是函数指针。SEL变量只是一个整数，他是该方法的ID。以前的函数调用，是根据函数名， 也就是字符串去查找函数体。但现在，我们是根据一个ID整数来查找方法，整数的查找字自然要比字符串的查找快得多！ 所以，动态绑定的特定不仅方便，而且效率更高。

动态加载就是根据需求动态地加载资源，在运行时加载新类。在运行时创建一个新类,只需要3步:

1. 为 class pair分配存储空间 ,使用 objc_allocateClassPair函数
2. 增加需要的方法使用class_addMethod函数,增加实例变量用class_addIvar
3. 用objc_registerClassPair函数注册这个类,以便它能被别人使用。

> 使用这些函数请引#import

> <objc runtime.h="">
> </objc>

## UIWindow

**UIWindow** UIWindow继承自UIView，UIWindow是一种特殊的UIView，通常在一个程序中只会有一个UIWindow， 但可以手动创建多个UIWindow，同时加到程序里面。即使有多个UIWindow对象， 也只有一个UIWindow可以接受到用户的触屏事件（即主窗口）。

iOS程序启动完毕后，先创建Application，再创建它的代理，之后创建UIWindow（创建的第一个对象是UIApplication， 接着创建控制器的view，最后将控制器的view添加到UIWindow上，于是控制器的view就显示在屏幕上了。

一个iOS程序之所以能显示到屏幕上，完全是因为它有UIWindow。也就说，没有UIWindow，就看不见任何UI界面。 **主窗口和次窗口**

```objc
[self.window makekeyandvisible]; 让窗口成为主窗口，并且显示出来。有这个方法，才能把信息显示到屏幕上。
```

因为Window有makekeyandvisible这个方法，可以让这个Window凭空的显示出来，而其他的view没有这个方法， 所以它只能依赖于Window，Window显示出来后，view才依附在Window上显示出来。

```objc
[self.window make keywindow];//让uiwindow成为主窗口，但不显示。
```

次窗口，需要定义一个Window属性来保存变量。 window的属性定义为strong，就是为了让其不销毁， 一个应用程序只能有一个主窗口。只有主窗口才能响应键盘的输入事件，如果不能输入内容， 可以查看是否是显示在主窗口上，不在主窗口上的不能响应。 **WindowLevel** UIWindow有三个层级，分别是Normal，StatusBar，Alert。Normal级别是最低的，StatusBar处于中等水平， Alert级别最高。而通常我们的程序的界面都是处于Normal这个级别上的，系统顶部的状态栏应该是处于StatusBar级别 ，UIActionSheet和UIAlertView这些通常都是用来中断正常流程，提醒用户等操作，因此位于Alert级别。

根据window显示级别优先的原则，级别高的会显示在上面，级别低的在下面，我们程序正常显示的view位于最底层。

当Level层级相同的时候，只有第一个设置为KeyWindow的显示出来，后面同级的再设置KeyWindow也不会显示。 UIWindow在显示的时候是不管KeyWindow是谁，都是Level优先的，即Level最高的始终显示在最前面。

**UIWindow是显示的起点**

1. UIWindow对象是所有UIView的根，管理和协调应用程序的显示。
2. UIViewController对象负责管理所有UIView的层次结构，并响应设备的方向变化。
3. UIView对象定义了一个屏幕上的一个矩形区域，同时处理该区域的绘制和触屏事件。 可以在这个区域内绘制图形和文字，还可以接收用户的操作。一个UIView的实例可以包含和管理若干个子UIView。

**UIWindow在程序中的作用**

1. 作为容器，包含app所要显示的所有视图
2. 传递触摸消息到程序中view和其他对象
3. 与UIViewController协同工作，方便完成设备方向旋转的支持

**storyboard项目中的创建过程** 当用户点击应用程序图标的时候，先执行Main函数，执行UIApplicationMain(), 根据其第三个和第四个参数创建Application，创建代理，并且把代理设置给application （看项目配置文件info.plist里面的storyboard的name，根据这个name找到对应的storyboard）， 开启一个事件循环，当程序加载完毕，他会调用代理的didFinishLaunchingWithOptions:方法。 在调用didFinishLaunchingWithOptions:方法之前，会加载storyboard，在加载的时候创建一个window， 接下来会创建箭头所指向的控制器，把该控制器设置为UIWindow的根控制器，接下来再将window显示出来， 即看到了运行后显示的界面。

**rootViewController和addSubview的不同** 创建一个控制器，把view添加到uiwindow上面有两种方式

1. rootViewController rootViewController是UIWindow的一个遍历方法，通过设置该属性为要添加view对应的ViewController， UIWindow将会自动将其view添加到当前window中，同时负责ViewController和view的生命周期的维护， 防止其过早释放

2. addSubview 直接将view通过addSubview方式添加到window中，程序负责维护view的生命周期以及刷新， 但是并不会为去理会view对应的ViewController，因此采用这种方法将view添加到window以后， 我们还要保持view对应的ViewController的有效性，不能过早释放。

提示：不通过控制器的view也可以做开发，但是在实际开发中，不要这么做，不要直接把view添加到UIWindow上面去。 因为，难以管理。以后的开发中，建议使用rootViewController。

**UIView有关图层布局的方法** 一个 UIView 里面可以包含许多的 Subview（其他的 UIView），而这些 Subview 彼此之间是有层级关系的。

1. 新增Subview

  ```objc
  [UIView addSubview:Subview];     //替UIView增加一个Subview
  ```

2. 移动图层 在UIView中将Subview往前或是往后移动一个图层，往前移动会覆盖住较后层的 Subview，而往后移动则会被较上层的Subview所覆盖。

  ```objc
  UIView bringSubviewToFront:Subview];       //将Subview往前移动一个图层（与它的前一个图层对调位置）
  [UIView sendSubviewToBack:Subview];      //将Subview往后移动一个图层（与它的后一个图层对调位置）
  ```

3. 交换两个Subview彼此的图层层级

  ```objc
  [UIView exchangeSubviewAtIndex:indexA withSubviewAtIndex:indexB];    //交换两个图层
  ```

4. 取得子视图在UIView中的索引值（Index）

  ```objc
  NSInteger index = [[UIView subviews] indexOfObject:Subview名称];       //取得Index
  ```

## **如何用一行代码计算NSString字符的个数**

正确答案是`[str lengthOfBytesUsingEncoding:NSUTF32StringEncoding]/4` 有少部分朋友答对了。 当然还有其他方式，但绝不是`str.length`。 length返回的是以`utf16`为单位的code unit个数。 像很多emoji表情都会占2个unit,实际却是一个字符。不了解的朋友需要补充下Unicode相关知识。

## 滑动的时候隐藏navigation bar

```objc
navigationController.hidesBarsOnSwipe = Yes;
```

## 消除导航条返回键带的title

```objc
[[UIBarButtonItem appearance] setBackButtonTitlePositionAdjustment:UIOffsetMake(0, -60)
                                                 forBarMetrics:UIBarMetricsDefault];
```

## 将Navigationbar变成透明而不模糊

```objc
[self.navigationController.navigationBar setBackgroundImage:[UIImage new]
                         forBarMetrics:UIBarMetricsDefault];
self.navigationController.navigationBar .shadowImage = [UIImage new];
self.navigationController.navigationBar .translucent = YES;
```

## 用一个pan手势来代替UISwipegesture的各个方向

```objc
- (void)pan:(UIPanGestureRecognizer *)sender
{
typedef NS_ENUM(NSUInteger, UIPanGestureRecognizerDirection) {
UIPanGestureRecognizerDirectionUndefined,
UIPanGestureRecognizerDirectionUp,
UIPanGestureRecognizerDirectionDown,
UIPanGestureRecognizerDirectionLeft,
UIPanGestureRecognizerDirectionRight
};

static  UIPanGestureRecognizerDirection direction = UIPanGestureRecognizerDirectionUndefined;

switch (sender.state) {
case  UIGestureRecognizerStateBegan: {
if  (direction == UIPanGestureRecognizerDirectionUndefined) {
CGPoint velocity = [sender velocityInView:recognizer.view];
BOOL isVerticalGesture = fabs(velocity.y) > fabs(velocity.x);
if (isVerticalGesture) {
    if (velocity.y > 0) {
    direction = UIPanGestureRecognizerDirectionDown;
    }    
    else  {
    direction = UIPanGestureRecognizerDirectionUp;
    }
}
else
{
if (velocity.x > 0) {
direction = UIPanGestureRecognizerDirectionRight;
}
else
{
direction = UIPanGestureRecognizerDirectionLeft;
}
}
}
break ;
}

case UIGestureRecognizerStateChanged: {
switch (direction) {
case UIPanGestureRecognizerDirectionUp: {
[self handleUpwardsGesture:sender];
break ;
}

case UIPanGestureRecognizerDirectionDown: {
[self handleDownwardsGesture:sender];
break;
}

case  UIPanGestureRecognizerDirectionLeft: {
[self handleLeftGesture:sender];
break;
}

case UIPanGestureRecognizerDirectionRight: {
[self handleRightGesture:sender];
break ;
}
default : {
break;
}
}
break;
}

case  UIGestureRecognizerStateEnded: {
direction = UIPanGestureRecognizerDirectionUndefined;  
break;
}

default:
break;
}
}
```

## 拉伸图片不变形

```objc
[[UIImage imageNamed:@""] stretchableImageWithLeftCapWidth:10 topCapHeight:10];
[[UIImage imageNamed:@""] resizableImageWithCapInsets:UIEdgeInsetsMake(10, 10, 10, 10)];
```

## Gif图片显示优化

[FLAnimatedImage](https://github.com/Flipboard/FLAnimatedImage)

```objc
FLAnimatedImage *image = [FLAnimatedImage animatedImageWithGIFData:[NSData dataWithContentsOfURL:[NSURL URLWithString:@"https://upload.wikimedia.org/wikipedia/commons/2/2c/Rotating_earth_%28large%29.gif"]]];
FLAnimatedImageView *imageView = [[FLAnimatedImageView alloc] init];
imageView.animatedImage = image;
imageView.frame = CGRectMake(0.0, 0.0, 100.0, 100.0);
[self.view addSubview:imageView];
```

## CollectionView实现tableview的悬停header

[CSStickyHeaderFlowLayout](https://github.com/jamztang/CSStickyHeaderFlowLayout)

```objc
#import "CSStickyHeaderFlowLayout.h"
- (void)viewDidLoad {
 [super viewDidLoad]; // Locate your layout     
CSStickyHeaderFlowLayout *layout = (id)self.collectionViewLayout;
if ([layout isKindOfClass:[CSStickyHeaderFlowLayout class]]) {
layout.parallaxHeaderReferenceSize = CGSizeMake(320, 200);
 }
}


- (UICollectionReusableView *)collectionView:(UICollectionView *)collectionView viewForSupplementaryElementOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath { // Check the kind if it's CSStickyHeaderParallaxHeader
if ([kind isEqualToString:CSStickyHeaderParallaxHeader]) {
UICollectionReusableView *cell = [collectionView dequeueReusableSupplementaryViewOfKind:kind withReuseIdentifier:@"header" forIndexPath:indexPath];
 return cell;
}
}
```

## iOS中日志打印Q&A

**打印当前的函数和行号？**

```objc
NSLog(@"%s:%d obj=%@", __func__, __LINE__, obj);
```

其中func和LINE都是预编译的宏，编译时会分别替换为当前函数和当前行号。 下面是一些常用于打印日志的宏。

宏                   | 说明
------------------- | ----------------------------
**func**            | 打印当前函数或方法，c字符串
**LINE**            | 打印当前行号，整数
**FILE**            | 打印当前文件路径，c字符串
**PRETTY_FUNCTION** | 打印当前函数或方法（在C++中会包含参数类型），c字符串

**打印一个类名，消息名，当前堆栈信息？** 使用以下方法在运行时动态获取这些信息。 代码 | 说明 --- | --- `NSStringFromSelector(SEL)` | 获取selector的名字 `NSStringFromSelector(_cmd)` | 获取当前方法名 `NSStringFromClass([object class])` | 获取object的类名 `[NSThread callStackSymbols]` | 获取当前线程的栈，是一个NSArry，包含堆栈中所有函数名。

**将日志打印到一个文件?** 使用freopen函数重定向标准输出和标准出错文件。因为printf函数会向标准输出（stdout）打印，而NSLog函数会向标准出错（stderr）打印。重新定向标准输出（stdout）和标准出错（stderr）到一个文件将会使他们打印日志到一个文件中。

```objc
freopen(“/tmp/log.txt”, “a+”, stdout);
freopen(“/tmp/log.txt”, “a+”, stderr);
```

## 图片旋转时抗锯齿

> 将某一个视图进行旋转，旋转时会发现图片边缘出现了很多锯齿。即使把layer的edgeAntialiasingMask属性设置了依然会有锯齿。如何才能消除锯齿呢？如果你仔细，你会发现那些边缘虚化(透明)的图片在旋转时并不会出现锯齿。那么如果我们把这些图片的边缘透明化，会不会解决这个问题呢？

在图片边缘加了一个像素的透明区域，代码如下：

```objc
- (UIImage*)antialiasedImageOfSize:(CGSize)size scale:(CGFloat)scale{
    UIGraphicsBeginImageContextWithOptions(size, NO, scale);
    [self drawInRect:CGRectMake(1, 1, size.width-2, size.height-2)];
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return image;
}
```

> Layer的edgeAntialiasingMask属性并不能有效抗锯齿，只需要在图片边缘加入1个像素的透明区域就可以完美实现图片抗锯齿了。

## 使用UIAppearance来自定义应用的外观

**自定义导航条背景**

```objc
[[UINavigationBar appearance] setBackgroundImage:[UIImage imageNamed:@"background"] forB
```

**自定义导航标题文字属性**

```objc
[[UINavigationBar appearance] setTitleTextAttributes:@{UITextAttributeTextColor:[UIColor darkGrayColor],UITextAttributeTextShadowColor:[UIColor clearColor]}];
```

**自定义导航条返回和左右按钮按钮背景**

```objc
[[UIBarButtonItem appearanceWhenContainedIn:[UINavigationBar class], nil] setBackButtonBackgroundImage:[UIImage imageNamed:@"back_button_background"] forState:UIControlStateNormal barMetrics:UIBarMetricsDefault];
[[UIBarButtonItem appearanceWhenContainedIn:[UINavigationBar class], nil] setBackgroundImage:[UIImage imageNamed:@"button_background"] forState:UIControlStateNormal barMetrics:UIBarMetricsDefault];
```

**自定义底部Tab条的背景**

```objc
[[UITabBar appearance] setBackgroundImage:[UIImage imageNamed:@"background"]];
```

**自定义底部条标题文字属性**

```objc
[[UITabBarItem appearance] setTitleTextAttributes:@{UITextAttributeTextColor:[UIColor grayColor]} forState:UIControlStateNormal];
[[UITabBarItem appearance] setTitleTextAttributes:@{UITextAttributeTextColor:[UIColor orangeColor]} forState:UIControlStateSelected];
```

> 只要是头文件中有"UI_APPEARANCE_SELECTOR"标记的方法都是可以用UIAppearance协议对象去调的。 注意这些自定义方法都要在相应的对象显示之前调用，可以放到App启动后立刻配置， 以后只要这个对象显示之前，就会设置相应的属性。

**创建一个可自定义外观的控件** 对于我们自己定义的控件，也可以支持UIAppearance协议，这样我们的控件也能支持自定义了。你只需要写一个设置外观的settor，然后在settor方法后面加上"UI_APPEARANCE_SELECTOR"标记就可以，其他什么都不需要做。比如一个可以自定义选择状态背景颜色的TableViewCell。

```objc
@interface CustomCell : UITableViewCell
- (void)setSelectedBackgroundColor:(UIColor*)color UI_APPEARANCE_SELECTOR;
@end

@implementation CustomCell
- (id)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier
{
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        self.selectedBackgroundView = [UIView new];
        self.selectedBackgroundView.backgroundColor = [UIColor lightGrayColor];
    }
    return self;
}
- (void)setSelectedBackgroundColor:(UIColor*)color{
    self.selectedBackgroundView.backgroundColor = color;
}
@end
```

注意，官方文档中强调Appearance的setter定义格式应为：

```objc
- (void)setProperty:(PropertyType)property forAxis1:(IntegerType)axis1 axis2:(IntegerType)axis2 axisN:(IntegerType)axisN;
- (PropertyType)propertyForAxis1:(IntegerType)axis1 axis2:(IntegerType)axis2 axisN:(IntegerType)axisN;
```

**UIAppearance实现原理** 在通过UIAppearance调用"UI_APPEARANCE_SELECTOR"标记的方法来配置外观时，UIAppearance实际上没有进行任何实际调用，而是把这个调用保存起来（在Objc中可以用NSInvocation对象来保存一个调用）。当实际的对象显示之前（添加到窗口上，drawRect:之前），就会对这个对象调用之前保存的调用。当这个setter调用后，你的界面风格自定义就完成了。

# **编译器的实现流程**

# **GCC和LLVM的区别**

# **OC基础**

- Notification在多线程时会有什么问题？怎么解决？有问题，发送和接收需要在同一个线程中， 如果不在需要定义一个通知队列，当post来时看看是否为期望线程，不是的话就将其放入队列， 然后发送signal到期望线程，待收到signal就从队列移除。
- 举几个会引起block循环引用的例子。
- SEL和IMP 的区别？
- 图片缓存机制，如果一个cell对应图片下载很慢，这时对cell删除操作应该怎么处理。
- MVVM是为了解决什么样的问题
- Core Data处理大量数据同步操作
- class的载入过程
- delegate和block是为了解决什么问题设计的，什么时侯用block什么时侯用delegate
- # define定义变量和const定义有什么区别

- 如何看待React Native

- ReactiveCocoa是为了解决什么设计的，什么时侯用

- 自己设计应用网络层时会考虑哪些问题？

- 持久层，使用sqlite如何设计版本迁移方案

# **内部实现原理**

- block的底层实现原理？
- 通知中心的实现原理？
- Category为什么可以添加方法，不可以添加实例变量？
- iOS的堆内存是怎么管理的？
- @property是如何生成一个成员变量和其setter，getter方法的？
- runloop内部是如何实现的
- autoreleasepool是如何实现的

# **实例实现**

- 设计一个可离线评论，有网再将数据传到服务器的API和客户端实现方案。
- 如何做一个View能够出现在应用所有页面的最上面。
- 设计一个排队系统可以让每个在队中的人看到自己队列所处位置和变化，队伍可能随时有人加入和退出， 当有人退出影响到用户位置排名时需要及时通知反馈到用户。
