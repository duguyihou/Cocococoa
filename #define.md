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