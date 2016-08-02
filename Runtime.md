# Runtime
> 1. 动态的添加对象的成员变量和方法
2. 动态交换两个方法的实现
3. 实现分类也可以添加属性
4. 实现NSCoding的自动归档和解档
5. 实现字典转模型的自动转换

在编译时你写的 Objective-C 函数调用的语法都会被翻译成一个 C 的函数调用 - objc\_msgSend() 。比如，下面两行代码就是等价的：
	[array insertObject:foo atIndex:5];

	objc_msgSend(array, @selector(insertObject:atIndex:), foo, 5);
消息传递的关键藏于 objc\_object 中的 isa 指针和 objc\_class 中的 class dispatch table。
## `objc_object`, `objc_class` 以及 `Ojbc_method`
### SEL
 先看一下SEL的概念，Objective-C在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的整型标识(Int类型的地址)，这个标识就是SEL。
     SEL也是@selector的类型，用来表示OC运行时的方法的名字。来看一下OC中的定义
	typedef id (*IMP)(id, SEL, ...);
     本质上，SEL只是一个指向方法的指针（准确的说，只是一个根据方法名hash化了的KEY值，能唯一代表一个方法），它的存在只是为了加快方法的查询速度。这个查找过程我们将在下面说明。
      我们可以在运行时添加新的selector，也可以在运行时获取已存在的selector。
### IMP
实际上是一个函数指针，指向方法实现的首地址，定义如下：
	typedef id (*IMP)(id, SEL, ...);
关于IMP的几点说明：
- 使用当前CPU架构实现的标准的C调用约定
- 第一个参数是指向self的指针（如果是实例方法，则是类实例的内存地址；如果是类方法，则是指向元类的指针）
- 第二个参数是方法选择器(selector)，
- 从第三个参数开始是方法的实际参数列表。
    通过取得IMP，我们可以跳过Runtime的消息传递机制，直接执行IMP指向的函数实现，这样省去了Runtime消息传递过程中所做的一系列查找操作，会比直接向对象发送消息高效一些，当然必须说明的是，这种方式只适用于极特殊的优化场景，如效率敏感的场景下大量循环的调用某方法。
```
### Method
	typedef struct objc_method *Method;
	struct objc_method {
			SEL method_name
			char *method_types
			IMP method_imp
	}
```
Method = SEL + IMP + method\_types, 相当于在SEL和IMP之间建立了一个映射。_
在 Objective-C 中，类、对象和方法都是一个 C 的结构体，从 objc/objc.h 头文件中，我们可以找到他们的定义：
```
	struct objc_object {
	    Class isa  OBJC_ISA_AVAILABILITY;
	};

	struct objc_class {
	    Class isa  OBJC_ISA_AVAILABILITY;
	#if !__OBJC2__
	    Class super_class;
	    const char *name;
	    long version;
	    long info;
	    long instance_size;
	    struct objc_ivar_list *ivars;
	    **struct objc_method_list **methodLists**;
	    **struct objc_cache *cache**;
	    struct objc_protocol_list *protocols;
	#endif
	};
	struct objc_method_list {
	    struct objc_method_list *obsolete;
	    int method_count;

	#ifdef __LP64__
	    int space;
	#endif

	    /* variable length structure */
	    struct objc_method method_list[1];
	};

	struct objc_method {
	    SEL method_name;
	    char *method_types;    /* a string representing argument/return types */
	  IMP method_imp;
	};
```
`objc_method_list`本质是一个有`objc_method`元素的可变长度的数组。一个 `objc_method`结构体中有函数名，也就是SEL，有表示函数类型的字符串 (见 Type Encoding) ，以及函数的实现IMP。

从这些定义中可以看出发送一条消息也就 `objc_msgSend`做了什么事。举`objc_msgSend(obj, foo)`这个例子来说：
1. 首先，通过 obj 的 isa 指针找到它的 class ;
2. 在 class 的 method list 找 foo ;
3. 如果 class 中没到 foo，继续往它的 superclass 中找 ;
4. 一旦找到 foo 这个函数，就去执行它的实现IMP .
但这种实现有个问题，效率低。但一个 class 往往只有 20% 的函数会被经常调用，可能占总调用次数的 80% 。每个消息都需要遍历一次 `objc_method_list` 并不合理。如果把经常被调用的函数缓存下来，那可以大大提高函数查询的效率。这也就是 `objc_class `中另一个重要成员 `objc_cache `做的事情 - 再找到 foo 之后，把 foo 的 `method_name` 作为 key ，`method_imp` 作为 value 给存起来。当再次收到 foo 消息的时候，可以直接在 cache 里找到，避免去遍历 `objc_method_list`.
## 动态方法解析和转发
在上面的例子中，如果 `foo` 没有找到会发生什么？通常情况下，程序会在运行时挂掉并抛出 unrecognized selector sent to … 的异常。但在异常抛出前，Objective-C 的运行时会给你三次拯救程序的机会：
1. Method resolution
2. Fast forwarding
3. Normal forwarding
### Method Resolution
首先，Objective-C 运行时会调用 `+resolveInstanceMethod: `或者 `+resolveClassMethod:`，让你有机会提供一个函数实现。如果你添加了函数并返回 YES， 那运行时系统就会重新启动一次消息发送的过程。还是以 foo 为例，你可以这么实现：
```
	void fooMethod(id obj, SEL _cmd)
	{
	    NSLog(@"Doing foo");
	}

	+ (BOOL)resolveInstanceMethod:(SEL)aSEL
	{
	    if(aSEL == @selector(foo:)){
	        class_addMethod([self class], aSEL, (IMP)fooMethod, "v@:");
	        return YES;
	    }
	    return [super resolveInstanceMethod];
	}
```
Core Data 有用到这个方法。NSManagedObjects 中 properties 的 getter 和 setter 就是在运行时动态添加的。
如果 resolve 方法返回 NO ，运行时就会移到下一步：**消息转发**（Message Forwarding）。

PS：iOS 4.3 加入很多新的 runtime 方法，主要都是以 imp 为前缀的方法，比如 `imp_implementationWithBlock()` 用 block 快速创建一个 imp 。
上面的例子可以重写成：
```
	IMP fooIMP = imp_implementationWithBlock(^(id _self) {
	    NSLog(@"Doing foo");
	});

	class_addMethod([self class], aSEL, fooIMP, "v@:");
```
### Fast forwarding
如果目标对象实现了`-forwardingTargetForSelector:`，Runtime 这时就会调用这个方法，给你把这个消息转发给其他对象的机会。
```
	- (id)forwardingTargetForSelector:(SEL)aSelector
	{
	    if(aSelector == @selector(foo:)){
	        return alternateObject;
	    }
	    return [super forwardingTargetForSelector:aSelector];
	}
```
只要这个方法返回的不是 nil 和 self，整个消息发送的过程就会被重启，当然发送的对象会变成你返回的那个对象。否则，就会继续 Normal Fowarding 。

这里叫 Fast ，只是为了区别下一步的转发机制。因为这一步不会创建任何新的对象，但下一步转发会创建一个 NSInvocation 对象，所以相对更快点。
### Normal forwarding
这一步是 Runtime 最后一次给你挽救的机会。首先它会发送 `-methodSignatureForSelector:`消息获得函数的参数和返回值类型。如果`-methodSignatureForSelector:`返回 nil ，Runtime 则会发出`-doesNotRecognizeSelector:`消息，程序这时也就挂掉了。如果返回了一个函数签名，Runtime 就会创建一个 NSInvocation 对象并发送`-forwardInvocation:`消息给目标对象。

NSInvocation 实际上就是对一个消息的描述，包括selector 以及参数等信息。所以你可以在`-forwardInvocation:`里修改传进来的 NSInvocation 对象，然后发送`-invokeWithTarget:`消息给它，传进去一个新的目标：
```
	- (void)forwardInvocation:(NSInvocation *)invocation
	{
	    SEL sel = invocation.selector;

	    if([alternateObject respondsToSelector:sel]) {
	        [invocation invokeWithTarget:alternateObject];
	    }
	    else {
	        [self doesNotRecognizeSelector:sel];
	    }
	}
```
Cocoa 里很多地方都利用到了消息传递机制来对语言进行扩展，如 Proxies、NSUndoManager 跟 Responder Chain。NSProxy 就是专门用来作为代理转发消息的；NSUndoManager 截取一个消息之后再发送；而 Responder Chain 保证一个消息转发给合适的响应者。
## 总结
Objective-C 中给一个对象发送消息会经过以下几个步骤：
1. 在对象类的 dispatch table 中尝试找到该消息。如果找到了，跳到相应的函数IMP去执行实现代码；
2. 如果没有找到，Runtime 会发送`+resolveInstanceMethod:`或者 `+resolveClassMethod:`尝试去 resolve 这个消息；
3. 如果 resolve 方法返回 NO，Runtime 就发送` -forwardingTargetForSelector:`允许你把这个消息转发给另一个对象；
4. 如果没有新的目标对象返回， Runtime 就会发送`-methodSignatureForSelector:`和`-forwardInvocation:`消息。你可以发送`-invokeWithTarget:`消息来手动转发消息或者发送`-doesNotRecognizeSelector:`抛出异常。

# [Associative机制使用场景](http://blog.sina.com.cn/s/blog_60342e330101tcz1.html)

## 概念
objective-c有两个扩展机制：category和associative。我们可以通过category来扩展方法，但是它有个很大的局限性，不能扩展属性。于是，就有了专门用来扩展属性的机制：associative。
## 使用方法
在iOS开发过程中，category比较常见，而associative就用的比较少。associative的主要原理，就是把两个对象相互关联起来，使得其中的一个对象作为另外一个对象的一部分。

使用associative，我们可以不用修改类的定义而为其对象增加存储空间。这在我们无法访问到类的源码的时候或者是考虑到二进制兼容性的时候是非常有用。

associative是基于关键字的。因此，我们可以为任何对象增加任意多的associative，每个都使用不同的关键字即可。associative是可以保证被关联的对象在关联对象的整个生命周期都是可用的。

associative机制提供了三个方法：
```
OBJC_EXPORT void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
```
```
OBJC_EXPORT id objc_getAssociatedObject(id object, const void *key)
```
```
OBJC_EXPORT void objc_removeAssociatedObjects(id object)
```
### 创建associative
创建associative使用的是：objc_setAssociatedObject。它把一个对象与另外一个对象进行关联。该函数需要四个参数：源对象，关键字，关联的对象、关联策略。

关键字是一个void类型的指针。每一个关联的关键字必须是唯一的。通常都是会采用静态变量来作为关键字。

关联策略表明了相关的对象是通过赋值，保留引用还是复制的方式进行关联的；还有这种关联是原子的还是非原子的。这里的关联策略和声明属性时的很类似。这种关联策略是通过使用预先定义好的常量来表示的。

比如，我们想对一个UIView，添加一个NSString类型的tag。可以这么做：
```
- (void)setTagString:(NSString *)value {
         objc_setAssociatedObject(self, KEY_TAGSTRING, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```
### 获取associative对象
获取相关联的是函数objc_getAssociatedObject。

继续上面的例子，从一个UIView的实例中，获取一个NSString类型的tag
```
- (NSString *)tagString {

         NSObject *obj = objc_getAssociatedObject(self, KEY_TAGSTRING);

         if (obj && [obj isKindOfClass:[NSString class]]) {

              return (NSString *)obj;

    }



         return nil;

}
```
### 断开associative
断开associative是使用objc_setAssociatedObject函数，传入nil值即可。
```
objc_setAssociatedObject(self, KEY_TAGSTRING, nil, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```
使用函数objc_removeAssociatedObjects可以断开所有associative。通常情况下不建议这么做，因为他会断开所有关联。
## 应用场景
### TagString
上面的例子提到，在UIView中添加NSString类型的标记，就是一个非常实用的例子。全部的代码如下：
```
@interface UIView(BDTag)
@property (nonatomic, retain) NSString *tagString;
- (UIView *)viewWithTagString:(NSString *)value;
@end
```
```
# import "UIView+BDTag.h"
# undef   KEY_TAGSTRING
# define KEY_TAGSTRING     "UIView.tagString"
@implementation UIView(BDTag)
@dynamic tagString;
- (NSString *)tagString {
         NSObject *obj = objc_getAssociatedObject(self, KEY_TAGSTRING);
         if (obj && [obj isKindOfClass:[NSString class]]) {
              return (NSString *)obj;
    }
         return nil;
}
- (void)setTagString:(NSString *)value {
         objc_setAssociatedObject(self, KEY_TAGSTRING, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
- (UIView *)viewWithTagString:(NSString *)value {
        if (nil == value) {
              return nil;
    }
         for (UIView *subview in self.subviews) {
              NSString *tag = subview.tagString;
              if ([tag isEqualToString:value])
              {
                     return subview;
              }
         }
         return nil;
}
@end
```
苹果虽然有提供NSInteger类型的tag属性，用于标记相应的ui。但是在处理比较复杂的逻辑的时候，往往NSInteger类型的标记不能满足需求。为其添加了NSString类型的标记后。就能使用字符串，快速的标记ui，并且使用viewWithTagString方法，快速找到你所需要的ui。
### 为NSObject子类添加任何信息
这是一个方便，强大，并且简单的类。利用associative机制，为任何Object，添加你所需要的信息。比如用户登录，向服务端发送用户名/密码时，可以将这些信息绑定在请求的项之中。等请求完成后，再取出你所需要的信息，进行逻辑处理。而不需要另外设置成员，保存这些数据。

具体的实现如下：
```
@interface NSObject (BDAssociation)
- (id)associatedObjectForKey:(NSString*)key;
- (void)setAssociatedObject:(id)object forKey:(NSString*)key;
@end
```
```
# import
# import "NSObject+BDAssociation.h"
@implementation NSObject (BDAssociation)
static char associatedObjectsKey;
- (id)associatedObjectForKey:(NSString*)key {
    NSMutableDictionary *dict = objc_getAssociatedObject(self, &associatedObjectsKey);
    return [dict objectForKey:key];
}
- (void)setAssociatedObject:(id)object forKey:(NSString*)key {
    NSMutableDictionary *dict = objc_getAssociatedObject(self, &associatedObjectsKey);
    if (!dict) {
        dict = [[NSMutableDictionary alloc] init];
        objc_setAssociatedObject(self, &associatedObjectsKey, dict, OBJC_ASSOCIATION_RETAIN);
    }
    [dict setObject:object forKey:key];
}
@end

```
### 内存回收检测
记得在我刚开始学iOS开发的时候，经常出现内存泄露的问题，于是就在各个view controller的dealloc中打Log。这种方法虽然有效，但比较挫，不好管理。

这里贴出一种漂亮的解决方案，利用associative机制。让object在回收时，自动输出回收信息。
```
@interface NSObject (BDLogDealloc)
- (void)logOnDealloc;
@end
# import "NSObject+BDLogDealloc.h"
static char __logDeallocAssociatedKey__;
@interface LogDealloc : NSObject
@property (nonatomic, copy) NSString* message;
@end
@implementation NSObject (LogDealloc)
- (void)logOnDealloc {
    if(objc_getAssociatedObject(self, &__logDeallocAssociatedKey__) == nil) {
        LogDealloc* log = [[[LogDealloc alloc] init] autorelease];
        log.message = NSStringFromClass(self.class);
        objc_setAssociatedObject(self, &__logDeallocAssociatedKey__, log, OBJC_ASSOCIATION_RETAIN);
    }
}
@end
@implementation LogDealloc
- (void)dealloc {
    NSLog(@"dealloc: %@", self.message);
    [_message release];
    [super dealloc];
}
@end
```

# Runtime Method
## 定义
```
typedef struct objc_method *Method;
typedef struct objc_selector *SEL;
typedef void (*IMP)(void /* id, SEL, ... */ );
//方法描述
struct objc_method_description {
	SEL name;               //方法名称
	char *types;            //参数类型字符串
};
//以下代码是 ObjC2.0 之前method的定义
struct objc_method {
    SEL method_name;
    char *method_types;
    IMP method_imp;
}
```
* `SEL`selector的简写,俗称方法选择器,实质存储的是方法的名称
* `IMP`implement的简写,俗称方法实现,看源码得知它就是一个函数指针
* `Method`对上述两者的一个包装结构.

## 函数
```
//判断类中是否包含某个方法的实现
BOOL class_respondsToSelector(Class cls, SEL sel)
//获取类中的方法列表
Method *class_copyMethodList(Class cls, unsigned int *outCount)
//为类添加新的方法,如果方法该方法已存在则返回NO
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)
//替换类中已有方法的实现,如果该方法不存在添加该方法
IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)
//获取类中的某个实例方法(减号方法)
Method class_getInstanceMethod(Class cls, SEL name)
//获取类中的某个类方法(加号方法)
Method class_getClassMethod(Class cls, SEL name)
//获取类中的方法实现
IMP class_getMethodImplementation(Class cls, SEL name)
//获取类中的方法的实现,该方法的返回值类型为struct
IMP class_getMethodImplementation_stret(Class cls, SEL name)
//获取Method中的SEL
SEL method_getName(Method m)
//获取Method中的IMP
IMP method_getImplementation(Method m)
//获取方法的Type字符串(包含参数类型和返回值类型)
const char *method_getTypeEncoding(Method m)
//获取参数个数
unsigned int method_getNumberOfArguments(Method m)
//获取返回值类型字符串
char *method_copyReturnType(Method m)
//获取方法中第n个参数的Type
char *method_copyArgumentType(Method m, unsigned int index)
//获取Method的描述
struct objc_method_description *method_getDescription(Method m)
//设置Method的IMP
IMP method_setImplementation(Method m, IMP imp)
//替换Method
void method_exchangeImplementations(Method m1, Method m2)
//获取SEL的名称
const char *sel_getName(SEL sel)
//注册一个SEL
SEL sel_registerName(const char *str)
//判断两个SEL对象是否相同
BOOL sel_isEqual(SEL lhs, SEL rhs)
//通过块创建函数指针,block的形式为^ReturnType(id self,参数,...)
IMP imp_implementationWithBlock(id block)
//获取IMP中的block
id imp_getBlock(IMP anImp)
//移出IMP中的block
BOOL imp_removeBlock(IMP anImp)
//调用target对象的sel方法
id objc_msgSend(id target, SEL sel, 参数列表...)
```

#[Method Swizzling 和 AOP](http://tech.glowing.com/cn/method-swizzling-aop/)
介绍一个技巧，最好的方式就是提出具体的需求，然后用它跟其他的解决方法做比较。
所以，先来看看我们的需求：对 App 的用户行为进行追踪和分析。简单说，就是当用户看到某个 View 或者点击某个 Button 的时候，就把这个事件记下来。
## 手动添加
最直接粗暴的方式就是在每个`viewDidAppear`里添加记录事件的代码。
```
	@implementation MyViewController ()

	- (void)viewDidAppear:(BOOL)animated
	{
	    [super viewDidAppear:animated];

	    // Custom code

	    // Logging
	    [Logging logWithEventName:@“my view did appear”];
	}


	- (void)myButtonClicked:(id)sender
	{
	    // Custom code

	    // Logging
	 [Logging logWithEventName:@“my button clicked”];
	}
```
这种方式的缺点也很明显：它破坏了代码的干净整洁。因为 Logging 的代码本身并不属于 ViewController 里的主要逻辑。随着项目扩大、代码量增加，你的 ViewController 里会到处散布着 Logging 的代码。这时，要找到一段事件记录的代码会变得困难，也很容易忘记添加事件记录的代码。

你可能会想到用继承或类别，在重写的方法里添加事件记录的代码。代码可以是长的这个样子：
```
	@implementation UIViewController ()

	- (void)myViewDidAppear:(BOOL)animated
	{
	    [super viewDidAppear:animated];

	    // Custom code

	    // Logging
	    [Logging logWithEventName:NSStringFromClass([self class])];
	}


	- (void)myButtonClicked:(id)sender
	{
	    // Custom code

	    // Logging
	    NSString *name = [NSString stringWithFormat:@“my button in %@ is clicked”, NSStringFromClass([self class])];
	    [Logging logWithEventName:name];
	}
```
Logging 的代码都很相似，通过继承或类别重写相关方法是可以把它从主要逻辑中剥离出来。但同时也带来新的问题：
1. 你需要继承 UIViewController, UITableViewController, UICollectionViewController 所有这些 ViewController ，或者给他们添加类别；
2. 每个 ViewController 里的 ButtonClick 方法命名不可能都一样；
3. 你不能控制别人如何去实例化你的子类；
4. 对于类别，你没办法调用到原来的方法实现。大多时候，我们重写一个方法只是为了添加一些代码，而不是完全取代它。
5. 如果有两个类别都实现了相同的方法，运行时没法保证哪一个类别的方法会给调用。
## Method Swizzling
Method Swizzling 利用 Runtime 特性把一个方法的实现与另一个方法的实现进行替换。
每个类里都有一个 Dispatch Table ，将方法的名字（SEL）跟方法的实现（IMP，指向 C 函数的指针）一一对应。Swizzle 一个方法其实就是在程序运行时在 Dispatch Table 里做点改动，让这个方法的名字（SEL）对应到另个 IMP 。

首先定义一个类别，添加将要 Swizzled 的方法：
```
	@implementation UIViewController (Logging)

	- (void)swizzled_viewDidAppear:(BOOL)animated
	{
	    // call original implementation
	    [self swizzled_viewDidAppear:animated];

	    // Logging
	    [Logging logWithEventName:NSStringFromClass([self class])];
	}
```
代码看起来可能有点奇怪，像递归不是么。当然不会是递归，因为在 runtime 的时候，函数实现已经被交换了。调用 `viewDidAppear: `会调用你实现的 `swizzled_viewDidAppear:`，而在 `swizzled_viewDidAppear: `里调用` swizzled_viewDidAppear:` 实际上调用的是原来的 `viewDidAppear:` 。
接下来实现 swizzle 的方法 ：
```
	@implementation UIViewController (Logging)

	void swizzleMethod(Class class, SEL originalSelector, SEL swizzledSelector)
	{
	    // the method might not exist in the class, but in its superclass
	    Method originalMethod = class_getInstanceMethod(class, originalSelector);
	    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

	    // class_addMethod will fail if original method already exists
	    BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));

	    // the method doesn’t exist and we just added one
	    if (didAddMethod) {
	        class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
	 }
	    else {
	        method_exchangeImplementations(originalMethod, swizzledMethod);
	    }
	}
```
这里唯一可能需要解释的是 `class_addMethod` 。要先尝试添加原 selector 是为了做一层保护，因为如果这个类没有实现 originalSelector ，但其父类实现了，那 `class_getInstanceMethod` 会返回父类的方法。这样 `method_exchangeImplementations`替换的是父类的那个方法，这当然不是你想要的。所以我们先尝试添加 `orginalSelector`，如果已经存在，再用 `method_exchangeImplementations`把原方法的实现跟新的方法实现给交换掉。

最后，我们只需要确保在程序启动的时候调用 `swizzleMethod` 方法。比如，我们可以在之前 `UIViewController `的 `Logging` 类别里添加 `+load:` 方法，然后在 `+load: `里把 `viewDidAppear` 给替换掉：
```
	@implementation UIViewController (Logging)

	+ (void)load
	{
	    swizzleMethod([self class], @selector(viewDidAppear:), @selector(swizzled_viewDidAppear:));
	}
```
一般情况下，类别里的方法会重写掉主类里相同命名的方法。如果有两个类别实现了相同命名的方法，只有一个方法会被调用。但` +load:` 是个特例，当一个类被读到内存的时候， runtime 会给这个类及它的每一个类别都发送一个 `+load:` 消息。

其实，这里还可以更简化点：直接用新的 IMP 取代原 IMP ，而不是替换。只需要有全局的函数指针指向原 IMP 就可以。
```
	void (gOriginalViewDidAppear)(id, SEL, BOOL);

	void newViewDidAppear(UIViewController *self, SEL _cmd, BOOL animated)
	{
	    // call original implementation
	    gOriginalViewDidAppear(self, _cmd, animated);

	    // Logging
	    [Logging logWithEventName:NSStringFromClass([self class])];
	}

	+ (void)load
	{
	    Method originalMethod = class_getInstanceMethod(self, @selector(viewDidAppear:));
	    gOriginalViewDidAppear = (void *)method_getImplementation(originalMethod);

	if(!class_addMethod(self, @selector(viewDidAppear:), (IMP) newViewDidAppear, method_getTypeEncoding(originalMethod))) {
	        method_setImplementation(originalMethod, (IMP) newViewDidAppear);
	    }
	}
```
通过 Method Swizzling ，我们成功把逻辑代码跟处理事件记录的代码解耦。当然除了 Logging ，还有很多类似的事务，如 Authentication 和 Caching。这些事务琐碎，跟主要业务逻辑无关，在很多地方都有，又很难抽象出来单独的模块。这种程序设计问题，业界也给了他们一个名字 - Cross Cutting Concerns。

而像上面例子用 Method Swizzling 动态给指定的方法添加代码，以解决 Cross Cutting Concerns 的编程方式叫：Aspect Oriented Programming

## Aspect Oriented Programming （面向切面编程）
Wikipedia 里对 AOP 是这么介绍的:
> An aspect can alter the behavior of the base code by applying advice (additional behavior) at various join points (points in a program) specified in a quantification or query called a pointcut (that detects whether a given join point matches).

在 Objective-C 的世界里，这句话意思就是利用 Runtime 特性给指定的方法添加自定义代码。有很多方式可以实现 AOP ，Method Swizzling 就是其中之一。而且幸运的是，目前已经有一些第三方库可以让你不需要了解 Runtime ，就能直接开始使用 AOP 。

Aspects 就是一个不错的 AOP 库，封装了 Runtime ， Method Swizzling 这些黑色技巧，只提供两个简单的API：
```
	+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
	                          withOptions:(AspectOptions)options
	                       usingBlock:(id)block
	                            error:(NSError **)error;
	- (id<AspectToken>)aspect_hookSelector:(SEL)selector
	                      withOptions:(AspectOptions)options
	                       usingBlock:(id)block
	                            error:(NSError **)error;
```
使用 Aspects 提供的 API，我们之前的例子会进化成这个样子：
```
	@implementation UIViewController (Logging)

	+ (void)load
	{
	    [UIViewController aspect_hookSelector:@selector(viewDidAppear:)
	                              withOptions:AspectPositionAfter
	                               usingBlock:^(id<AspectInfo> aspectInfo) {
	        NSString *className = NSStringFromClass([[aspectInfo instance] class]);
	        [Logging logWithEventName:className];
	                               } error:NULL];
	}
```

你可以用同样的方式在任何你感兴趣的方法里添加自定义代码，比如 IBAction 的方法里。更好的方式，你提供一个 Logging 的配置文件作为唯一处理事件记录的地方：
```
	@implementation AppDelegate (Logging)

	+ (void)setupLogging
	{
	    NSDictionary *config = @{
	        @"MainViewController": @{
	            GLLoggingPageImpression: @"page imp - main page",
	            GLLoggingTrackedEvents: @[
	                @{
	                    GLLoggingEventName: @"button one clicked",
	                    GLLoggingEventSelectorName: @"buttonOneClicked:",
	                    GLLoggingEventHandlerBlock: ^(id<AspectInfo> aspectInfo) {
	                        [Logging logWithEventName:@"button one clicked"];
	                    },
	                },
	                @{GLLoggingEventName: @"button two clicked",
	                    GLLoggingEventSelectorName: @"buttonTwoClicked:",
	                    GLLoggingEventHandlerBlock: ^(id<AspectInfo> aspectInfo) {
	                        [Logging logWithEventName:@"button two clicked"];
	                    },
	                },
	           ],
	        },

	        @"DetailViewController": @{
	            GLLoggingPageImpression: @"page imp - detail page",
	        }
	    };

	    [AppDelegate setupWithConfiguration:config];
	}
	+ (void)setupWithConfiguration:(NSDictionary *)configs
	{
	    // Hook Page Impression
	    [UIViewController aspect_hookSelector:@selector(viewDidAppear:)
	                              withOptions:AspectPositionAfter
	                               usingBlock:^(id<AspectInfo> aspectInfo) {
	                                       NSString *className = NSStringFromClass([[aspectInfo instance] class]);
	                                    [Logging logWithEventName:className];
	                               } error:NULL];
	 // Hook Events
	    for (NSString *className in configs) {
	        Class clazz = NSClassFromString(className);
	        NSDictionary *config = configs[className];

	        if (config[GLLoggingTrackedEvents]) {
	            for (NSDictionary *event in config[GLLoggingTrackedEvents]) {
	                SEL selekor = NSSelectorFromString(event[GLLoggingEventSelectorName]);
	                AspectHandlerBlock block = event[GLLoggingEventHandlerBlock];

	                [clazz aspect_hookSelector:selekor
	                               withOptions:AspectPositionAfter
	                                usingBlock:^(id<AspectInfo> aspectInfo) {
	                                    block(aspectInfo);
	                                } error:NULL];

	            }
	        }
	    }
	}
```
然后在 `-application:didFinishLaunchingWithOptions:` 里调用 `setupLogging`：
```
	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
	    // Override point for customization after application launch.

	    [self setupLogging];
	    return YES;
	}
```

### Associated Objects
- [Objective-C Associated Objects 的实现原理](http://blog.leichunfeng.com/blog/2015/06/26/objective-c-associated-objects-implementation-principle/)

### MethodSwizzling
- [MethodSwizzling](http://cocoadev.com/MethodSwizzling)
- [Objective-C Method Swizzling 的最佳实践](http://blog.leichunfeng.com/blog/2015/06/14/objective-c-method-swizzling-best-practice/)

## 最后的话
利用 objective-C Runtime 特性和 Aspect Oriented Programming ，我们可以把琐碎事务的逻辑从主逻辑中分离出来，作为单独的模块。它是对面向对象编程模式的一个补充。Logging 是个经典的应用，这里做个抛砖引玉，发挥想象力，可以做出其他有趣的应用。

参考文章：
- [Runtime 10种用法](http://www.jianshu.com/p/3182646001d1)最全总结
- [**Objective-C Runtime 1小时入门教程**](http://www.ianisme.com/ios/2019.html) -ian
- [**刨根问底Objective－C Runtime**](http://chun.tips/blog/2014/11/05/bao-gen-wen-di-objective[nil]c-runtime(1)[nil]-self-and-super/)
- [理解 Objective-C Runtime](http://justinyan.me/post/1624)
- [Objective-C的动态特性](http://limboy.me/ios/2013/08/03/dynamic-tips-and-tricks-with-objective-c.html)
