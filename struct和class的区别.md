## struct和class的区别
swift中，class是引用类型，struct是值类型。值类型在传递和赋值时将进行复制，而引用类型则只会使用引用对象的一个"指向"。所以他们两者之间的区别就是两个类型的区别。

class有这几个功能struct没有的：

* class可以继承，这样子类可以使用父类的特性和方法
* 类型转换可以在runtime的时候检查和解释一个实例的类型
* 可以用deinit来释放资源
* 一个类可以被多次引用

struct也有这样几个优势：
* 结构较小，适用于复制操作，相比于一个class的实例被多次引用更加安全。
* 无须担心内存memory leak或者多线程冲突问题

array在swift中是用struct实现的。Apple重写过一次array，然后复制就是深度拷贝了。要是多次拷贝且不进行修改的话，所有arrays指向的都是同一个物理地址，只是指针移动，所以性能上还是不错的。当然要是修改的话，array就会重新拷贝一份，这个时候开销就有点大了。

引用猫神OneV的博客：
```
var arr = [0,0,0]
var newArr = arr
arr[0] = 1
//Check arr and newArr
arr //[1, 0, 0]
newArr // before beta3:[1, 0, 0], after beta3:[0, 0, 0]
```
> 所以可以猜测其实在背后 Array和 Dictionary的行为并不是像其他 struct 那样简单的在栈上分配，而是类似参照那样，通过栈上指向堆上位置的指针来实现的。而对于它的复制操作，也是在相对空间较为宽裕的堆上来完成的。当然，现在还无法（或者说很难）拿到最后的汇编码，所以这只是一个猜测而已。
补充：
**C**语言中，struct与的class的区别：
struct只是作为一种复杂数据类型定义，不能用于面向对象编程。

**C++**中，struct和class的区别：
对于成员访问权限以及继承方式，class中默认的是private的，而struct中则是public的。class还可以用于表示模板类型，struct则不行。

