# iOS Events and Responder Chain

## 事件类型（Event Type）

- 触控事件（UIEventTypeTouches）：单点、多点触控以及各种手势操作；
- 传感器事件（UIEventTypeMotion）：重力、加速度传感器等；
- 远程控制事件（UIEventTypeRemoteControl）：远程遥控iOS设备多媒体播放等；

## 响应者对象（Responder Object）

esponder object 是能够响应并处理事件的对象，是构成响应链和事件传递链的节点。

举个栗子，当手指去触摸屏幕上 UIView 的实例对象 aView。触摸的动作会产生一个触摸事件 UIEventTypeTouches，而接收触摸事件的对象 aView，就是一个 responder object。

一个事件有可能被多个 responder 接收，第一个接收事件的 responder 就叫 first responder。

在 iOS 中，几乎所有类都是 responder，比如 UIWindow、UIView、UIControl、UIControllers 等，而这些类都有一个共同的父类 ---- UIResponder。UIResponder 声明了用于处理事件的接口，并定义了默认的行为。

Responder 继承链如下：

![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note66142_1.png)

## 响应者链（responder chain）

当一个事件发生时，如果 first responder 不处理，事件就会继续往下传递，被下个 responder 接收，如果下个 responder 也不处理，又会被下下个 responder 接收...... 直到一个 responder 处理了事件或者没有 responder 了。这些 responder 按照传递次序连接起来的链条就构成了响应者链。

iOS 上的响应者链：

![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note66142_2.png)

由于不同的 app 内的布局和层次结构的不同，响应顺序也会有所不同，但事件的传递路径会遵守基本规则。从图中可以看到，响应者链有以下特点：

- 响应者链通常是由 initial view 开始；
- UIView 的 nextResponder 它的 superview；如果 UIView 已经是其所在的 UIViewController 的 top view，那么 UIView 的 nextResponder 就是 UIViewController；
- UIViewController 如果有 Super ViewController，那么它的 nextResponder 为其 Super ViewController 最表层的 View；如果没有，那么它的 nextResponder 就是 UIWindow；
- UIWindow 的 contentView 指向 UIApplication，将其作为 nextResponder；
- UIApplication 是一个响应者链的终点，它的 nextResponder 指向nil，整个 responder chain 结束。

需要说明是，如果当前的 responder 不处理事件，并希望将其传递给 nextResponder 时，需要手动编写代码，才会继续往下传递，否则事件会被废弃。

```objc
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{  
    // 将事件传递给 nextResponder
    id theNextResponder = [self nextResponder];
    [theNextResponder touchesBegan:touches withEvent:event];
}
```

--------------------------------------------------------------------------------

## Hit-Test View 与 Hit-Testing

当用户与 iPhone 的触摸屏产生互动时，硬件就会探测到物理接触并且通知操作系统。操作系统就会创建相应的事件，并将其传递给当前正在运行的应用程序的事件队列。然后这个事件会被事件循环传递给优先响应对象，即 Hit-Test View 。

Hit-Test View 就是事件被触发时和用户交互的对象，寻找 Hit-Test View 的过程就叫做 Hit-Testing。

首先假设现在有如下这么一个 UI 布局，一种有 A B C D E 五个视图。

![](http://7xs5iw.com1.z0.glb.clouddn.com/image_note66142_3.png)

假设用户触摸了上图的 View E 区域，那么 iOS 将会按下面的顺序反复检测 subview 来寻找 Hit-Test View

1. 触摸区域在视图 A 内，所以检测视图 A 的 subview B 和 C；
2. 触摸区域不在视图 B 内，但是在视图 C 内，所以检查视图 C 的 subview D 和 E；
3. 触摸区域不在视图 D 内，在视图 E 中；

视图 E 在整个视图体系中是 lowest view，所以视图 E 就是 Hit-Test View 。

> 总结：所以关于事件的链有两条：事件的响应链；Hit-Testing 时事件的传递链。

> - 响应链：由离用户最近的view向系统传递。 initial view –> super view –> .....–> view controller –> window –> Application –> AppDelegate
> - Hit-Testing 链：由系统向离用户最近的view传递。 UIKit –> active app's event queue –> window –> root view –>......–>lowest view

## ViewController如何响应Touch事件？

ViewController类都继承自UIResponder类，它们也可以响应Touch事件。ViewController何时响应Touch事件呢？iOS中Touch事件响应是根据响应链来进行处理的，Touch事件会逐个发向各个节点直到这个节点响应这个事件。

而根据SDK记载： ViewController在响应链中的位于ViewController的view和它的superView之间的，因此只有在Touch在ViewController的view内部，而且viewController的view不响应Touch，ViewController才接受到Touch事件。

- [iOS事件分发机制（一） hit-Testing](http://suenblog.duapp.com/blog/100031/iOS事件分发机制（一）%20hit-Testing#sidebar)
- [iOS事件分发机制（二）The Responder Chain](http://suenblog.duapp.com/blog/100032/iOS事件分发机制（二）The%20Responder%20Chain)
- [iOS事件响应链中Hit-Test View的应用](http://www.jianshu.com/p/d8512dff2b3e)

参考文档：

- [iOS Developer Library: Event Handling Guide for iOS](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009541-CH1-SW1)
- [iOS Developer Library: UIKit Framework Reference](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIKit_Framework/)
- [Cocoa Application Competencies for iOS: Responder Object](https://developer.apple.com/library/ios/documentation/General/Conceptual/Devpedia-CocoaApp/Responder.html)
- [iOS Events and Responder Chain](https://www.zybuluo.com/MicroCai/note/66142)
