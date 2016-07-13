# block和对象的内存管理
## 获取对象
block获取外部对象的
```
/********************** capturing objects **********************/
typedef void (^blk_t)(id obj);
blk_t blk;
- (void)viewDidLoad
{
    [self captureObject];
    blk([[NSObject alloc] init]);
    blk([[NSObject alloc] init]);
    blk([[NSObject alloc] init]);
}
- (void)captureObject
{
    id array = [[NSMutableArray alloc] init];
    blk = [^(id obj) {
             [array addObject:obj];
             NSLog(@"array count = %ld", [array count]);
          } copy];
}
```
`clang -rewrite-objc`编译后
```
/* a struct for the Block and some functions */
struct __main_block_impl_0
{
    struct __block_impl impl;
    struct __main_block_desc_0 *Desc;
    id __strong array;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, id __strong _array, int flags=0) : array(_array)
    {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself, id obj)
{
    id __strong array = __cself->array;
    [array addObject:obj];
    NSLog(@"array count = %ld", [array count]);
}
static void __main_block_copy_0(struct __main_block_impl_0 *dst, __main_block_impl_0 *src)
{
    _Block_object_assign(&dst->array, src->array, BLOCK_FIELD_IS_OBJECT);
}
static void __main_block_dispose_0(struct __main_block_impl_0 *src)
{
    _Block_object_dispose(src->array, BLOCK_FIELD_IS_OBJECT);
}
struct static struct __main_block_desc_0
{
    unsigned long reserved;
    unsigned long Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = {  0,
                                sizeof(struct __main_block_impl_0),
                                __main_block_copy_0,
                                __main_block_dispose_0
                             };
/* Block literal and executing the Block */
blk_t blk;
{
    id __strong array = [[NSMutableArray alloc] init];
    blk = &__main_block_impl_0(__main_block_func_0,
                               &__main_block_desc_0_DATA,
                               array,
                               0x22000000);
    blk = [blk copy];
}
(*blk->impl.FuncPtr)(blk, [[NSObject alloc] init]);
(*blk->impl.FuncPtr)(blk, [[NSObject alloc] init]);
(*blk->impl.FuncPtr)(blk, [[NSObject alloc] init]);
```
在本例中，当变量变量作用域结束时，array 被废弃，强引用失效，NSMutableArray 类的实例对象会被释放并废弃。在这危难关头，block 及时调用了 copy 方法，在 `_Block_object_assign` 中，将 array 赋值给 block 成员变量并持有。所以上面代码可以正常运行，打印出来的 `array count`依次递增。

总结代码可正常运行的原因关键就在于 block 通过调用 copy 方法，持有了 `__strong` 修饰的外部变量，使得外部对象在超出其作用域后得以继续存活，代码正常执行。

在以下情形中， block 会从栈拷贝到堆：
* 当 block 调用 copy 方法时，如果 block 在栈上，会被拷贝到堆上；
* 当 block 作为函数返回值返回时，编译器自动将 block 作为 `_Block_copy` 函数，效果等同于 block 直接调用 `copy` 方法；
* 当 block 被赋值给 `__strong id` 类型的对象或 block 的成员变量时，编译器自动将 block 作为 `_Block_copy` 函数，效果等同于 block 直接调用 `copy` 方法；
* 当 block 作为参数被传入方法名带有 `usingBlock` 的 Cocoa Framework 方法或 GCD 的 API 时。这些方法会在内部对传递进来的 block 调用 `copy` 或 `_Block_copy` 进行拷贝;

除此之外，都需要手动调用。

>**延伸阅读：Objective-C 结构体中的 __strong 成员变量**
注意到 `__main_block_impl_0` 结构体有什么异常没？在 C 结构体中出现了 `__strong` 关键字修饰的变量。
通常情况下， Objective-C 的编译器因为无法检测 C 结构体初始化和释放的时间，不能进行有效的内存管理，所以 Objective-C 的 C 结构体成员是不能用 `__strong`、`__weak` 等等这类关键字修饰。然而 runtime 库是可以在运行时检测到 block 的内存变化，如 block 何时从栈拷贝到堆，何时从堆上释放等等，所以就会出现上述结构体成员变量用 `__strong` 修饰的情况。

## __block 变量和对象
__block 说明符可以修饰任何类型的自动变量。
```
/******* block 修饰对象 *******/
__block id obj = [[NSObject alloc] init];
```
ARC 下，对象所有权修饰符默认为 `__strong`，即
```
__block id __strong obj = [[NSObject alloc] init];
```

```
/******* block 修饰对象转换后的代码 *******/
/* struct for __block variable */
struct __Block_byref_obj_0
{
    void *__isa;
    __Block_byref_obj_0 *__forwarding;
    int __flags;
    int __size;
    void (*__Block_byref_id_object_copy)(void*, void*);
    void (*__Block_byref_id_object_dispose)(void*);
    __strong id obj;
};
static void __Block_byref_id_object_copy_131(void *dst, void *src)
{
    _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}
static void __Block_byref_id_object_dispose_131(void *src)
{
    _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
/* __block variable declaration */
__Block_byref_obj_0 obj = { 0,
                            &obj,
                            0x2000000,
                            sizeof(__Block_byref_obj_0),
                            __Block_byref_id_object_copy_131,
                            __Block_byref_id_object_dispose_131,
                            [[NSObject alloc] init]
                           };
```

`__block id __strong obj`的作用和`id __strong obj`的作用十分类似。当`__block id __strong obj`从栈上拷贝到堆上时,`_Block_object_assign`被调用，block 持有`obj`；当`__block id __strong obj`从堆上被废弃时,`_Block_object_dispose`被调用用以释放此对象，block 引用消失。

所以，只要是堆上的`__strong`修饰符修饰的`__block`对象类型的变量，和 block 内获取到的`__strong`修饰符修饰的对象类型的变量，编译器都能对它们的内存进行适当的管理。

如果上面的`__strong`换成`__weak`，结果会怎样呢？
```
/********************** capturing __weak objects **********************/
typedef void (^blk_t)(id obj);
blk_t blk;
- (void)viewDidLoad
{
    [self captureObject];
    blk([[NSObject alloc] init]);
    blk([[NSObject alloc] init]);
    blk([[NSObject alloc] init]);
}
- (void)captureObject
{
    id array = [[NSMutableArray alloc] init];
    id __weak array2 = array;
    blk = [^(id obj) {
             [array2 addObject:obj];
             NSLog(@"array2 count = %ld", [array2 count]);
          } copy];
}
```
结果是
```
array2 count = 0
array2 count = 0
array2 count = 0
```
原因很简单，array2 是弱引用，当变量作用域结束，array 所指向的对象内存被释放，array2 指向 nil，向 nil 对象发送 count 消息就返回结果 0 了。

如果 `__weak` 再改成 `__unsafe_unretained` 呢？`__unsafe_unretained` 修饰的对象变量指针就相当于一个普通指针。使用这个修饰符有点需要注意的地方是，当指针所指向的对象内存被释放时，指针变量不会被置为 nil。所以当使用这个修饰符时，一定要注意不要通过悬挂指针（指向被废弃内存的指针）来访问已经被废弃的对象内存，否则程序就会崩溃。

如果 `__unsafe_unretained` 再改成` __autoreleasing` 会怎样呢？会报错，编译器并不允许你这么干！如果你这么写
```
__block id __autoreleasing obj = [[NSObject alloc] init];
```
编译器就会报下面的错误，意思就是 `__block` 和 `__autoreleasing` 不能同时使用。
```
error: __block variables cannot have __autoreleasing ownership __block id __autoreleasing obj = [[NSObject alloc] init];
```
### 循环引用
block 如果使用不小心，就容易出现循环引用，导致内存泄露。到底哪里泄露了呢？
```
// ARC enabled
/************** MyObject Class **************/
typedef void (^blk_t)(void);
@interface MyObject : NSObject
{
    blk_t blk_;
}
@end
@implementation MyObject
- (id)init
{
    self = [super init];
    blk_ = ^{NSLog(@"self = %@", self);};
    return self;
}
- (void)dealloc
{
    NSLog(@"dealloc");
}
@end
/************** main function **************/
int main()
{
    id myObject = [[MyObject alloc] init];
    NSLog(@"%@", myObject);
    return 0;
}
```
由于 `self` 是 `__strong` 修饰，在 ARC 下，当编译器自动将代码中的 block 从栈拷贝到堆时，block 会强引用和持有 `self`，而 `self` 恰好也强引用和持有了 block，就造成了传说中的循环引用。

![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note58470_2.png)

由于循环引用的存在，造成在 main() 函数结束时，内存仍然无法释放，即内存泄露。编译器也会给出警告信息
> warning: capturing 'self' strongly in this block is likely to lead to a retain cycle [-Warc-retain-cycles]
blk_ = ^{NSLog(@"self = %@", self);};
note: Block will be retained by an object strongly retained by the captured object
blk_ = ^{NSLog(@"self = %@", self);};

为了避免这种情况发生，可以在变量声明时用 `__weak` 修饰符修饰变量 `self`，让 block 不强引用 `self`，从而破除循环。iOS4 和 Snow Leopard 由于对 weak 的支持不够完全，可以用 `__unsafe_unretained` 代替。
```
- (id)init
{
    self = [super init];
    id __weak tmp = self;
    blk_ = ^{NSLog(@"self = %@", tmp);};
    return self;
}
```

![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note58470_3.png)

再看一个例子
```
@interface MyObject : NSObject
{
    blk_t blk_;
    id obj_;
}
@end
@implementation MyObject
- (id)init
{
    self = [super init];
    blk_ = ^{ NSLog(@"obj_ = %@", obj_); };
    return self;
}
...
...
@end
```
上面的例子中，虽然没有直接使用 self，却也存在循环引用的问题。因为对于编译器来说，`obj_` 就相当于 `self->obj_`，所以上面的代码就会变成
```
blk_ = ^{ NSLog(@"obj_ = %@", self->obj_); };
```
所以这个例子只要用 `__weak`，在 `init` 方法里面加一行即可
```
id __weak obj = obj_;
```
破解循环引用还有一招，使用 __block 修饰对象，在 block 内将对象置为 nil 即可，如下
```
typedef void (^blk_t)(void);
@interface MyObject : NSObject
{
    blk_t blk_;
}
@end
@implementation MyObject
- (id)init
{
    self = [super init];
    __block id tmp = self;
    blk_ = ^{
                NSLog(@"self = %@", tmp);
                tmp = nil;
            };
    return self;
}
- (void)execBlock
{
    blk_();
}
- (void)dealloc
{
    NSLog(@"dealloc");
}
@end
int main()
{
    id object = [[MyObject alloc] init];
    [object execBlock];
    return 0;
}
```

这个例子挺有意思的，如果执行 `execBlock` 方法，就没有循环引用，如果不执行就有循环引用，挺值得玩味的。一方面，使用 __block 挺危险的，万一代码中不执行 block ，就造成了循环引用，而且编译器还没法检查出来；另一方面，使用 __block 可以让我们通过 __block 变量去控制对象的生命周期，而且有可能在一些非常老旧的 MRC 代码中，由于不支持 __weak，我们可以使用此方法来代替 __unsafe_unretained，从而避免悬挂指针的问题。

还有个值得一提的时，在 MRC 下，使用 __block 说明符也可以避免循环引用。因为当 block 从栈拷贝到堆时，__block 对象类型的变量不会被 retain，没有 __block 说明符的对象类型的变量则会被 retian。正是由于 __block 在 ARC 和 MRC 下的巨大差异，我们在写代码时一定要区分清楚到底是 ARC 还是 MRC。

>尽管 ARC 已经如此普及，我们可能已经可以不用去管 MRC 的东西，但要有点一定要明白，ARC 和 MRC 都是基于引用计数的内存管理，其本质上是一个东西，只不过 ARC 在编译期自动化的做了内存引用计数的管理，使得系统可以在适当的时候保留内存，适当的时候释放内存。

### Copy 和 Release
在 ARC 下，有时需要手动拷贝和释放 block。在 MRC 下更是如此，可以直接用 `copy` 和 `release` 来拷贝和释放
```
void (^blk_on_heap)(void) = [blk_on_stack copy];
[blk_on_heap release];
```
拷贝到堆后，就可以 用 retain 持有 block
```
[blk_on_heap retain];
```
然而如果 block 在栈上，使用 `retain` 是毫无效果的，因此推荐使用 `copy` 方法来持有 block。

block 是 C 语言的扩展，所以可以在 C 中使用 block 的语法。比如，在上面的例子中，可以直接使用 `Block_copy` 和 `Block_release` 函数来代替 `copy` 和 `release` 方法
```
void (^blk_on_heap)(void) = Block_copy(blk_on_stack);
Block_release(blk_on_heap);
```
`Block_copy` 的作用相当于之前看到过的 `_Block_copy` 函数，而且 Objective-C runtime 库在运行时拷贝 block 用的就是这个函数。同理，释放 block 时，runtime 调用了 `Block_release` 函数。

最后这里有一篇总结 block 的文章的很不错，推荐大家看看：http://tanqisen.github.io/blog/2013/04/19/gcd-block-cycle-retain/
