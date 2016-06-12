# 单例模式(Singleton)
- UIApplication类提供了`＋sharedAPplication`方法创建和获取UIApplication单例
- NSBundle类提供了`+mainBunle`方法获取NSBundle单例
- NSFileManager类提供了`＋defaultManager`方法创建和获得NSFileManager单例（PS：有些时候我们得放弃使用单例模式，使用－init方法去实现一个新的实例，比如使用委托时）
- NSNotificationCenter提供了`＋defaultCenter`方法创建和获取NSNotificationCenter单例
- NSUserDefaults类提供了`＋defaultUserDefaults`方法去创建和获取NSUserDefaults单例
单例是一个在 Cocoa 中很常用的模式了。对于一些希望能在全局方便访问的实例，或者在 app 的生命周期中只应该存在一个的对象，我们一般都会使用单例来存储和访问。在 Objective-C 中单例的公认的写法类似下面这样：
单例模式新建和获取实例的类模版
```
	//Singleton.h
	@interface Singleton : NSObject
	+ (Singleton *)sharedSingleton; // 1
	@end
	//Singleton.m
	#import "Singleton.h"
	@implementation Singleton
	static Singleton *sharedSingleton = nil; // 2

	+ (Singleton *)sharedSingleton{
	    static dispatch_once_t once; // 3
	    dispatch_once(&once,^{
	        sharedSingleton = [[self alloc] init]; // 4
	        //doSometing
	    });
	    return sharedSingleton; // 5
	}
```
1. 声明一个可以新建和获取单个实例对象的方法
2. 声明一个static类型的类变量
3. 使用 GCD 中的 dispatch\_once\_t 可以保证里面的代码只被调用一次，以此保证单例在线程上的安全。
4. 调用dispatch\_once执行该任务指定的代码块，在该代码块中实例化上文声明的类变量
5. 返回在整个应用的生命周期中只会被实例化一次的变量
