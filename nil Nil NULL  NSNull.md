##不管是NULL、nil还是Nil，它们本质上都是一样的，都是`(void *)0`，只是写法不同。这样做的意义是为了区分不同的数据类型，比如你一看到用到了NULL就知道这是个C指针，看到nil就知道这是个Objective-C对象，看到Nil就知道这是个Class类型的数据。

>注意：NULL是C指针指向的值为空；nil是OC对象指针自己本身为空，不是值为空


[StackOverFlow](http://stackoverflow.com/questions/32452889/difference-between-nullable-nullable-and-nullable-in-objective-c/33682230#33682230)

一个下划线首字母大写的是 clang 标准，剩下两种是 OC 标准，不带下划线的可以修饰 property。

