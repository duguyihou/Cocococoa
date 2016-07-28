# objc

## Objective-C对象消息名关键词

我们在写Objective－C的代码时，在进行某个动作（action）时，会发送一些相关联的消息。经常会遇到以下的一些关键词：

- should 决定某个动作是否要发生，如果返回NO，则不会执行这个动作，也不会有will和did消息（下面将会说明）了。如shouldAutorotateToInterfaceOrientation:
- will 通常在某个动作发生之前，如viewWillAppear、viewWillDisappear等
- did 通常在某个动作发生之后，如viewDidAppear，didAddSubView等

> 所以它们的调用顺序依次是：should->will->action->did 写代码时一定要搞清这个顺序，否则很容易出现逻辑错误。


## 能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？

不能向编译后得到的类中增加实例变量； 能向运行时创建的类中添加实例变量；

因为编译后的类已经注册在 runtime 中，类结构体中的 objc_ivar_list 实例变量的链表 和 instance_size 实例变量的内存大小已经确定，同时runtime 会调用 class_setIvarLayout 或 class_setWeakIvarLayout 来处理 strong weak 引用。所以不能向存在的类中添加实例变量；

运行时创建的类是可以添加实例变量，调用 class_addIvar 函数。但是得在调用 objc_allocateClassPair 之后， objc_registerClassPair 之前，原因同上。


## loadView是干嘛用的？

当你访问一个ViewController的view属性时，如果此时view的值是nil，那么， ViewController就会自动调用loadView这个方法。这个方法就会加载或者创建一个view对象，赋值给view属性。 loadView默认做的事情是：如果此ViewController存在一个对应的nib文件，那么就加载这个nib。 否则，就创建一个UIView对象。

如果你用Interface Builder来创建界面，那么不应该重载这个方法。

如果你想自己创建view对象，那么可以重载这个方法。此时你需要自己给view属性赋值。你自定义的方法不应该调用super。

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
Object-c的类不可以多重继承;可以实现多个接口，通过实现多个接口可以完成C++的多重继承;Category是类别， 一般情况用分类好，用Category去重写类的方法，仅对本Category有效，不会影响到其他类与原有类的关系。多继承在这里是用protocol 委托代理 来实现的

## #import 跟#include 又什么区别，@class呢, #import<> 跟 #import""又什么区别

# import是Objective-C导入头文件的关键字，#include是C/C++导入头文件的关键字, 使用#import头文件会自动只导入一次，不会重复导入，相当于#include和#pragma once; @class告诉编译器某个类的声明，当执行时，才去查看类的实现文件，可以解决头文件的相互包含;

# import<>用来包含系统的头文件，#import""用来包含用户头文件。


## 常见的object-c的数据类型有那些， 和C的基本数据类型有什么区别?如：NSInteger和int

object-c的数据类型有NSString，NSNumber，NSArray，NSMutableArray，NSData等等， 这些都是class，创建后便是对象，而C语言的基本数据类型int，只是一定字节的内存空间， 用于存放数值;NSInteger是基本数据类型，并不是NSNumber的子类，当然也不是NSObject的子类。 NSInteger是基本数据类型Int或者Long的别名(NSInteger的定义typedef long NSInteger)， 它的区别在于，NSInteger会根据系统是32位还是64位来决定是本身是int还是Long。

## id 声明的对象有什么特性?

Id 声明的对象具有运行时的特性，即可以指向任意类型的objcetive-c的对象;

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

### oc中的协议和java中的接口概念有何不同?

OC中的代理有2层含义，官方定义为 formal和informal protocol。前者和Java接口一样。 informal protocol中的方法属于设计模式考虑范畴，不是必须实现的，但是如果有实现，就会改变类的属性。 其实关于正式协议，类别和非正式协议我很早前学习的时候大致看过，也写在了学习教程里 "非正式协议概念其实就是类别的另一种表达方式"这里有一些你可能希望实现的方法，你可以使用他们更好的完成工作"。 这个意思是，这些是可选的。比如我门要一个更好的方法，我们就会申明一个这样的类别去实现。然后你在后期可以直接使用这些更好的方法。 这么看，总觉得类别这玩意儿有点像协议的可选协议。" 现在来看，其实protocal已经开始对两者都统一和规范起来操作，因为资料中说"非正式协议使用interface修饰"， 现在我们看到协议中两个修饰词："必须实现(@requied)"和"可选实现(@optional)"。


## 代理(Delegate)的作用?

> 代理的目的是改变或传递控制链。

- 允许一个类在某些特定时刻通知到其他类，而不需要获取到那些类的指针。可以减少框架复杂度。
- 代理可以理解为java中的回调监听机制的一种类似。

## 通知和协议的不同之处?

协议有控制链(has-a)的关系，通知没有。 delegate针对one-to-one关系，用于sender接受到reciever的某个功能反馈值。

notification针对one-to-one/many/none,reciver,用于通知多个object某个事件。

## 什么是推送(push)消息?

推送通知更是一种技术。 简单点就是客户端获取资源的一种手段。 普通情况下，都是客户端主动的pull。 推送则是服务器端主动push。


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

## 类NSObject的哪些方法经常被使用?

NSObject是Objetive-C的基类，其由NSObject类及一系列协议构成。 其中类方法alloc、class、 description 对象方法init、dealloc、– performSelector:withObject:afterDelay:等经常被使用

## 什么是简便构造方法?

简便构造方法一般由CocoaTouch框架提供，如NSNumber的 `+ numberWithBool:` `+ numberWithChar:` `+ numberWithDouble:` `+ numberWithFloat:` `+ numberWithInt:` Foundation下大部分类均有简便构造方法，我们可以通过简便构造方法，获得系统给我们创建好的对象，并且不需要手动释放。

## NSPredicate

通过给定的逻辑条件作为约束条件，完成对数据的筛选。

```objc
predicate = [NSPredicate predicateWithFormat:@"customerID == %d",n];
a = [customers filteredArrayUsingPredicate:predicate];
```

## ViewController的didReceiveMemoryWarning怎么被调用：

`[supper didReceiveMemoryWarning];`

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

## 320框架

ui框架，导入320工程作为框架包如同添加一个普通框架一样。cover(open) ?flower框架 (2d 仿射技术)， 内部核心类是CATransform3D.


## 在一个对象的方法里面：self.name= "object"；和 name ="object" 有什么不同吗?

self.name ="object"：会调用对象的setName()方法；

name = "object"：会直接把object赋值给当前对象的name属性。

## xib文件的构成分为哪3个图标？都具有什么功能。

File's Owner 是所有 nib 文件中的每个图标，它表示从磁盘加载 nib 文件的对象；

First Responder 就是用户当前正在与之交互的对象；

View 显示用户界面；完成用户交互；是 UIView 类或其子类。


## **tableView 的重用机制**

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

## **如何用一行代码计算NSString字符的个数**

正确答案是`[str lengthOfBytesUsingEncoding:NSUTF32StringEncoding]/4` 有少部分朋友答对了。 当然还有其他方式，但绝不是`str.length`。 length返回的是以`utf16`为单位的code unit个数。 像很多emoji表情都会占2个unit,实际却是一个字符.需要补充下Unicode相关知识。

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
