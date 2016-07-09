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

## 对block的理解

Block分为三种，分别是全局block、栈block和堆block。ARC之后，我们并不需要手动copy到堆上，通常都已经交给编译器来完成。

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

## 对MVC和MVVM的理解
MVC是出现比较早的架构设计模式，而且到现在已经是很成熟了。出现MVVM的原因是MVC中的V越来越复杂，于是才有人想要给V瘦身。

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

如果列表比较复杂，对于上面的做好后，还是不够流畅，就需要通过Instruments工具来检测哪些地方可以优化了。

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
