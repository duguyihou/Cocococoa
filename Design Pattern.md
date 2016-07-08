## Observer Pattern

观察者模式(Observer Pattern)：定义对象间的一种一对多依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象皆得到通知并被自动更新。
在iOS中典型的推模型实现方式为NSNotificationCenter和KVO。

### NSNotificationCenter

![NSNotificationCenter](http://7xs5iw.com1.z0.glb.clouddn.com/1721232-dd65f5d099b64955.JPG)

1. 观察者Observer，通过NSNotificationCenter的addObserver:selector:name:object接口来注册对某一类型通知感兴趣。在注册时候一定要注意，NSNotificationCenter不会对观察者进行引用计数+1的操作，我们在程序中释放观察者的时候，一定要去报从center中将其注销了。

2. 通知中心NSNotificationCenter，通知的枢纽。

3. 被观察的对象，通过postNotificationName:object:userInfo:发送某一类型通知，广播改变。

4. 通知对象NSNotification，当有通知来的时候，Center会调用观察者注册的接口来广播通知，同时传递存储着更改内容的NSNotification对象。

### KVO
KVO的全称是Key-Value Observer，即键值观察。是一种没有中心枢纽的观察者模式的实现方式。一个主题对象管理所有依赖于它的观察者对象，并且在自身状态发生改变的时候主动通知观察者对象。
1. 注册观察者
```
[object addObserver:self forKeyPath:property options:NSKeyValueObservingOptionNew context:]
```

2. 更改主题对象属性的值，即触发发送更改的通知。

3. 在制定的回调函数中，处理收到的更改通知。

4. 注销观察者 [object removeObserver:self forKeyPath:property]。
