# 圆角的问题(Offscreen rendering的问题)
[ReviewCode](http://www.reviewcode.cn/article.html?reviewId=7](#)
## 离屏渲染是什么？
OpenGL中，GPU屏幕渲染有以下两种方式： On-Screen Rendering 意为当前屏幕渲染，指的是GPU的渲染操作是在当前用于显示的屏幕缓冲区中进行。 Off-Screen Rendering 意为离屏渲染，指的是GPU在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。
相比于当前屏幕渲染，离屏渲染的代价是很高的，主要体现在两个方面： 创建新缓冲区 要想进行离屏渲染，首先要创建一个新的缓冲区。 上下文切换 离屏渲染的整个过程，需要多次切换上下文环境：先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上有需要将上下文环境从离屏切换到当前屏幕。而上下文环境的切换是要付出很大代价的。 哪些行为会导致Offscreen rendering?
- custom drawRect: (any, even if you simply fill the background with color)
- CALayer corner radius
- CALayer shadow
- CALayer mask
- any custom drawing using CGContext
## 如何解决？
圆角使用UIImageView来处理。
简单来说，底层铺一个UIImageView,然后用GraphicsContext生成一张带圆角的图。
	@implementation UIImage (RoundedCorner)
```
	- (UIImage *)yal_imageWithRoundedCornersAndSize:(CGSize)sizeToFit andCornerRadius:(CGFloat)radius
	{
	    CGRect rect = (CGRect){0.f, 0.f, sizeToFit};

	    UIGraphicsBeginImageContextWithOptions(sizeToFit, NO, UIScreen.mainScreen.scale);
	    CGContextAddPath(UIGraphicsGetCurrentContext(),
	                     [UIBezierPath bezierPathWithRoundedRect:rect cornerRadius:radius].CGPath);
	    CGContextClip(UIGraphicsGetCurrentContext());

	    [self drawInRect:rect];
	    UIImage *output = UIGraphicsGetImageFromCurrentImageContext();

	    UIGraphicsEndImageContext();

	    return output;
	}

	@end
```
设置圆角的方法也有很多种。
1.cornerRadius
2.UIBezierPath
	- (void)drawRect:(CGRect)rect {
	  CGRect bounds = self.bounds;
	  [[UIBezierPath bezierPathWithRoundedRect:rect cornerRadius:8.0] addClip];

	  [self.image drawInRect:bounds];
	}
3.maskLayer(CAShapeLayer) 相关链接[[Applying Rounded Corners] http://articles.cocoahope.com/blog/2013/03/06/applying-rounded-corners ](#)[[Mastering UIKit Performance] https://yalantis.com/blog/mastering-uikit-performance/ ](#)
