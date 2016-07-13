# block的实现
>block 只是 Objective-C 对闭包的实现，并不是 iOS 独有的概念，在 C++、Java 等语言也有实现闭包，名称不同而已。

test.m 代码
```
int main()
{
    void (^blk)(void) = ^{ printf("Block\n"); };
    blk();
    return 0;
}
```

## 将test.m 代码用clang工具翻译test.cpp代码

```
clang -rewrite-objc test.m
```

test.cpp

```
struct __block_impl
{
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
struct __main_block_impl_0
{
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0)
    {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    printf("Block\n");
}
static struct __main_block_desc_0
{
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0) };
int main()
{
    void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA);
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    return 0;
}
```

## block 结构体信息详解
### struct __block_impl

```
// __block_impl 是 block 实现的结构体
struct __block_impl
{
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
```

* `isa`

指向实例对象，表明 block 本身也是一个 Objective-C 对象。block 的三种类型：`_NSConcreteStackBlock`、`_NSConcreteGlobalBlock`、`_NSConcreteMallocBlock`，即当代码执行时，isa 有三种值
>- impl.isa = &_NSConcreteStackBlock;
- impl.isa = &_NSConcreteMallocBlock;
- impl.isa = &_NSConcreteGlobalBlock;

>从这些实例对象可以看出 block 所在内存区域分别为stack、ROData、heap，后面文章会详说。

* `Flags`
按位承载 block 的附加信息；
* `Reserved`
保留变量；
* `FuncPtr`
函数指针，指向block要执行的函数，即{ printf("Block\n") };

### struct __main_block_impl_0

```
// __main_block_impl_0 是 block 实现的结构体，也是 block 实现的入口
struct __main_block_impl_0
{
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0)
    {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
```

* `impl`
block实现的结构体变量。
* `Desc`
描述block的结构体变量
* `__main_block_impl_0`
结构体的构造函数，初始化结构体变量`impl`、`Desc`。

### static void __main_block_func_0
```
// __main_block_func_0 是 block 要最终要执行的函数代码
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    printf("Block\n");
}
```

### static struct __main_block_desc_0
```
// __main_block_desc_0 是 block 的描述信息结构体
static struct __main_block_desc_0
{
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0) };
```
* `reserved`
结构体信息保留字段
* `Block_size`
结构体大小
此处已定义了一个该结构体类型的变量 `__main_block_desc_0_DATA`

## block 实现的执行流程
1. main()
2. 调用 __main_block_impl_0 构造函数初始化结构体__main_block_impl_0（__main_block_func_0 , __main_block_desc_0_DATA）;
3. 得到的__main_block_impl_0 类型变量赋值给 blk
4. 执行 blk->FuncPtr()函数，即 printf("Block\n");
5. END

## block 获取外部变量

运行下面的代码
```
int main()
{
    int intValue = 1;
    void (^blk)(void) = ^{ printf("intValue = %d\n", intValue); };
    blk();
    return 0;
}

```
打印结果
```
intValue = 1
```

和第一段源码不同的是，这里多了个局部变量 intValue，而且还在 block 里面获取到了。

通过前一段对 block 源码的学习，我们已经了解到 block 的函数定义在 main() 函数之外，那它又是如何获取 main() 里面的局部变量呢？为了解开疑惑，我们再次用 clang 重写这段代码
```
struct __block_impl
{
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
struct __main_block_impl_0
{
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int intValue;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _intValue, int flags=0) : intValue(_intValue)
    {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself)
{
    int intValue = __cself->intValue; // bound by copy
    printf("intValue = %d\n", intValue);
}
static struct __main_block_desc_0
{
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main()
{
    int intValue = 1;
    void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, intValue);
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    return 0;
}

```

原来 block 通过参数值传递获取到 `intValue` 变量，通过函数
>__main_block_impl_0 (void *fp, struct __main_block_desc_0 *desc, int _intValue, int flags=0) : intValue(_intValue)

保存到 `__main_block_impl_0` 结构体的同名变量 `intValue`，通过代码 `int intValue = __cself->intValue`; 取出 `intValue`，打印出来。

>构造函数`__main_block_impl_0`冒号后的表达式`intValue(_intValue)`的意思是，用 `_intValue`初始化结构体成员变量 `intValue`。
有四种情况下应该使用初始化表达式来初始化成员：
1. 初始化const成员
2. 初始化引用成员
3. 当调用基类的构造函数，而它拥有一组参数时
4. 当调用成员类的构造函数，而它拥有一组参数时
[参考：C++类成员冒号初始化以及构造函数内赋值](http://blog.csdn.net/zj510/article/details/8135556)

至此，我们已经了解了block 的实现，以及获取外部变量的原理。但是，我们还不能在 block 内修改 intValue 变量。如果你有心试下，在 block 内部修改 `intValue` 的值，会报编译错误
>Variable is not assignable(missing __block type specifier)
