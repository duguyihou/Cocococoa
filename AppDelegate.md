# AppDelegate
>AppDelegate实际上是实现了UIApplicationDelegate协议的类。UIApplicationDelegate协议定义了很多和Application状态、消息相关的方法。

在创建project的时候，Xcode会自动生成一个AppDelegate类,并在程序运行的时候创建AppDelegate对象（方式是通过`UIApplicationMain`的方法，在main.m文件中，这个就是App的入口函数）。这样在App运行过程中，就会调用AppDelegate中的方法。比如：程序运行起来时，会调用`applicationDidFinishLaunchingWithOptions:`的方法，可以在该方法中填写代码，一般来说，如果创建的是view-based-application的话。这里会有如下代码：
```
[self.window addSubView: viewController.view];
[self.window makeKeyAndVisible];
```
在UIApplication接收到系统事件和生命周期事件时，会把相应的事件传递给UIApplicationDelegate进行处理，生命周期函数大都是可选的,但为了应用程序的健壮性应该实现它们。

* UIApplication:一般是指运行中的应用程序，主要工作是处理用户事件，它会引起一个队列，所有的用户事件都放入到队列中,逐个被处理掉，在处理的时候，它会发送事件到一个合适的处理事件的目标控件，此外UIApplication还维护在本应用程序中打开的Window列表（UIWindow实例），这样它就可以接触应用程序中的任何一个UIView对象；
* AppDelegate：负责为另外一个对象处理特定事件的类，比如我们load完一个页面的时候，委托就帮助UIApplication完成`didFinishLaunchingWithOptions`动作，相应地在这个方法里面执行对应地action；
* 委托是给一个对象提供机会对另一个对象中的变化作出反映或者影响另一个对象的行为，通常包括3种动词：`should`、`will`、`did`.
