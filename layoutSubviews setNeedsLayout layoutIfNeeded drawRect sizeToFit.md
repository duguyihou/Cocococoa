## `layoutSubviews`

这个方法，默认没有做任何事情，需要子类进行重写。系统在很多时候会去调用这个方法：

1. 初始化不会触发`layoutSubviews`，但是如果设置了不为CGRectZero的frame的时候就会触发。
2. addSubview会触发`layoutSubviews`
3. 设置view的Frame会触发`layoutSubviews`，当然前提是frame的值设置前后发生了变化
4. 滚动一个UIScrollView会触发`layoutSubviews`
5. 旋转Screen会触发父UIView上的`layoutSubviews`事件
6. 改变一个UIView大小的时候也会触发父UIView上的`layoutSubviews`事件
7. 直接调用`setLayoutSubviews`

>在苹果的官方文档中强调: You should override this method only if the autoresizing behaviors of the subviews do not offer the behavior you want.layoutSubviews, 当我们在某个类的内部调整子视图位置时，需要调用。反过来的意思就是说：如果你想要在外部设置subviews的位置，就不要重写。

## `setNeedsLayout`
标记为需要重新布局，异步调用`layoutIfNeeded()`刷新布局，不立即刷新，但`layoutSubviews()`一定会被调用
## `layoutIfNeeded`
如果有需要刷新的标记，立即调用layoutSubviews进行布局

>如果要立即刷新，要先调用`setNeedsLayout`，把标记设为需要布局，然后马上调用`layoutIfNeeded()`，实现布局
在视图第一次显示之前，标记总是“需要刷新”的，可以直接调用`layoutIfNeeded`

## 在哪里创建autolayout?

* View中:直接在init方法里创建.
* ViewController中:直接在`viewDidLoad()`里创建.

如果用IB创建约束,在viewDidLoad里不能获取到某个view的正确frame,怎么办?
这个时候你需要在一个叫`viewDidLayoutSubviews()`里的方法里获取一个view的正确frame.
## drawRect
**重绘**
drawRect在以下情况下会被调用：

* 如果在UIView初始化时没有设置rect大小，将直接导致drawRect不被自动调用。drawRect调用是在`Controller->loadView`, `Controller->viewDidLoad` 两方法之后掉用的.所以不用担心在控制器中,这些View的drawRect就开始画了.这样可以在控制器中设置一些值给View(如果这些View draw的时候需要用到某些变量值).
* 该方法在调用sizeToFit后被调用，所以可以先调用sizeToFit计算出size。然后系统自动调用drawRect:方法。
* 通过设置contentMode属性值为UIViewContentModeRedraw。那么将在每次设置或更改frame的时候自动调用drawRect:。
* 直接调用setNeedsDisplay，或者setNeedsDisplayInRect:触发drawRect:，但是有个前提条件是rect不能为0。

## sizeToFit
* sizeToFit会自动调用sizeThatFits方法
* sizeToFit不应该在子类中被重写，应该重写sizeThatFits
* sizeThatFits传入的参数是receiver当前的size，返回一个适合的size
* sizeToFit可以被手动直接调用sizeToFit和sizeThatFits方法都没有递归，对subviews也不负责，只负责自己


## 重载Autolayout的问题
```
	- (void)refreshConstraints {
	    [self.view setNeedsUpdateConstraints];
	    [self.view updateConstraintsIfNeeded];
	    [self.view layoutIfNeeded];
	    [self.view setNeedsDisplay];
	}
```
> `setNeedsUpdateConstraints` 确保在未来的某个阶段`updateConstraintsIfNeeded`这个方法会调用 `updateConstraints`.
> `setNeedsLayout` 确保在未来的某个阶段 `layoutIfNeeded` 这个方法会调用 `layoutSubviews`.
