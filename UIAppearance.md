## 使用UIAppearance来自定义应用的外观

**自定义导航条背景**

```objc
[[UINavigationBar appearance] setBackgroundImage:[UIImage imageNamed:@"background"] forB
```

**自定义导航标题文字属性**

```objc
[[UINavigationBar appearance] setTitleTextAttributes:@{UITextAttributeTextColor:[UIColor darkGrayColor],UITextAttributeTextShadowColor:[UIColor clearColor]}];
```

**自定义导航条返回和左右按钮按钮背景**

```objc
[[UIBarButtonItem appearanceWhenContainedIn:[UINavigationBar class], nil] setBackButtonBackgroundImage:[UIImage imageNamed:@"back_button_background"] forState:UIControlStateNormal barMetrics:UIBarMetricsDefault];
[[UIBarButtonItem appearanceWhenContainedIn:[UINavigationBar class], nil] setBackgroundImage:[UIImage imageNamed:@"button_background"] forState:UIControlStateNormal barMetrics:UIBarMetricsDefault];
```

**自定义底部Tab条的背景**

```objc
[[UITabBar appearance] setBackgroundImage:[UIImage imageNamed:@"background"]];
```

**自定义底部条标题文字属性**

```objc
[[UITabBarItem appearance] setTitleTextAttributes:@{UITextAttributeTextColor:[UIColor grayColor]} forState:UIControlStateNormal];
[[UITabBarItem appearance] setTitleTextAttributes:@{UITextAttributeTextColor:[UIColor orangeColor]} forState:UIControlStateSelected];
```

> 只要是头文件中有"UI_APPEARANCE_SELECTOR"标记的方法都是可以用UIAppearance协议对象去调的。 注意这些自定义方法都要在相应的对象显示之前调用，可以放到App启动后立刻配置， 以后只要这个对象显示之前，就会设置相应的属性。

**创建一个可自定义外观的控件** 对于我们自己定义的控件，也可以支持UIAppearance协议，这样我们的控件也能支持自定义了。你只需要写一个设置外观的settor，然后在settor方法后面加上"UI_APPEARANCE_SELECTOR"标记就可以，其他什么都不需要做。比如一个可以自定义选择状态背景颜色的TableViewCell。

```objc
@interface CustomCell : UITableViewCell
- (void)setSelectedBackgroundColor:(UIColor*)color UI_APPEARANCE_SELECTOR;
@end

@implementation CustomCell
- (id)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier
{
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        self.selectedBackgroundView = [UIView new];
        self.selectedBackgroundView.backgroundColor = [UIColor lightGrayColor];
    }
    return self;
}
- (void)setSelectedBackgroundColor:(UIColor*)color{
    self.selectedBackgroundView.backgroundColor = color;
}
@end
```

注意，官方文档中强调Appearance的setter定义格式应为：

```objc
- (void)setProperty:(PropertyType)property forAxis1:(IntegerType)axis1 axis2:(IntegerType)axis2 axisN:(IntegerType)axisN;
- (PropertyType)propertyForAxis1:(IntegerType)axis1 axis2:(IntegerType)axis2 axisN:(IntegerType)axisN;
```

**UIAppearance实现原理** 在通过UIAppearance调用"UI_APPEARANCE_SELECTOR"标记的方法来配置外观时，UIAppearance实际上没有进行任何实际调用，而是把这个调用保存起来（在Objc中可以用NSInvocation对象来保存一个调用）。当实际的对象显示之前（添加到窗口上，drawRect:之前），就会对这个对象调用之前保存的调用。当这个setter调用后，你的界面风格自定义就完成了。
