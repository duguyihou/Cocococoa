# Respond Chain
## 什么是事件响应链，点击屏幕时是如何互动的，事件的传递。

对于IOS设备用户来说，他们操作设备的方式主要有三种：触摸屏幕、晃动设备、通过遥控设施控制设备。 对应的事件类型有以下三种：

1. 触屏事件（Touch Event）
2. 运动事件（Motion Event） 3.远端控制事件（Remote-Control Event）

**响应者链（Responder Chain）** 响应者对象（Responder Object），指的是有响应和处理事件能力的对象。响应者链就是由一系列的响应者对象构成的一个层次结构。

UIResponder是所有响应对象的基类，在UIResponder类中定义了处理上述各种事件的接口。 我们熟悉的UIApplication、 UIViewController、UIWindow和所有继承自UIView的UIKit类 都直接或间接的继承自UIResponder，所以它们的实例都是可以构成响应者链的响应者对象。

响应者链有以下特点： 1、响应者链通常是由视图（UIView）构成的； 2、一个视图的下一个响应者是它视图控制器（UIViewController）（如果有的话），然后再转给它的父视图（Super View）； 3、视图控制器（如果有的话）的下一个响应者为其管理的视图的父视图； 4、单例的窗口（UIWindow）的内容视图将指向窗口本身作为它的下一个响应者 需要指出的是，Cocoa Touch应用不像Cocoa应用，它只有一个UIWindow对象，因此整个响应者链要简单一点； 5、单例的应用（UIApplication）是一个响应者链的终点，它的下一个响应者指向nil，以结束整个循环。

**点击屏幕时是如何互动的** iOS系统检测到手指触摸(Touch)操作时会将其打包成一个UIEvent对象，并放入当前活动Application的事件队列， 单例的UIApplication会从事件队列中取出触摸事件并传递给单例的UIWindow来处理，UIWindow对象首 先会使用hitTest:withEvent:方法寻找此次Touch操作初始点所在的视图(View)， 即需要将触摸事件传递给其处理的视图，这个过程称之为hit-test view。

UIWindow实例对象会首先在它的内容视图上调用hitTest:withEvent:，此方法会在其视图层级结构中的 每个视图上调用pointInside:withEvent:（该方法用来判断点击事件发生的位置是否处于当前视图范围内， 以确定用户是不是点击了当前视图），如果pointInside:withEvent:返回YES，则继续逐级调用， 直到找到touch操作发生的位置，这个视图也就是要找的hit-test view。

hitTest:withEvent:方法的处理流程如下:首先调用当前视图的pointInside:withEvent:方法判断 触摸点是否在当前视图内；若返回NO,则hitTest:withEvent:返回nil;若返回YES,则向当前视图的 所有子视图(subviews)发送hitTest:withEvent:消息，所有子视图的遍历顺序是从最顶层视图一直到到最底层视图， 即从subviews数组的末尾向前遍历，直到有子视图返回非空对象或者全部子视图遍历完毕； 若第一次有子视图返回非空对象，则hitTest:withEvent:方法返回此对象，处理结束；如所有子视图都返回非， 则hitTest:withEvent:方法返回自身(self)。

事件的传递和响应分两个链：

传递链：由系统向离用户最近的view传递。UIKit –> active app's event queue –> window –> root view –>......–>lowest view 响应链：由离用户最近的view向系统传递。initial view –> super view –> .....–> view controller –> window –> Application
