## iOS Life Cycle

**应用程序的状态** **Not running未运行**：程序没启动。 **Inactive未激活**：程序在前台运行，不过没有接收到事件。在没有事件处理情况下程序通常停留在这个状态。 **Active激活**：程序在前台运行而且接收到了事件。这也是前台的一个正常的模式。 **Backgroud后台**：程序在后台而且能执行代码，大多数程序进入这个状态后会在在这个状态上停留一会。 时间到之后会进入挂起状态(Suspended)。有的程序经过特殊的请求后可以长期处于Backgroud状态。 **Suspended挂起**：程序在后台不能执行代码。系统会自动把程序变成这个状态而且不会发出通知。 当挂起时，程序还是停留在内存中的，当系统内存低时，系统就把挂起的程序清除掉，为前台程序提供更多的内存。

iOS的入口在main.m文件：

```objc
int main(int argc, char *argv[])
{
@autoreleasepool {
    return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
}
}
```

main函数的两个参数，iOS中没有用到，包括这两个参数是为了与标准ANSI C保持一致。 UIApplicationMain函数， 前两个和main函数一样，重点是后两个。

后两个参数分别表示程序的主要类(principal class)和代理类(delegate class)。 如果主要类(principal class)为nil，将从Info.plist中获取，如果Info.plist中不存在对应的key， 则默认为UIApplication；如果代理类(delegate class)将在新建工程时创建。 根据UIApplicationMain函数，程序将进入AppDelegate.m，这个文件是xcode新建工程时自动生成的。 下面看一下AppDelegate.m文件，这个关乎着应用程序的生命周期。

1. application didFinishLaunchingWithOptions：当应用程序启动时执行，应用程序启动入口， 只在应用程序启动时执行一次。若用户直接启动，lauchOptions内无数据,若通过其他方式启动应用， lauchOptions包含对应方式的内容。
2. applicationWillResignActive：在应用程序将要由活动状态切换到非活动状态时候， 要执行的委托调用，如 按下 home 按钮，返回主屏幕，或全屏之间切换应用程序等。
3. applicationDidEnterBackground：在应用程序已进入后台程序时，要执行的委托调用。
4. applicationWillEnterForeground：在应用程序将要进入前台时(被激活)，要执行的委托调用， 刚好与applicationWillResignActive 方法相对应。
5. applicationDidBecomeActive：在应用程序已被激活后，要执行的委托调用， 刚好与applicationDidEnterBackground 方法相对应。
6. applicationWillTerminate：在应用程序要完全推出的时候，要执行的委托调用， 这个需要要设置UIApplicationExitsOnSuspend的键值。

**初次启动**： iOS_didFinishLaunchingWithOptions iOS_applicationDidBecomeActive **按下home键**： iOS_applicationWillResignActive iOS_applicationDidEnterBackground **点击程序图标进入**： iOS_applicationWillEnterForeground iOS_applicationDidBecomeActive

当应用程序进入后台时,应该保存用户数据或状态信息，所有没写到磁盘的文件或信息，在进入后台时， 最后都写到磁盘去，因为程序可能在后台被杀死。释放尽可能释放的内存。

```objc
- (void)applicationDidEnterBackground:(UIApplication *)application
```

方法有大概5秒的时间让你完成这些任务。如果超过时间还有未完成的任务，你的程序就会被终止而且从内存中清除。

如果还需要长时间的运行任务，可以在该方法中调用

```objc
[application beginBackgroundTaskWithExpirationHandler:^{

    NSLog(@"begin Background Task With Expiration Handler");

}];
```

**程序终止** 程序只要符合以下情况之一，只要进入后台或挂起状态就会终止：

1. OS4.0以前的系统
2. app是基于iOS4.0之前系统开发的。
3. 设备不支持多任务
4. 在Info.plist文件中，程序包含了 UIApplicationExitsOnSuspend 键。

系统常常是为其他app启动时由于内存不足而回收内存最后需要终止应用程序， 但有时也会是由于app很长时间才响应而终止。如果app当时运行在后台并且没有暂停， 系统会在应用程序终止之前调用app的代理的方法 - (void)applicationWillTerminate:(UIApplication *)application， 这样可以让你可以做一些清理工作。你可以保存一些数据或app的状态。这个方法也有5秒钟的限制。 超时后方法会返回程序从内存中清除。

注意：用户可以手工关闭应用程序。

## 简述视图控件器的生命周期。

loadView 尽管不直接调用该方法，如多手动创建自己的视图，那么应该覆盖这个方法并将它们赋值给试图控制器的 view 属性。

viewDidLoad 只有在视图控制器将其视图载入到内存之后才调用该方法，这是执行任何其他初始化操作的入口。

viewDidUnload 当试图控制器从内存释放自己的方法的时候调用，用于清楚那些可能已经在试图控制器中创建的对象。

viewVillAppear 当试图将要添加到窗口中并且还不可见的时候或者上层视图移出图层后本视图变成顶级视图时调用该方法， 用于执行诸如改变视图方向等的操作。实现该方法时确保调用 [super viewWillAppear:

viewDidAppear 当视图添加到窗口中以后或者上层视图移出图层后本视图变成顶级视图时调用， 用于放置那些需要在视图显示后执行的代码。确保调用 [super viewDidAppear：] 。

## 请简要说明viewDidLoad和viewDidUnload何时调用

viewDidLoad在view从nib文件初始化时调用，loadView在controller的view为nil时调用。 此方法在编程实现view时调用，view控制器默认会注册memory warning notification， 当view controller的任何view没有用的时候，viewDidUnload会被调用，在这里实现将retain的view release， 如果是retain的IBOutlet view 属性则不要在这里release，IBOutlet会负责release 。

## 如果你需要对view做一些其他的定制操作，在viewDidLoad里面去做。

根据上面的文档可以知道，有两种情况：

1. 如果你用了nib文件，重载这个方法就没有太大意义。因为loadView的作用就是加载nib。 如果你重载了这个方法不调用super，那么nib文件就不会被加载。如果调用了super，那么view已经加载完了， 你需要做的其他事情在viewDidLoad里面做更合适。

2. 如果你没有用nib，这个方法默认就是创建一个空的view对象。如果你想自己控制view对象的创建， 例如创建一个特殊尺寸的view，那么可以重载这个方法，自己创建一个UIView对象，然后指定 self.view = myView; 但这种情况也没有必要调用super，因为反正你也不需要在super方法里面创建的view对象。如果调用了super， 那么就是浪费了一些资源而已
