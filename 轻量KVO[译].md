# [译]轻量KVO
>原文地址:[Lightweight Key-Value Observing](http://chris.eidhof.nl/posts/lightweight-key-value-observing.html)

![](http://7xs5iw.com1.z0.glb.clouddn.com/kvo_objects_implementation.jpg)

在这篇文章中，我会实现一个自己用的简单 KVO 类，我认为 KVO 非常棒，然而对于我大部分的使用场景来说，有这两个问题：
1. 我不喜欢在 `observeValueForKeyPath:ofObject:change:context:`方法里通过 keyPath 值来做调度，当 Observe 比较多的对象时，会使得代码变得杂乱和迷惑。
2. 必须手动的来注册和删除一个观察者，如果能自动做就好了。

So，我们开始这个实现。这个技巧我第一次是在 THObserversAndBinders 项目中见到，本篇内容也仅仅描述了一下里面的做法，同时做了简化。
首先，我们定义一下我们的这个类，我们这个帮助类的类名是`Observer`:
```
@interface Observer : NSObject
+ (instancetype)observerWithObject:(id)object
                           keyPath:(NSString*)keyPath
                            target:(id)target
                          selector:(SEL)selector;
@end
```

Observer 类的这个类方法有四个参数，每个参数都是自解释的，我选择使用 target/action 模式，当然也可以使用 block，但是那样的话需要做 weakSelf/strongSelf 的转换，你懂的，通常来说分来来做比较好。

我们做的是在初始化方法中设置 KVO，并在 dealloc 方法中移除。这意味着一旦 Observer 对象被 retain，我们就有了一个观察者，下面这段代码是从我的一个 ViewCOntroller 中拿来的：
```
self.usernameObserver = [Observer observerWithObject:self.user
                                             keyPath:@"name"
                                              target:self
                                            selector:@selector(usernameChanged)];
```

把这个 Observer 对象作为一个属性放在 ViewController 中来保证被 retain，一旦我们的 Viewcontroller 被释放，就会设置它为 nil，observer 就停止观察了。

在这个实现中，使用一个 weak 引用指向被观察对象和观察者 (target) 是很重要的，如果两个中的其中一个是 nil，我们就停止向观察者发送消息。
```
@interface Observer ()
@property (nonatomic, weak) id target;
@property (nonatomic) SEL selector;
@property (nonatomic, weak) id observedObject;
@property (nonatomic, copy) NSString* keyPath;
@end
```

初始化器里设置 KVO 通知，使用 self 作为 context，如果我们会有一个子类也添加类似的观察者时就很有必要了。
```
- (id)initWithObject:(id)object keyPath:(NSString*)keyPath target:(id)target selector:(SEL)selector
{
  if (self) {
    self.target = target;
    self.selector = selector;
    self.observedObject = object;
    self.keyPath = keyPath;
    [object addObserver:self forKeyPath:keyPath options:0 context:self];
  }
  return self;
}
```
一旦被观察者发生变化，我们就通知观察者（target），如果它还存在的话：
```
- (void)observeValueForKeyPath:(NSString*)keyPath ofObject:(id)object change:(NSDictionary*)change context:(void*)context
{
if (context == self) {
  id strongTarget = self.target;
  if ([strongTarget respondsToSelector:self.selector]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    [strongTarget performSelector:self.selector];
#pragma clang diagnostic pop
  }
}
}
```
最后在 dealloc 方法中移除观察者对象:
```
- (void)dealloc
{
    id strongObservedObject = self.observedObject;
    if (strongObservedObject) {
        [strongObservedObject removeObserver:self forKeyPath:self.keyPath];
    }
}
```
这就是全部内容了。还有很多可以扩展的地方，比如增加 block 的支持，或者我比较喜欢的 trick：再增加爱一个方便的构造方法用来第一次直接调用 action。然而，我想的是展现出这个技术的核心部分，你可以根据自己的需求来调整它。

这个技术的优点是在使用 KVO 的时候不需要记住太多东西，仅仅 retain 住 Observer 对象，然后在完成的试试置为 nil 即可，剩下的会自动完成。
