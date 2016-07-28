## OC中load方法和initialize方法的异同

**对于load方法，官方的文档说明如下：**

Invoked whenever a class or category is added to the Objective-C runtime; implement this method to perform class-specific behavior upon loading. The load message is sent to classes and categories that are both dynamically loaded and statically linked, but only if the newly loaded class or category implements a method that can respond.

The order of initialization is as follows:

- All initializers in any framework you link to.
- All +load methods in your image.
- All C++ static initializers and C/C++ **attribute**(constructor) functions in your image.
- All initializers in frameworks that link to you. In addition:

- A class's +load method is called after all of its superclasses' +load methods.

- A category +load method is called after the class's own +load method. In a custom implementation of load you can therefore safely message other unrelated classes from the same image, but any load methods implemented by those classes may not have run yet.

文档也说清楚了，**对于load方法，只要文件被引用就会被调用。load方法调用顺序是父类的load方法 优先调用于子类的load方法，而本类的load方法优先于category调用**。 **对于+initialize方法，官方的文档说明如下：** Initializes the class before it receives its first message.

The runtime sends initialize to each class in a program just before the class, or any class that inherits from it, is sent its first message from within the program. The runtime sends the initialize message to classes in a thread-safe manner. Superclasses receive this message before their subclasses. The superclass implementation may be called multiple times if subclasses do not implement initialize--the runtime will call the inherited implementation--or if subclasses explicitly call [super initialize]. If you want to protect yourself from being run multiple times, you can structure your implementation along these lines:

```objective-c
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

Because initialize is called in a **thread-safe** manner and the order of initialize being called on different classes is not guaranteed, it's important to do the minimum amount of work necessary in initialize methods.

Specifically, any code that takes locks that might be required by other classes in their initialize methods is liable to lead to **deadlocks**.

Therefore you should not rely on initialize for complex initialization, and should instead limit it to straightforward, class local initialization. initialize is invoked **only once per class**. If you want to perform independent initialization for the class and for categories of the class, you should implement load methods.

**文档也很明确的说明了：**文件被引用并不代表initialize就会被调用，只有类或者子类中第一次有函数调用时， 都会调用initialize。initialize是线程安全的，我们不能在initialize方法中加锁，这有可能导致死锁。 我们也不应该在函数中实现复杂的代码。initialize只会被调用一次。

**+load和+initialize共同点：**

- 在不考虑开发者主动使用的情况下，系统最多会调用一次
- 如果父类和子类都被调用，父类的调用一定在子类之前
- 这两个方法不适合做复杂的操作，应该是足够简单
- 在使用时都不要过重地依赖于这两个方法，除非真正必要。在使用时一定要注意防止死锁！
- 都不需要调用[super load]、[super initialize]

**+load和+initialize不同点：**

- load方法没有自动释放池，如果做数据处理，需要释放内存，则开发者得自己添加autoreleasepool来管理内存的释放。
- 和load不同，即使子类不实现initialize方法，也会把父类的实现继承过来调用一遍。注意的是在此之前， 父类的方法已经被执行过一次了，同样不需要super调用。

## NSObject的load和initialize方法

**load和initialize的共同特点** 在不考虑开发者主动使用的情况下，系统最多会调用一次 如果父类和子类都被调用，父类的调用一定在子类之前 都是为了应用运行提前创建合适的运行环境 在使用时都不要过重地依赖于这两个方法，除非真正必要

**load和initialize的区别** **load方法**

调用时机比较早，运行环境有不确定因素。具体说来，在iOS上通常就是App启动时进行加载， 但当load调用的时候，并不能保证所有类都加载完成且可用，必要时还要自己负责做auto release处理。 对于有依赖关系的两个库中，被依赖的类的load会优先调用。但在一个库之内，调用顺序是不确定的。

对于一个类而言，没有load方法实现就不会调用，不会考虑对NSObject的继承。

一个类的load方法不用写明[super load]，父类就会收到调用，并且在子类之前。

Category的load也会收到调用，但顺序上在主类的load调用之后。

不会直接触发initialize的调用。

**initialize方法相关要点**

initialize的自然调用是在第一次主动使用当前类的时候。

在initialize方法收到调用时，运行环境基本健全。

initialize的运行过程中是能保证线程安全的。

和load不同，即使子类不实现initialize方法，会把父类的实现继承过来调用一遍。注意的是在此之前， 父类的方法已经被执行过一次了，同样不需要super调用。

由于initialize的这些特点，使得其应用比load要略微广泛一些。可用来做一些初始化工作， 或者单例模式的一种实现方案。
