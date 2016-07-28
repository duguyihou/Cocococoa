# Interview-desgin Patterns

## 对MVC和MVVM的理解 你还熟悉什么设计模式？

MVC是出现比较早的架构设计模式，而且到现在已经是很成熟了。出现MVVM的原因是MVC中的V越来越复杂， 于是才有人想要给V瘦身。

> 设计模式：并不是一种新技术，而是一种编码经验，使用比如java中的接口，iphone中的协议， 继承关系等基本手段，用比较成熟的逻辑去处理某一种类型的事情，总结为所谓设计模式。 面向对象编程中，java已经归纳了23种设计模式。

- mvc设计模式 ：模型，视图，控制器，可以将整个应用程序在思想上分成三大块，对应是的数据的存储或处理， 前台的显示，业务逻辑的控制。 Iphone本身的设计思想就是遵循mvc设计模式。其不属于23种设计模式范畴。

- 代理模式：代理模式给某一个对象提供一个代理对象，并由代理对象控制对源对象的引用. 比如一个工厂生产了产品，并不想直接卖给用户，而是搞了很多代理商，用户可以直接找代理商买东西， 代理商从工厂进货.常见的如QQ的自动回复就属于代理拦截，代理模式在iphone中得到广泛应用.

- 单例模式：说白了就是一个类不通过alloc方式创建对象，而是用一个静态方法返回这个类的对象。 系统只需要拥有一个的全局对象，这样有利于我们协调系统整体的行为， 比如想获得[UIApplication sharedApplication];任何地方调用都可以得到 UIApplication的对象， 这个对象是全局唯一的。

- 观察者模式： 当一个物体发生变化时，会通知所有观察这个物体的观察者让其做出反应。 实现起来无非就是把所有观察者的对象给这个物体，当这个物体的发生改变， 就会调用遍历所有观察者的对象调用观察者的方法从而达到通知观察者的目的。

- 工厂模式：

  ```objc
  public class Factory{
  public static Sample creator(int which){
  if (which==1)
  return new SampleA();
  else if (which==2)
  return new SampleB();
  }
  }
  ```

## 对于单例(Singleton)的理解

在objective-c中要实现一个单例类，至少需要做以下四个步骤：

- 为单例对象实现一个静态实例，并初始化，然后设置成nil。

- 实现一个实例构造方法检查上面声明的静态实例是否为nil，如果是则新建并返回一个本类的实例，

- 重写allocWithZone方法，用来保证其他人直接使用alloc和init试图获得一个新实力的时候不产生一个新实例，

- 适当实现allocWitheZone，copyWithZone，release和autorelease。

## 如何使用Xcode设计通用应用?

使用MVC模式设计应用，其中Model层完成脱离界面，即在Model层，其是可运行在任何设备上， 在controller层，根据iPhone与iPad(独有UISplitViewController)的不同特点选择不同的viewController对象。 在View层，可根据现实要求，来设计，其中以xib文件设计时，其设置其为universal。
