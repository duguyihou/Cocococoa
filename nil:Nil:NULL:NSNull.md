## nil/Nil/NULL/NSNull

1. NULL 声明位置在stddef.h文件 对于普通的iOS开发者来说，通常NULL的定义就是：`# define NULL ((void*)0)` 因此，NULL本质上是：(void_)0 NULL表示C指针为空`charchar_ string = NULL;`
2. nil<br>
  声明在objc.h文件 对于普通iOS开发者来说，nil的定义形式为：`# define nil __DARWIN_NULL` 就是说nil最终是DARWIN_NULL的宏定义，DARWIN_NULL是定义在_types.h中的宏。`#define __DARWIN_NULL ((void *)0)` 也就是说，nil本质上是：(void *)0 用于表示指向Objective-C中对象的指针为空

  ```objc
  NSString *string = nil;  
  id anyObject = nil;
  ```

3. Nil 声明位置在objc.h文件 和上面讲到的nil一样，Nil本质上也是：(void *)0 用于表示Objective-C类（Class）类型的变量值为空

  ```objc
  Class anyClass = Nil;
  ```

4. NSNull 声明位置在NSNull.h文件 定义 ```objc @interface NSNull : NSObject

  <nscopying, nssecurecoding="">
  </nscopying,>

5. (NSNull *)null;<br>
  @end ``` 从定义中可以看出，NSNull是一个Objective-C类，只不过这个类相当特殊，因为它表示的是空值， 即什么都不存。它也只有一个单例方法+[NSUll null]。该类通常用于在集合对象中保存一个空的占位对象。

我们通常初始化NSArray对象的形式如下：

```objc
NSArray *arr = [NSArray arrayWithObjects:@"wang",@"zz",nil];
```

当NSArray里遇到nil时，就说明这个数组对象的元素截止了，即NSArray只关注nil之前的对象， nil之后的对象会被抛弃。比如下面的写法：

```objc
NSArray *arr = [NSArray arrayWithObjects:@"wang",@"zz",nil,@"foogry"];
```

这是NSArray中只会保存wang和zz两个字符串，foogry字符串会被抛弃。 这种情况，就可以使用NSNull实现：

```objc
NSArray *arr = [NSArray arrayWithObjects:@"wang",@"zz",[NSNull null],@"foogry"];
```

从前面的介绍可以看出，不管是NULL、nil还是Nil，它们本质上都是一样的，都是(void *)0，只是写法不同。 这样做的意义是为了区分不同的数据类型，比如你一看到用到了NULL就知道这是个C指针， 看到nil就知道这是个Objective-C对象，看到Nil就知道这是个Class类型的数据。

注意：NULL是C指针指向的值为空；nil是OC对象指针自己本身为空，不是值为空
