# KVO 和 KVC 的使用和实现

## 如何使用 KVO/KVC
### 简单使用
先定义一个 `Persson` 类，当做被观察的对象
```
//Person.h
#import <Foundation/Foundation.h>
@interface Person : NSObject
@end
// Person.m
#import "Person.h"
@interface Person () {
    NSString *address; //地址
    CGFloat weight; //体重
}
@property (nonatomic, copy) NSString *name; //名字
@property (nonatomic, assign) NSInteger age; //年龄
@end
@implementation Person
@end
```

再定义一个 `PersonObserver` 类，用于观察 `Person`
```
//PersonObserver.h
#import <Foundation/Foundation.h>
@interface PersonObserver : NSObject
@end
//PersonObserver.m
#import "PersonObserver.h"
@implementation PersonObserver
//观察者需要实现的方法
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSLog(@"old: %@", [change objectForKey:NSKeyValueChangeOldKey]);
    NSLog(@"old: %@", [change objectForKey:NSKeyValueChangeNewKey]);
    NSLog(@"context: %@", context);
}
@end
```

观察者和被观察者准备就绪，即可进行测试
```
// 测试 KVO & KVC
- (void)testMethod
{
    Person *aPerson = [[Person alloc] init];
    PersonObserver *aPersonObserver = [[PersonObserver alloc] init];
    //添加观察者
    //也可以观察 age、address、weight
    [aPerson addObserver:aPersonObserver
              forKeyPath:@"name"
                 options:(NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld)
                 context:@"this is a context"];
    //设置key的value值，aPersonObserver接收到通知
    [aPerson setValue:@"LiLei" forKey:@"name"];
    NSLog(@"name: %@", [aPerson valueForKey:@"name"]);
    //移除观察者
    [aPerson removeObserver:aPersonObserver forKeyPath:@"name"];
}
```
当代码执行到第16行时，`aPersonObserver` 接收到通知，打印 `change` 和 `context` 值。整个代码的执行结果如下：
```
old: <null>
new: LiLei
context: this is a context
name: LiLei
```

如果 `Person` 类里面还有个 `Job` 的属性
```
@property (nonautomic, strong) Job *aJob; //工作
```

```
#import <Foundation/Foundation.h>
@interface Job : NSObject
@property (nonautomic, copy) NSString *companyName; //公司名字
@property (nonautomic, assign) CGFloat salary; //薪水
@end
```

要观察和设置 `Person` 的薪水，只要这么写就可以
```
// 观察 salary
[aPerson addObserver:aPersonObserver
              forKeyPath:@"aJob.salary"
                 options:(NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld)
                 context:@"this is a context"];
//设置月薪：20k
 [aPerson setValue:@"20000.0" forKey:@"aJob.salary"];
```

### 操作集合
如果 Person 需要车，就添加 cars 属性，于是就有了很多车
```
@property (nonautomic, copy) NSArray *cars; //很多车
```
对 `NSArray *cars` 这种有序集合属性的操作有两种方法
* 使用 `valueForKey` 将 `cars` 直接取出来，再对集合元素进行操作
* 通过下面的方法直接操作；
```
/**
 * 有序集合的操作
 * 将所有方法 <Key> 替换成 Cars，且首字母大写
 */
//必须实现，对应于NSArray的基本方法count:
-countOf<Key>
//这两个必须实现一个，对应于 NSArray 的方法 objectAtIndex: 和 objectsAtIndexes:  
-objectIn<Key>AtIndex:  
-<Key>AtIndexes:  
//不是必须实现的，但实现后可以提高性能，其对应于 NSArray 方法 getObjects:range:
-get<Key>:range:   
//两个必须实现一个，类似 NSMutableArray 的方法 insertObject:atIndex: 和 insertObjects:atIndexes:
-insertObject:in<Key>AtIndex:  
-insert<Key>:atIndexes:   
//两个必须实现一个，类似于 NSMutableArray 的方法 removeObjectAtIndex: 和 removeObjectsAtIndexes:
-removeObjectFrom<Key>AtIndex:  
-remove<Key>AtIndexes:  
//可选的，如果在此类操作上有性能问题，就需要考虑实现之
-replaceObjectIn<Key>AtIndex:withObject:  
-replace<Key>AtIndexes:with<Key>:  
```
相对应的，像 NSSet 这种无序集合同样也有如下的方法可以使用
```
/**
 * 无序集合的操作
 * 将所有方法 <Key> 替换成 Cars，且首字母大写
 */
//必须实现，对应于NSArray的基本方法count:
-countOf<Key>   
//这两个必须实现一个，对应于 NSArray 的方法 objectAtIndex: 和 objectsAtIndexes:
-objectIn<Key>AtIndex:  
-<key>AtIndexes:  
//不是必须实现的，但实现后可以提高性能，其对应于 NSArray 方法 getObjects:range:
-get<Key>:range:   
//两个必须实现一个，类似 NSMutableArray 的方法 insertObject:atIndex: 和 insertObjects:atIndexes:  
-insertObject:in<Key>AtIndex:  
-insert<Key>:atIndexes:  
//两个必须实现一个，类似于 NSMutableArray 的方法 removeObjectAtIndex: 和 removeObjectsAtIndexes:
-removeObjectFrom<Key>AtIndex:  
-remove<Key>AtIndexes:  
//这两个都是可选的，如果在此类操作上有性能问题，就需要考虑实现之
-replaceObjectIn<Key>AtIndex:withObject:  
-replace<Key>AtIndexes:with<Key>:    
```
但是，如果要使用这些方法，开发者需自己实现一遍，所以使用上相对麻烦。

### 键值验证（KVV，Key-Value Validate）
KVC 提供了键值验证（KVV）机制，让开发者有机会能够挽回错误，保证数据的一致性。开发者需先调用下面
```
- (BOOL)validateValue:(inout id *)ioValue forKey:(NSString *)inKey error:(out NSError **)outError;
```
这个方法会默认调用的实现方法如下
```
- (BOOL)validate<Key>:error:  
```
举个例子，上面的例子中，希望能够验证 `Person` 的属性 `name` 不为空，就可以这么写
```
// 测试 KVV
- (void)testMethod
{
    Person *aPerson = [[Person alloc] init];
    PersonObserver *aPersonObserver = [[PersonObserver alloc] init];
    //添加观察者
    [aPerson addObserver:aPersonObserver
              forKeyPath:@"name"
                 options:(NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld)
                 context:@"this is a context"];
    NSString *name = @"LiLei";
    NSString *key = @"name";
    NSError *error = nil;
    // KVV 验证
    BOOL isLegalName = [aPerson validateValue:&name forKey:key error:&error];
    if (isLegalName) {
        NSLog(@"it's a legal name.");
        [aPerson setValue:name forKey:key];
    }else{
        NSLog(@"the name is illegal.");
    }
    //移除观察者
    [aPerson removeObserver:aPersonObserver forKeyPath:@"name"];
}

```

```
//Person.m
//Person.m 文件新增验证名字的KVV方法
@implementation Person
- (BOOL)validateName:(NSString **)name error:(NSError * __autoreleasing *)outError
{
    if ((*name).length == 0)
    {
        (*name) = @"default name";
        return NO;
    }
    return YES;
}
@end
```

### 集合操作符（Collection Operator）
集合运算符是一种特殊的 key path，通过`- (id)valueForKeyPath:(NSString *)keyPath`方法获取集合中的信息，其格式如下：
```
keypathToCollectoin.@CollectionOperator.keypathToProperty
```
由数值组成的集合，总共有 5 中操作符
* sum：集合中所有数值的和
* avg：集合中所有数值的平均数
* max：集合中的最大数值
* min：集合中的最小数值
* count：集合元素的数量

```
//假设在 Person 类中还有个存储数值的数组 array
//获取array 中的所有数值的和
CGFloat sum = [aPerson valueForKeyPath:@"@sum.array"];
//获取 array 中所有数值的平均数
CGFloat avg = [aPerson valueForKeyPath:@"@avg.array"];
```

由对象组成的集合有 2 种操作符

* unionOfObjects：返回集合中的所有元素
* distinctUnionOfObjects：返回集合去重后的所有元素
由数组组成的集合（集合中有集合）有如下 3 种操作符。

* unionOfArrays：用于Array集合，返回集合中的所有元素
* distinctUnionOfArrays：用于Array集合，返回集合中去重后的所有元素
* distinctUnionOfSets：用于Set集合，返回集合中（去重后）的所有元素
由于 Set 中的元素本身就是不重复的，所以没有 unionOfSets 操作符。
### 手动键值观察
通过自动属性，建立键值观察，都属于自动键值观察。因为使用这种方法，只要设置键值，就会自动发出通知。而手动键值观察，不能使用自动化属性，需要自己写 setter/getter 方法，手动发送通知。
```
//手动通知的实现
@interface Person : NSObject
{
    NSString *name;
}
- (NSString *)name;
- (void)setName:(int)theName;
@end
@implementation Person
- (id) init
{
    self = [super init];
    if (nil != self) {
        name = @"LiLei";
    }
    return self;
}
- (NSString *)name
{
    return name;
}
- (void)setName:(NSString *)theName
{
    //发送通知：键值即将改变
    [self willChangeValueForKey:@"name"];
    name = theName;
    //发送通知：键值已经修改
    [self didChangeValueForKey:@"name"];
}
/**
 *  当设置键值之后，通过此方法，决定是否发送通知
 */
+ (BOOL) automaticallyNotifiesObserversForKey:(NSString *)key
{
    //当 key 为 name时，手动发送通知
    if ([key isEqualToString:@"age"]) {
        return NO;
    }
    //当为其他key时，自动发送通知
    return [super automaticallyNotifiesObserversForKey:key];
}
@end
```

### 设置属性之间的依赖
假如有个 `Person` 类，类里有三个属性，`fullName`、`firstName`、`lastName`。按照之前的知识，如果需要观察名字的变化，就要分别添加 `fullName`、`firstName`、`lastName` 三次观察，非常麻烦。如果能够只观察 `fullName`，并建立 `fullName` 和 `firstName`、`lastName` 的某种依赖关系，当发生变化时，也受到通知，那该多好啊！

KVC 刚好提供这种键之间的依赖方法，格式如下
```
+ (NSSet *)keyPathsForValuesAffecting<Key>;
```
这方法使得 Key 之间能够建立依赖关系，为了便于说明，直接用 属性依赖 这个词代替 Key 之间的依赖。含义不同，结果一致。
下面就使用这种方法解决 Key 之间的依赖关系。
`Person` 类为被观察者
```
//Person.h
#import <Foundation/Foundation.h>
@interface Person : NSObject
@end
//Person.m
#import "Person.h"
@interface Person ()
@property (nonatomic, copy) NSString *fullName; //名字，依赖于firstName、lastName
@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@end
@implementation Person
//设置属性依赖：fullName属性依赖于firstName、lastName
//如果观察name，当firstName、lastName发生变化时，观察者也会收到name变化通知
+ (NSSet *)keyPathsForValuesAffectingFullName
{
    NSSet *set = [NSSet setWithObjects:@"firstName", @"lastName", nil];
    return set;
}
@end
```
`PersonObserver` 类为观察者
```
//PersonObserver.h
#import <Foundation/Foundation.h>
@interface PersonObserver : NSObject
@end
//PersonObserver.m
#import "PersonObserver.h"
@implementation PersonObserver
//观察者需要实现的方法
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSLog(@"observer receive change infomation");
}
@end
```
准备就绪，就可以测试依赖的属性了
```
- (void)testMethod
{
    Person *aPerson = [[Person alloc] init];
    PersonObserver *aPersonObserver = [[PersonObserver alloc] init];
    //观察fullName属性
    [aPerson addObserver:aPersonObserver
              forKeyPath:@"fullName"
                 options:(NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld)
                 context:@"this is a context"];
    //设置firstName时，aPersonObserver仍然会收到name变化的通知
    [aPerson setValue:@"LiLei" forKey:@"firstName"];
    //移除观察者
    [aPerson removeObserver:aPersonObserver forKeyPath:@"fullName"];
}
```
输出结果，发现虽然观察的是 `fullName`，但是当修改 `firstName` 的时候，观察者也会收到 `fullName` 变化的通知，达到了我们的期望。

参考资料：
[Apple Documentation: Key-Value Coding Programming Guide](Apple Documentation: Key-Value Coding Programming Guide )

[objc中国 - 卢思豪：KVC 和 KVO ](http://objccn.io/issue-7-3/)

[KVC/KVO原理详解及编程指南](http://blog.csdn.net/wzzvictory/article/details/9674431)

[飘飘白云：深入浅出Cocoa 详解键值观察（KVO）及其实现机理 ](http://blog.csdn.net/kesalin/article/details/8194240)

[iliunian：iOS编程——Objective-C KVO/KVC机制 ](http://www.iliunian.com/919.html)
