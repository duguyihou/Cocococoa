## 对ARC的理解

ARC是编译器帮我们完成的，我们不再手动添加retain、relase、autorelease， 而且在运行期还会帮助我们优化。但是ARC并不是万能的，它并不能自我理解循环引用问题， 依然需要我们手动解决循环引用的问题。

ARC管理都会放到自动释放池中，如果我们需要做一些循环操作，生成大量的临时变量， 我们还是需要加一下autoreleasepool，以及时地释放内存。

ARC下对于属性修饰符不同，其内存管理策略也不一样：

- strong：强引用，引用计数加1
- weak：弱引用，引用计数没有加1
- copy：强引用，引用计数加1

ARC下还是有可能出现内存泄露的，内存得不到释放，特别是使用block的时候，一定要学会分析是否形成循环引用。

### 内存管理的几条原则是什么?按照默认法则.那些关键字生成的对象需要手动释放?在和property结合的时候怎样有效的避免内存泄露?

谁申请，谁释放 遵循Cocoa Touch的使用原则; 内存管理主要要避免"过早释放"和"内存泄漏"，对于"过早释放"需要注意@property设置特性时， 一定要用对特性关键字，对于"内存泄漏"，一定要申请了要负责释放，要细心。 关键字alloc 或new 生成的对象需要手动释放; 设置正确的property属性，对于retain需要在合适的地方释放，

### ARC和MRC

Objective-c中提供了两种内存管理机制MRC（MannulReference Counting）和ARC(Automatic Reference Counting)， 分别提供对内存的手动和自动管理，来满足不同的需求。Xcode 4.1及其以前版本没有ARC。

在MRC的内存管理模式下，与对变量的管理相关的方法有：retain,release和autorelease。 retain和release方法操作的是引用记数，当引用记数为零时，便自动释放内存。 并且可以用NSAutoreleasePool对象，对加入自动释放池（autorelease调用）的变量进行管理，当drain时回收内存。

1. retain，该方法的作用是将内存数据的所有权附给另一指针变量，引用数加1，即retainCount+= 1;
2. release，该方法是释放指针变量对内存数据的所有权，引用数减1，即retainCount-= 1;
3. autorelease，该方法是将该对象内存的管理放到autoreleasepool中。

在ARC中与内存管理有关的标识符，可以分为变量标识符和属性标识符，对于变量默认为__strong， 而对于属性默认为unsafe_unretained。也存在autoreleasepool。

其中assign/retain/copy与MRC下property的标识符意义相同，strong类似与retain,assign类似于unsafe_unretained，strong/weak/unsafe_unretained与ARC下变量标识符意义相同，只是一个用于属性的标识， 一个用于变量的标识(带两个下划短线__)。所列出的其他的标识符与MRC下意义相同。

### Objective-C如何对内存管理的,说说你的看法和解决方法?

Objective-C的内存管理主要有三种方式ARC(自动内存计数)、手动内存计数、内存池。

1. (Garbage Collection)自动内存计数：这种方式和java类似，在你的程序的执行过程中。 始终有一个高人在背后准确地帮你收拾垃圾，你不用考虑它什么时候开始工作，怎样工作。你只需要明白， 我申请了一段内存空间，当我不再使用从而这段内存成为垃圾的时候，我就彻底的把它忘记掉， 反正那个高人会帮我收拾垃圾。遗憾的是，那个高人需要消耗一定的资源，在携带设备里面， 资源是紧俏商品所以iPhone不支持这个功能。所以"Garbage Collection"不是本入门指南的范围， 对"Garbage Collection"内部机制感兴趣的同学可以参考一些其他的资料， 不过说老实话"Garbage Collection"不大适合适初学者研究。

解决: 通过alloc – initial方式创建的, 创建后引用计数+1, 此后每retain一次引用计数+1, 那么在程序中做相应次数的release就好了.

1. (Reference Counted)手动内存计数：就是说，从一段内存被申请之后， 就存在一个变量用于保存这段内存被使用的次数，我们暂时把它称为计数器，当计数器变为0的时候， 那么就是释放这段内存的时候。比如说，当在程序A里面一段内存被成功申请完成之后， 那么这个计数器就从0变成1(我们把这个过程叫做alloc)，然后程序B也需要使用这个内存， 那么计数器就从1变成了2(我们把这个过程叫做retain)。紧接着程序A不再需要这段内存了， 那么程序A就把这个计数器减1(我们把这个过程叫做release);程序B也不再需要这段内存的时候， 那么也把计数器减1(这个过程还是release)。当系统(也就是Foundation)发现这个计数器变 成员了0， 那么就会调用内存回收程序把这段内存回收(我们把这个过程叫做dealloc)。 顺便提一句，如果没有Foundation，那么维护计数器，释放内存等等工作需要你手工来完成。

解决:一般是由类的静态方法创建的, 函数名中不会出现alloc或init字样, 如[NSString string]和[NSArray arrayWithObject:], 创建后引用计数+0, 在函数出栈后释放, 即相当于一个栈上的局部变量. 当然也可以通过retain延长对象的生存期.

1. (NSAutoRealeasePool)内存池：可以通过创建和释放内存池控制内存申请和回收的时机.

解决:是由autorelease加入系统内存池, 内存池是可以嵌套的, 每个内存池都需要有一个创建释放对, 就像main函数中写的一样. 使用也很简单, 比如[[[NSString alloc]initialWithFormat:@"Hey you!"] autorelease], 即将一个NSString对象加入到最内层的系统内存池, 当我们释放这个内存池时, 其中的对象都会被释放.

### 如果我们不创建内存池，是否有内存池提供给我们?

界面线程维护着自己的内存池，用户自己创建的数据线程，则需要创建该线程的内存池

### 什么时候需要在程序中创建内存池?

用户自己创建的数据线程，则需要创建该线程的内存池
### 自动释放池(autoreleasepool)是什么,如何工作

当您向一个对象发送一个autorelease消息时，Cocoa就会将该对象的一个引用放入到最新的自动释放. 它仍然是个正当的对象，因此自动释放池定义的作用域内的其它对象可以向它发送消息。 当程序执行到作用域结束的位置时，自动释放池就会被释放，池中的所有对象也就被释放。

### OC的垃圾回收机制?

OC2.0有Garbage collection，但是iOS平台不提供。 一般我们了解的objective-c对于内存管理都是手动操作的，但是也有自动释放池。 但是差了大部分资料，貌似不要和arc机制搞混就好了。

### __weak

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

### 野指针是什么，iOS开发中什么情况下会有野指针？

所谓野指针，是指指向内存已经被释放的内存区的指针。

当进入播放页面时马上又返回上一个页面，偶尔出现闪退，原因就是出现了野指针（访问了已释放的对象内存区。 当进入播放页面时，就会立刻去解析视频数据，内部是FFMPEG操作，当快速返回上一个页面时， FFMPEG还在操作中，导致访问了已释放的对象。 使用block时，也会出现野指针。

## 调用一个类的静态方法需不需要release？

静态方法，就是类方法，不需要，类方法对象放在autorelease中
