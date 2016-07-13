# block和变量的内存管理
在 block 内获取了一个外部的局部变量，可以读取，但无法进行写入的修改操作。在 C 语言中有三种类型的变量，可在 block 内进行读写操作
>* 全局变量
* 全局静态变量
* 静态变量

`全局变量`和`全局静态变量`由于作用域在全局，所以在 block 内访问和读写这两类变量和普通函数没什么区别，而`静态变量`作用域在 block 之外，是怎么对它进行读写呢？通过 clang 工具，我们发现原来 `静态变量`是通过指针传递，将变量传递到 block 内，所以可以修改变量值。block的实现一文中的外部变量是通过值传递，自然没法对获取到的外部变量进行修改。由此，可以给我们一个启示，当我们需要修改外部变量时，是不是也可以像`静态变量`这样通过指针来修改外部变量的值呢？

Apple 早就为我们准备了这么一个东西 —— “__block”

## __block 说明符

使用 __block 的源码
```
int main()
{
    __block int intValue = 0;
    void (^blk)(void) = ^{
        intValue = 1;
    };
    return 0;
}
```
使用 clang 翻译后

```
struct __block_impl
{
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
struct __Block_byref_intValue_0
{
    void *__isa;
    __Block_byref_intValue_0 *__forwarding;
    int __flags;
    int __size;
    int intValue;
};
struct __main_block_impl_0
{
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __Block_byref_intValue_0 *intValue; // by ref
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_intValue_0 *_intValue, int flags=0) : intValue(_intValue->__forwarding)
    {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    __Block_byref_intValue_0 *intValue = __cself->intValue; // bound by ref
    (intValue->__forwarding->intValue) = 1;
}
static void __main_block_copy_0(struct __main_block_impl_0 *dst, struct __main_block_impl_0 *src)
{
    _Block_object_assign((void*)&dst->intValue, (void*)src->intValue, 8/*BLOCK_FIELD_IS_BYREF*/);
}
static void __main_block_dispose_0(struct __main_block_impl_0 *src)
{
    _Block_object_dispose((void*)src->intValue, 8/*BLOCK_FIELD_IS_BYREF*/);
}
static struct __main_block_desc_0
{
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = {  0,
                                sizeof(struct __main_block_impl_0),
                                __main_block_copy_0,
                                __main_block_dispose_0
                             };
int main()
{
    __attribute__((__blocks__(byref))) __Block_byref_intValue_0 \
    intValue =
    {
        (void*)0,
        (__Block_byref_intValue_0 *)&intValue,
        0,
        sizeof(__Block_byref_intValue_0),
        0
    };
    void (*blk)(void) = (void (*)()) &__main_block_impl_0   \
                (
                    (void *)__main_block_func_0,            \
                    &__main_block_desc_0_DATA,              \
                    (__Block_byref_intValue_0 *)&intValue,  \
                    570425344                               \
                );
    return 0;
}
```
在加了 __block 之后，比原来多了
> 1. `__Block_byref_intValue_0`结构体：用于封装 __block 修饰的外部变量。
2. `_Block_object_assign`函数：当 block 从栈拷贝到堆时，调用此函数。
3. `_Block_object_dispose`函数：当 block 从堆内存释放时，调用此函数。

OC源码中的 `__block intValue`翻译后变成了 `__Block_byref_intValue_0`结构体指针变量 `intValue`，通过指针传递到`block`内，这与前面说的`静态变量`的指针传递是一致的。除此之外，整体的执行流程与不加 __block 基本一致，不再赘述。但 `__Block_byref_intValue_0` 这个结构体需特别注意下

```
// 存储 __block 外部变量的结构体
struct __Block_byref_intValue_0
{
    void *__isa; // 对象指针
    __Block_byref_intValue_0 *__forwarding; // 指向自己的指针
    int __flags; // 标志位变量
    int __size; // 结构体大小
    int intValue; // 外部变量
};
```
![Pointer](http://7xs5iw.com1.z0.glb.clouddn.com/image_note57603_1.png)

在已有结构体指针指向 `__Block_byref_intValue_0` 时，结构体里面还多了个`__forwarding`指向自己的指针变量，难道不显得多余吗？一点也不，本文后面会阐述。

## block 的内存管理
在前文中，已经提到了 block 的三种类型 `NSConcreteGlobalBlock`、`_NSConcreteStackBlock`、`_NSConcreteMallocBlock`，见名知意，可以看出三种 block 在内存中的分布

![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note57603_2.png)

### _NSConcreteGlobalBlock
> 1. 当 block 字面量写在全局作用域时，即为 global block；
2. 当 block 字面量不获取任何外部变量时，即为 global block；

除了上述描述的两种情况，其他形式创建的 block 均为 stack block。
```
// 下面 block 虽然定义在 for 循环内，但符合第二种情况，所以也是 global block
typedef int (^blk_t)(int);
for (int rate = 0; rate < 10; ++rate)
{
    blk_t blk = ^(int count){return rate * count;};
}
```
`_NSConcreteGlobalBlock` 类型的 block 处于内存的 ROData 段，此处没有局部变量的骚扰，运行不依赖上下文，内存管理也简单的多。

### _NSConcreteStackBlock
_NSConcreteStackBlock 类型的 block 处于内存的栈区。global block 由于处在 data 段，可以通过指针安全访问，但 stack block 处在内存栈区，如果其变量作用域结束，这个 block 就被废弃，block 上的 __block 变量也同样会被废弃。
![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note57603_3.png)

为了解决这个问题，block 提供了 copy 的功能，将 block 和 __block 变量从栈拷贝到堆，就是下面要说的 _NSConcreteMallocBlock。

### _NSConcreteMallocBlock

当 block 从栈拷贝到堆后，当栈上变量作用域结束时，仍然可以继续使用 block

![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note57603_4.png)

此时，堆上的 block 类型为`_NSConcreteMallocBlock`，所以会将`_NSConcreteMallocBlock`写入 `isa`
```
impl.isa = &_NSConcreteMallocBlock;
```
如果你细心的观察上面的转换后的代码，会发现访问结构体`__Block_byref_intValue_0`内部的成员变量都是通过访问`__forwarding`指针完成的。为了保证能正确访问栈上的`__block`变量，进行 copy 操作时，会将栈上的`__forwarding`指针指向了堆上的 block 结构体实例。

## block 的自动拷贝和手动拷贝
在开启 ARC 时，大部分情况下编译器通常会将创建在栈上的 block 自动拷贝到堆上，只有当
>block 作为方法或函数的参数传递时，编译器不会自动调用 copy 方法；

但方法/函数在内部已经实现了一份拷贝了 block 参数的代码，或者如果编译器自动拷贝，那么调用者就不需再手动拷贝，比如：
>- 当 block 作为函数返回值返回时，编译器自动将 block 作为`_Block_copy`函数，效果等同于 block 直接调用`copy`方法；
- 当 block 被赋值给 __strong id 类型的对象或 block 的成员变量时，编译器自动将 block 作为`_Block_copy`函数，效果等同于 block 直接调用`copy`方法；
- 当 block 作为参数被传入方法名带有`usingBlock`的 Cocoa Framework 方法或 GCD 的 API 时。这些方法会在内部对传递进来的 block 调用`copy`或`_Block_copy`进行拷贝;

让我们看个 block 自动拷贝的例子
```
/************ ARC下编译器自动拷贝block ************/
typedef int (^blk_t)(int);
blk_t func(int rate)
{
    return ^(int count){return rate * count;};
}
```
上面的 block 获取了外部变量，所以是创建在栈上，当`func`函数返回给调用者时，脱离了局部变量`rate`的作用范围，如果调用者使用这个 block 就会出问题。那 ARC 开启的情况呢？运行这个 block 一切正常。和我们的预期结果不一样，ARC 到底给 block 施了什么魔法？我们将上面的代码翻译下
```
blk_t func(int rate)
{
    blk_t tmp = &__func_block_impl_0(__func_block_func_0, &__func_block_desc_0_DATA, rate);
    tmp = objc_retainBlock(tmp);
    return objc_autoreleaseReturnValue(tmp);
}
```
转换后出现两个新函数`objc_retainBlock`、`objc_autoreleaseReturnValue`。如果你看过runtime 库 ，在`runtime/objc-arr.mm`文件中就有这两个函数的实现：
```Objective-C
/*********** objc_retainBlock() 的实现 ***********/
id objc_retainBlock(id x)
{
#if ARR_LOGGING
    objc_arr_log("objc_retain_block", x);
    ++CompilerGenerated.blockCopies;
#endif
    return (id)_Block_copy(x);
}
// Create a heap based copy of a Block or simply add a reference to an existing one.
// This must be paired with Block_release to recover memory, even when running
// under Objective-C Garbage Collection.
BLOCK_EXPORT void *_Block_copy(const void *aBlock)
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
```

```Objective-C
/*********** objc_autoreleaseReturnValue() 的实现 ***********/
id objc_autoreleaseReturnValue(id obj)
{
#if SUPPORT_RETURN_AUTORELEASE
    assert(_pthread_getspecific_direct(AUTORELEASE_POOL_RECLAIM_KEY) == NULL);
    if (callerAcceptsFastAutorelease(__builtin_return_address(0))) {
        _pthread_setspecific_direct(AUTORELEASE_POOL_RECLAIM_KEY, obj);
        return obj;
    }
#endif
    return objc_autorelease(obj);
}
```

通过上面的代码和注释，意思就很明显了，由于 block 字面量是创建在栈内存，通过`objc_retainBlock()`函数拷贝到堆内存，让`tmp`重新指向堆上的 block，然后将`tmp`所指的堆上的 block 作为一个 Objective-C 对象放入 autoreleasepool 里面，从而保证了返回后的 block 仍然可以正确执行。

看完了 block 的自动拷贝，那么看看在 ARC 下需要手动拷贝 block 的例子
```
/************ ARC下编译器手动拷贝block ************/
- (id)getBlockArray
{
    int val = 10;
    return [[NSArray alloc] initWithObjects:
                            ^{NSLog(@"blk0:%d", val);},
                            ^{NSLog(@"blk1:%d", val);}, nil];
}
```
一个例子就了然，返回的数组里面的 block 是不可用的，需要再手动拷贝一次才可以，这个较为简单，就不作过多解释。

关于 block 的拷贝操作可以用一张表总结下

| Block Class    | Copied From | How "Copy" Works |
| :------------- | :------------- | :-------------|
| _NSConcreteStackBlock  | Stack   |Copy from the stack to the heap|
|_NSConcreteGlobalBlock | .data section of the program | Do nothing|
|_NSConcreteMallocBlock| Heap|Increment the reference count of the object|


**block的多次拷贝**：下面的例子在 ARC 下并不会产生内存泄露
```
// block 多次拷贝源码
blk = [[[[blk copy] copy] copy] copy];
```
clang编译后，
```
{
    blk_t tmp = [blk copy];
    blk = tmp;
}
{
    blk_t tmp = [blk copy];
    blk = tmp;
}
{
    blk_t tmp = [blk copy];
    blk = tmp;
}
{
    blk_t tmp = [blk copy];
    blk = tmp;
}
```

## __block 变量的内存管理
* 当 block 从栈内存被拷贝到堆内存时，__block 变量的变化如下图。需要说明的是，当栈上的 block 被拷贝到堆上，堆上的 block 再次被拷贝时，对 __block 变量已经没有影响了。
![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note57603_6.png)
![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note57603_7.png)
* 当多个 block 获取同一个 __block 变量，block 从栈被拷贝到堆时
![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note57603_8.png)
* 当 block 被废弃时，__block 变量被释放
![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note57603_9.png)

* __forwarding

前文已经说过，当 block 从栈被拷贝到堆时，`__forwarding`指针变量也会指向堆区的结构体。但是为什么要这么做呢？为什么要让原本指向栈区的结构体的指针，去指向堆区的结构体呢？看起来匪夷所思，实则原因很简单，要从 `__forwarding`产生的缘由说起。想想起初为什么要给 block 添加 copy 的功能，就是因为 block 获取了局部变量，当要在其他地方（超出局部变量作用范围）使用这个 block 的时候，由于访问局部变量异常，导致程序崩溃。为了解决这个问题，就给 block 添加了 copy 功能。在将 block 拷贝到堆上的同时，将`__forwarding`指针指向堆上结构体。后面如果要想使用 __block 变量，只要通过`__forwarding`访问堆上变量，就不会出现程序崩溃了。

```
/*************** __forwarding 的作用 ***************/
//猜猜下面代码的打印结果？
{
    __block int val = 0;
    void (^blk)(void) = [^{++val;} copy];
    ++val;
    blk();
    NSLog(@"%d", val);
}
```
一定有很多人会猜 1，其实打印 2。原因很简单，当栈上的 block 被拷贝到堆上时，栈上的`__forwarding`也会指向堆上的 __block 变量的结构体。

上面的代码中`^{++val;}`和 `++val`; 都会被转换成`++(val.__forwarding->val)`;，堆上的 val 被加了两次，最后打印堆上的 val 为 2。

![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note57603_10.png)
