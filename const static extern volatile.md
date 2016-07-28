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