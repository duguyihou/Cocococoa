## KVC & KVO

KVC:键 – 值编码是一种间接访问对象的属性使用字符串来标识属性，而不是通过调用存取方法，直接或通过实例变量访问的机制。

很多情况下可以简化程序代码。apple文档其实给了一个很好的例子。

KVO:键值观察机制，他提供了观察某一属性变化的方法，极大的简化了代码。

具体用看到嗯哼用到过的一个地方是对于按钮点击变化状态的的监控。

比如我自定义的一个button

```objc
[self addObserver:self forKeyPath:@"highlighted" options:0 context:nil];
#pragma mark?KVO
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object
change:(NSDictionary *)change context:(void *)context
{
if ([keyPath isEqualToString:@"highlighted"]) {
[self setNeedsDisplay];
}
}
```

对于系统是根据keypath去取的到相应的值发生改变，理论上来说是和kvc机制的道理是一样的。

对于kvc机制如何通过key寻找到value：

"当通过KVC调用对象时，比如：[self valueForKey:@"someKey"]时，程序会自动试图通过几种不同的 方式解析这个调用。首先查找对象是否带有 someKey 这个方法，如果没找到，会继续查找对象是否带 有someKey这个实例变量(iVar)，如果还没有找到，程序会继续试图调用 -(id) valueForUndefinedKey:这个方法。 如果这个方法还是没有被实现的话，程序会抛出一个NSUndefinedKeyException异常错误。

Key-Value Coding查找方法的时候，不仅仅会查找someKey这个方法，还会查找getsomeKey这个方法， 前面加一个get，或者_someKey以及_getsomeKey这几种形式。同时， 查找实例变量的时候也会不仅仅查找someKey这个变量，也会查找_someKey这个变量是否存在。
