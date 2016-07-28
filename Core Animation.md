## UIView的动画效果有那些?

UIViewAnimationOptionCurveEaseInOut UIViewAnimationOptionCurveEaseIn UIViewAnimationOptionCurveEaseOut UIViewAnimationOptionTransitionFlipFromLeft UIViewAnimationOptionTransitionFlipFromRight UIViewAnimationOptionTransitionCurlUp UIViewAnimationOptionTransitionCurlDown

## 动画有基本类型有哪几种；表视图有哪几种基本样式。

动画有两种基本类型：隐式动画和显式动画。

## Cocoa Touch提供了哪几种Core Animation过渡类型？

Cocoa Touch 提供了 4 种 Core Animation 过渡类型，分别为：交叉淡化、推挤、显示和覆盖。

## UIView与CLayer有什么区别？

1).UIView是iOS系统中界面元素的基础，所有的界面元素都继承自它。 它本身完全是由CoreAnimation来实现的 （Mac下似乎不是这样）。它真正的绘图部分， 是由一个叫CALayer（Core Animation Layer）的类来管理。 UIView本身，更像是一个CALayer的管理器， 访问它的跟绘图和跟坐标有关的属性，例如frame，bounds等 等，实际上内部都是在访问它所包含的CALayer的相关属性。

2).UIView有个layer属性，可以返回它的主CALayer实例，UIView有一个layerClass方法， 返回主layer所使用的 类，UIView的子类，可以通过重载这个方法，来让UIView使用不同的CALayer来显示，例如通过

```objc
- (class) layerClass {

         return ([CAEAGLLayer class]);
    }
```

使某个UIView的子类使用GL来进行绘制。

3).UIView 的 CALayer 类似 UIView 的子 View 树形结构，也可以向它的 layer 上添加子layer ， 来完成某些特殊的表示。即 CALayer 层是可以嵌套的。

```objc
grayCover = [[CALayer alloc] init];

    grayCover.backgroundColor = [[[UIColor blackColor] colorWithAlphaComponent:0.2] CGColor];

    [self.layer addSubLayer: grayCover];
```

在目标View上敷上一层黑色的透明薄膜。 4).UIView 的 layer 树形在系统内部，被维护着三份 copy 。

- 逻辑树，这里是代码可以操纵的；
- 动画树，是一个中间层，系统就在这一层上更改属性，进行各种渲染操作；
- 显示树，其内容就是当前正被显示在屏幕上得内容。

这三棵树的逻辑结构都是一样的，区别只有各自的属性。

5).动画的运作：对 UIView 的 subLayer （非主 Layer ）属性进行更改，系统将自动进行动画生成， 动画持续时间的缺省值似乎是 0.5 秒。

6).坐标系统： CALayer 的坐标系统比 UIView 多了一个 anchorPoint 属性，使用CGPoint 结构表示， 值域是 0~1 ，是个比例值。这个点是各种图形变换的坐标原点，同时会更改 layer 的 position 的位置， 它的缺省值是 {0.5,0.5} ，即在 layer 的中央。

7).渲染：当更新层，改变不能立即显示在屏幕上。当所有的层都准备好时，可以调用setNeedsDisplay 方法来重绘显示。

8).变换：要在一个层中添加一个 3D 或仿射变换，可以分别设置层的 transform 或affineTransform 属性。

9).变形： Quartz Core 的渲染能力，使二维图像可以被自由操纵，就好像是三维的。 图像可以在一个三维坐标系中以任意角度被旋转，缩放和倾斜。 CATransform3D 的一套方法提供了一些魔术般的变换效果。

## Quatrz 2D的绘图功能的三个核心概念是什么并简述其作用。

上下文：主要用于描述图形写入哪里； 路径：是在图层上绘制的内容； 状态：用于保存配置变换的值、填充和轮廓， alpha 值等。

## 解析XML文件有哪几种方式？

以 DOM 方式解析 XML 文件；以 SAX 方式解析 XML 文件；
