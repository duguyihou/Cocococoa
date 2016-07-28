## iOS的锁屏和解锁

**idleTimer** idleTimer 是iOS内置的时间监测机制，当在一段时间内未操作即进入锁屏状态。但有些应用程序是不需要锁住屏幕的， 比如游戏，视频这类应用。 可以通过设置UIApplication的idleTimerDisabled属性来指定iOS是否锁屏。

```objc
// 禁用休闲时钟
[[UIApplication sharedApplication] setIdleTimerDisabled: YES];
```

也可以使用这种语法

```objc
[UIApplication sharedApplication].idleTimerDisabled = YES;
```

但是，这个命令只能禁用自动锁屏，如果点击了锁屏按钮，仍然会进入锁屏的。有一点例外的是， AVPlayer不用设置idleTimerDisabled=YES，也能屏幕常亮，播放完成后过一分钟就自动关闭屏幕。 有兴趣的可以自己尝试一下。

**锁屏和解锁通知** iPhone的锁屏监测分为两种方式监听：一种是程序在前台，另一种程序在后台。 程序在前台，这种比较简单。 直接使用Darwin层的通知就可以了：

> Darwin是由苹果电脑于2000年所释出的一个开放原始码操作系统。Darwin 是MacOSX 操作环境的操作系统成份。 苹果电脑于2000年把Darwin 释出给开放原始码社群。现在的Darwin皆可以在苹果电脑的PowerPC 架构和X86 架构下执行， 而后者的架构只有有限的驱动程序支援。

```objc
#import <notify.h>
#define NotificationLock CFSTR("com.apple.springboard.lockcomplete")
#define NotificationChange CFSTR("com.apple.springboard.lockstate")
#define NotificationPwdUI CFSTR("com.apple.springboard.hasBlankedScreen")

static void screenLockStateChanged(CFNotificationCenterRef center,void* observer,CFStringRef name,const void* object,CFDictionaryRef userInfo)
{
NSString* lockstate = (__bridge NSString*)name;
if ([lockstate isEqualToString:(__bridge  NSString*)NotificationLock]) {
    NSLog(@"locked.");
} else {
    NSLog(@"lock state changed.");
}
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), NULL, screenLockStateChanged, NotificationLock, NULL, CFNotificationSuspensionBehaviorDeliverImmediately);
CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), NULL, screenLockStateChanged, NotificationChange, NULL, CFNotificationSuspensionBehaviorDeliverImmediately);
return YES;
}
```

notify.h的具体内容可以移步开发文档。这种方法，程序在前台是可以拿到的，在后台情况下就无能为力了。

第二种是程序退后台后，这时再锁屏就收不到上面的那个通知了，需要另外一种方式, 以循环的方式一直来检测是否是锁屏状态， 会消耗性能并可能被苹果挂起，需要合理设置循环时间。

```objc
static void setScreenStateCb()
{
uint64_t locked;

__block int token = 0;
notify_register_dispatch("com.apple.springboard.lockstate",&token,dispatch_get_main_queue(),^(int t){
});
notify_get_state(token, &locked);
NSLog(@"%d",(int)locked);
}

- (void)applicationDidEnterBackground:(UIApplication *)application
{
while (YES) {
    setScreenStateCb();
    sleep(5); // 循环5s
}
}
```
