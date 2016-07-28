## UIWindow

**UIWindow** UIWindow继承自UIView，UIWindow是一种特殊的UIView，通常在一个程序中只会有一个UIWindow， 但可以手动创建多个UIWindow，同时加到程序里面。即使有多个UIWindow对象， 也只有一个UIWindow可以接受到用户的触屏事件（即主窗口）。

iOS程序启动完毕后，先创建Application，再创建它的代理，之后创建UIWindow（创建的第一个对象是UIApplication， 接着创建控制器的view，最后将控制器的view添加到UIWindow上，于是控制器的view就显示在屏幕上了。

一个iOS程序之所以能显示到屏幕上，完全是因为它有UIWindow。也就说，没有UIWindow，就看不见任何UI界面。 **主窗口和次窗口**

```objc
[self.window makekeyandvisible]; 让窗口成为主窗口，并且显示出来。有这个方法，才能把信息显示到屏幕上。
```

因为Window有makekeyandvisible这个方法，可以让这个Window凭空的显示出来，而其他的view没有这个方法， 所以它只能依赖于Window，Window显示出来后，view才依附在Window上显示出来。

```objc
[self.window make keywindow];//让uiwindow成为主窗口，但不显示。
```

次窗口，需要定义一个Window属性来保存变量。 window的属性定义为strong，就是为了让其不销毁， 一个应用程序只能有一个主窗口。只有主窗口才能响应键盘的输入事件，如果不能输入内容， 可以查看是否是显示在主窗口上，不在主窗口上的不能响应。 **WindowLevel** UIWindow有三个层级，分别是Normal，StatusBar，Alert。Normal级别是最低的，StatusBar处于中等水平， Alert级别最高。而通常我们的程序的界面都是处于Normal这个级别上的，系统顶部的状态栏应该是处于StatusBar级别 ，UIActionSheet和UIAlertView这些通常都是用来中断正常流程，提醒用户等操作，因此位于Alert级别。

根据window显示级别优先的原则，级别高的会显示在上面，级别低的在下面，我们程序正常显示的view位于最底层。

当Level层级相同的时候，只有第一个设置为KeyWindow的显示出来，后面同级的再设置KeyWindow也不会显示。 UIWindow在显示的时候是不管KeyWindow是谁，都是Level优先的，即Level最高的始终显示在最前面。

**UIWindow是显示的起点**

1. UIWindow对象是所有UIView的根，管理和协调应用程序的显示。
2. UIViewController对象负责管理所有UIView的层次结构，并响应设备的方向变化。
3. UIView对象定义了一个屏幕上的一个矩形区域，同时处理该区域的绘制和触屏事件。 可以在这个区域内绘制图形和文字，还可以接收用户的操作。一个UIView的实例可以包含和管理若干个子UIView。

**UIWindow在程序中的作用**

1. 作为容器，包含app所要显示的所有视图
2. 传递触摸消息到程序中view和其他对象
3. 与UIViewController协同工作，方便完成设备方向旋转的支持

**storyboard项目中的创建过程** 当用户点击应用程序图标的时候，先执行Main函数，执行UIApplicationMain(), 根据其第三个和第四个参数创建Application，创建代理，并且把代理设置给application （看项目配置文件info.plist里面的storyboard的name，根据这个name找到对应的storyboard）， 开启一个事件循环，当程序加载完毕，他会调用代理的didFinishLaunchingWithOptions:方法。 在调用didFinishLaunchingWithOptions:方法之前，会加载storyboard，在加载的时候创建一个window， 接下来会创建箭头所指向的控制器，把该控制器设置为UIWindow的根控制器，接下来再将window显示出来， 即看到了运行后显示的界面。

**rootViewController和addSubview的不同** 创建一个控制器，把view添加到uiwindow上面有两种方式

1. rootViewController rootViewController是UIWindow的一个遍历方法，通过设置该属性为要添加view对应的ViewController， UIWindow将会自动将其view添加到当前window中，同时负责ViewController和view的生命周期的维护， 防止其过早释放

2. addSubview 直接将view通过addSubview方式添加到window中，程序负责维护view的生命周期以及刷新， 但是并不会为去理会view对应的ViewController，因此采用这种方法将view添加到window以后， 我们还要保持view对应的ViewController的有效性，不能过早释放。

提示：不通过控制器的view也可以做开发，但是在实际开发中，不要这么做，不要直接把view添加到UIWindow上面去。 因为，难以管理。以后的开发中，建议使用rootViewController。

**UIView有关图层布局的方法** 一个 UIView 里面可以包含许多的 Subview（其他的 UIView），而这些 Subview 彼此之间是有层级关系的。

1. 新增Subview

  ```objc
  [UIView addSubview:Subview];     //替UIView增加一个Subview
  ```

2. 移动图层 在UIView中将Subview往前或是往后移动一个图层，往前移动会覆盖住较后层的 Subview，而往后移动则会被较上层的Subview所覆盖。

  ```objc
  UIView bringSubviewToFront:Subview];       //将Subview往前移动一个图层（与它的前一个图层对调位置）
  [UIView sendSubviewToBack:Subview];      //将Subview往后移动一个图层（与它的后一个图层对调位置）
  ```

3. 交换两个Subview彼此的图层层级

  ```objc
  [UIView exchangeSubviewAtIndex:indexA withSubviewAtIndex:indexB];    //交换两个图层
  ```

4. 取得子视图在UIView中的索引值（Index）

  ```objc
  NSInteger index = [[UIView subviews] indexOfObject:Subview名称];       //取得Index
  ```
