写C函数时，可以将其定义为inline函数。对于inline函数而言，其在编译时，会在调用处内联展开这个函数；在runtime，直接执行展开的代码，而避免了函数调用的开销。另外，如果函数实参是常量，这些常量的值也可在编译期被直接替换。

不过，对于inline声明的函数，Clang并不总是会做这种优化处理。如果将优化级别(Optimization Level)设置为`None[-O0]`时，编译器并不会尝试去优化代码，所以inline函数并不会内联展开，而是独立编译。
这种情况下，如果我们依然希望inline函数内联展开，则可以使用`__attribute__((always_inline))`属性来强制编译时内联展开函数。

- [An Inline Function is As Fast As a Macro](http://gcc.gnu.org/onlinedocs/gcc/Inline.html)
- [C语言inline详细讲解](http://www.cnblogs.com/cnmaizi/archive/2011/01/19/1939686.html)
- [Force a function to be inline in Clang/LLVM](http://stackoverflow.com/questions/25602813/force-a-function-to-be-inline-in-clang-llvm)