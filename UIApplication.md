**UIApplication**
上面我们使用了UIApplication的IdleTimerDisabled方法，下面就大概了解下UIApplication吧。

UIApplication，每个程序只能有一个，系统使用的是单例模式，用[UIApplication sharedApplication]来得到一个实例。 这个单例实例是在系统启动时由main函数里面的UIApplicationMain方法生成，实现的是UIApplicationDelegate的Protocol， 也就是AppDelegate的一个实例。每次通过[UIApplication sharedApplication]调用的就是它。 UIApplication保存一个UIWindow对象序列，用来快速恢复views。

UIApplication在程序里的作用很多，大致如下所示：

```objc
一、远程提醒，就是push notification注册；
二、可以连接到UIUndoManager；在Cocoa中使用NSUndoManager可以很方便的完成撤销操作。
NSUndoManager会记录下修改、撤销操作的消息。这个机制使用两个NSInvocation对象栈。当进行操作时，
控制器会添加一个该操作的逆操作的invocation到Undo栈中。当进行Undo操作时，
Undo操作的逆操作会倍添加到Redo栈中，就这样利用Undo和Redo两个堆栈巧妙的实现撤销操作。需要注意的是，
堆栈中存放的都是NSInvocation实例。
三、检查能否打开某个URL，并且打开URL；这个功能可以配合应用的自定义URL功能，来检测是否安装了某个应用。
使用的是[[UIApplication sharedApplication] canOpenURL:url]方法。如果返回YES，
可执行[[UIApplication sharedApplication] openURL:url];
四、注册Local Notification；
五、在后台运行以及从后台转为前台时的操作；
六、防止屏幕睡眠：即上面的[[UIApplication sharedApplication] setIdleTimerDisabled: YES];
七、手动调整status bar的位置和状态，如设置为竖屏、横屏等；
八、设置badge number，就是图标右上角的数字；
九、每当应用联网时，在状态栏上会显示联网小菊花。UIApplication可以设置是否出现。
UIApplication *app = [UIApplication sharedApplication];
app.networkActivityIndicatorVisible =!app.networkActivityIndicatorVisible;//转动
app.networkActivityIndicatorVisible = app.networkActivityIndicatorVisible;//不转动
```
UIUndoManager示例

```objc
- (void) one
{
position = position + 10;
[[undoManager prepareWithInvocationTarget:self] two];
[self showTheChangesToThePostion];
}

- (void) two
{
position = position - 10;
[[undoManager prepareWithInvocationTarget:self] one];
[self showTheChangesToThePostion];
}
```

prepareWithInvocationTarget:方法记录了target并返回UndoManager， 然后UndoManager重载了forwardInvocation方法，也就将two方法的Invocation添加到undo栈中了。
