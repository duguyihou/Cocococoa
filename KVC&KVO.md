# KVC
>[官方文档](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/BasicPrinciples.html)


让一个类实现 KVO 的方式是遵循 NSKeyValueCoding 这个协议，该协议中定义了 2 个方法：`valueForKey:`and`setValue:forKey:`. 这两个方法用来通过 key 访问和获取对象属性。
## 键和键的路径
键是一个描述对象具体属性的字符。原则上，一个键与一个寄存器方法或者随着接收值而变化的实例变量的名称相符。这些键必须使用ASCII编码，以小写字母开头，同时不包含空格。

以下是一些可以使用的键：`payee`, `openingBalance`, `transactions` 以及 `amount`.

键的路径是一个用来遍历指定对象的属性序列，这个序列是以点分隔的字符串。这个键里的第一个属性是与接收者相关的，同时随后的每一个键都相对前一个属性的值相关。

例如，键的路径 address.street 可以从接收者对象得到 address 属性当中的值，然后可以查到与 address 对象相关联的 street 属性。

## 利用KVC机制获取属性的值
`valueForKey` 方法返回与接收者相关的特定键值。如果相对这个键没有对应的存取器和实例变量，那么接收者会给自己发动一条 `valueForUndefinedKey:`的消息。默认的 `valueForUndefinedKey` 的实现会抛出一个`NSUndefinedKeyException` ，它的子类可以重载这个行为。

相似地， valueForKeyPath: 方法也返回与接收者相关的特定键值。任何在键路径上的非键值编码的键只要符合条件，其对象就会获取一个 valueForUndefinedKey 的信息。

`dictionaryWithValuesForKeys:`方法可以获取一个与接收者相关的键值数组。返回的 NSDictionary 包含所有在数组当中的键的值。

>注意：一些泛型类，例如 NSArray, NSSet, 以及 NSDictionary 是不能包含 nil 作为其中的值的。相反，你可以用一个 NSNull 来代替 nil 。 NSNull 可以作为一个单一的实例出现在类属性里用来代替 nil 。 dictionaryWithValuesForKeys: 和 setValuesForKeysWithDictionary: 这两个方法的实现可以自动地把 null 转化成 NSNull ，所以你也不必要在类里明确地测试 NSNull 的转化了。

当一个值是返回给一个包含了多对多关系的键，而且这个键并不是路径中的最后一个键时，这个返回的值就会是包含了所有在多对多的键右侧的键的所有值的集合。例如，获取路径 transactions.payee 的值会返回一个包括了所有交易中所有收款人对象的数组。这个规则对多维数组也适用，路径 accounts.transactions.payee 将会返回所有账户的所有交易的所有收款人对象的数组。

## 利用KVC设置属性的值

setValue:forKey: 方法可以给指定的与接收者相关的键赋值为给定的数值。 setValue:forKey: 方法的实现可以解开 NSValue 的对象，声明纯数据和结构并分配给属性。参考文档 [Scalar and Structure Support](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/DataTypes.html#//apple_ref/doc/uid/20002171-BAJEAIEE) 可以深入了解关于包装和解开包装的语法和用法。

如果指定的键并不存在，接收者就会被发送 `setValue:forUndefinedKey:`的消息。默认的 `setValue:forUndefinedKey:`实现会抛出一个 NSUndefinedKeyException。然而，子类可以重写这个方法，用默认的方法处理请求。

`setValue:forKeyPath:`和上面的方法做法相似，但是它允许处理一个键的路径甚至是单独一个键。

最后,`setValuesForKeysWithDictionary:`可以把给定的字典赋值给接收者的属性，利用字典的键来辨别属性。默认的实现对每个键值对引用`setValue:forKey:`，然后按要求用 NSNull 代替 nil 。

你还需要额外考虑的是，在你给一个没有对象的属性赋值为 nil 会发生什么呢？这种情况下，接收者会给自己发送 `setNilValueForKey:`的消息。默认的 `setNilValueForKey:`实现会抛出一个 NSInvalidArgumentException。你的应用可以重载这个方法来替代默认值或者标记的值，然后用新的值调用 `setValue:forKey:`。

## KVC & 点语法
Objective-C中的点语法和KVC是相互垂直的两种用法。无论是否使用点句法，你都可以使用KVC，也无论你是否使用KVC，你都可以使用点句法。虽然点句法随时都可以使用，但是用法有不同。(You can use key-value coding whether or not you use the dot syntax, and you can use the dot syntax whether or not you use KVC. Both, though, make use of a “dot syntax.” )在KVC里，点语法是用来界定键路径当中的元素的。记住，当你用点语法调用一个属性的时候，你调用的是接收者的标准存取器方法。

你可以使用KVC调用一个属性，下面给出了一个定义类的样例：
```
@interface MyClass
@property NSString *stringProperty;
@property NSInteger integerProperty;
@property MyClass *linkedInstance;
@end
```

你可以通过KVC在一个实例中调用它的属性：
```
MyClass *myInstance = [[MyClass alloc] init];
NSString *string = [myInstance valueForKey:@"stringProperty"];
[myInstance setValue:@2 forKey:@"integerProperty"];
```

为了分辨点语法在KVC和原生语法之间的区别，你可以参考以下代码:
```
MyClass *anotherInstance = [[MyClass alloc] init];
myInstance.linkedInstance = anotherInstance;
myInstance.linkedInstance.integerProperty = 2;
```
以下代码和上例等价：
```
MyClass *anotherInstance = [[MyClass alloc] init];
myInstance.linkedInstance = anotherInstance;
[myInstance setValue:@2 forKeyPath:@"linkedInstance.integerProperty"];
```

键-值编码中的基本调用包括``-valueForKey:`和`-setValue:forKey:``。以字符串的形式向对象发送消息，这个字符串是我们关注的属性的关键。
`valueForKey:`首先查找以键`-key`或`-isKey`命名的`getter`方法。如果不存在`getter`方法（假如我们没有通过`@synthesize`提供存取方法），它将在对象内部查找名为`_key`或`key`的实例变量。
对于KVC，Cocoa自动放入和取出标量值（int，float和struct）放入`NSNumber`或`NSValue`中；当使用`-setValue:ForKey:`时，它自动将标量值从这些对象中取出。仅KVC具有这种自动包装功能，常规方法调用和属性语法不具备该功能。
`-setValue:ForKey:`的工作方式和`-valueForKey:`相同。它首先查找名称的`setter`方法，如果不存在`setter`方法，它将在类中查找名为`_key`或`key`的实例变量。
使用KVC访问属性的代价比直接使用存取方法要大，所以只在需要的时候才用。

KVC不仅可以访问对象属性，也能访问一些标量（例如 int 和 CGFloat）和 struct（例如 CGRect）。Foundation 框架会为我们自动封装它们。
```
@property (nonatomic) CGFloat height;
[object setValue:@(20) forKey:@"height"];
```

有关KVC的更多用法，参看下面的文章：
http://blog.csdn.net/wzzvictory/article/details/9674431
http://objccn.io/issue-7-3/

# KVO
> [官方文档](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html)

KVO是一种称为键－值观察的机制，对象可以通过它得到其他对象特性属性的变更通知。
## At a Glance
KVO是一种称为键－值观察的机制，对象可以通过它得到其他对象特性属性的变更通知。这种机制在MVC模式的场景中很重要，控制器对象观察模型对象的属性，视图对象通过控制器观察模型对象的属性。另外，一个模型对象可能会观察其他模型对象或者甚至本身。这一机制基于NSKeyValueObserving非正式协议，Cocoa通过这个协议为所有遵守协议的对象提供了一种自动化的属性观察能力。要实现自动观察，参与KVO的对象需要符合KVC的要求和存取方法，也可以手动实现观察者通知，也可以两者都保留。

设置一个属性的观察者需要三步，理解这些步骤可以更清楚的知道KVO的工作框图：
1. 首先看看你当前的场景如果使用KVO是否更妥当，比如，当一个实例的某个具体属性有任何变更的时候，另一个实例需要被通知。

![](http://7xs5iw.com1.z0.glb.clouddn.com/kvo_objects.jpg)

比如，BankObject中的accountBalance属性有任何变更时，某个PersonObject对象都要觉察到。
2. 这个`PersonObject`对象必须注册成为`BankObject`的`accountBalance`属性的观察者，可以通过发送addObserver:forKeyPath:options:context:消息来实现。

![](http://7xs5iw.com1.z0.glb.clouddn.com/kvo_objects_connection.jpg)

>注意：`addObserver:forKeyPath:options:context:`方法在你指定的两个实例间建立联系，而不是在两个类之间。

3. 为了回应变更通知，观察者必须实现`observeValueForKeyPath:ofObject:change:context:`方法。这个方法的实现决定了观察者如何回应变更通知。你可以在这个方法里自定义如何回应被观察属性的变更。

![](http://7xs5iw.com1.z0.glb.clouddn.com/kvo_objects_implementation.jpg)

[Registering for Key-Value Observing](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOBasics.html#//apple_ref/doc/uid/20002252-BAJEAIEE) 描述如何注册接收观察通知。

4. 当一个被观察属性的值以符合KVO方式变更或者当它依赖的键变更时，`observeValueForKeyPath:ofObject:change:context:`方法会被自动执行。

![](http://7xs5iw.com1.z0.glb.clouddn.com/kvo_objects_notification.jpg)

[Registering Dependent Keys](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVODependentKeys.html#//apple_ref/doc/uid/20002179-BAJEAIEE)解释如何具体说明一个键的值依赖另一个键的值。

KVO’s primary benefit is that you don’t have to implement your own scheme to send notifications every time a property changes. Its well-defined infrastructure has framework-level support that makes it easy to adopt—typically you do not have to add any code to your project. In addition, the infrastructure is already full-featured, which makes it easy to support multiple observers for a single property, as well as dependent values.

## Registering for Key-Value Observing
为了接收属性的KVO通知，需要三件事：
- 被观察的类必须KVO兼容你想观察的属性。
- 必须使用`addObserver:forKeyPath:options:context:`为被观察的对象注册观察对象。
- 观察对象必须实现` observeValueForKeyPath:ofObject:change:context:`。

### Registering as an Observer
>通过发送`addObserver:forKeyPath:options:context:`消息来注册观察者。

被观察者必须首先通过发送`addObserver:forKeyPath:options:context:`信息传递观察者对象以及被观察的键值路径以达到属性变更时观察者被通知的目的。可选参数可详细提供给观察者何时发送变更通知的信息。`NSKeyValueObservingOptionNew`和`NSKeyValueObservingOptionOld`选项分别标识在观察者接收通知时`change`字典对应入口提供更改后的值和更改前的值。若要接收这两个值，会用到位运算`OR`。可以用以下方式提取出改变前后的值：
```
id oldValue = change[NSKeyValueChangeOldKey];
id newValue = change[NSKeyValueChangeNewKey];

```

>更简单的办法是用 NSKeyValueObservingOptionPrior 选项.

我们常常需要当一个值改变的时候更新 UI，但是我们也要在第一次运行代码的时候更新一次 UI。我们可以用 KVO 并添加 `NSKeyValueObservingOptionInitial`的选项 来一箭双雕地做好这样的事情。这将会让 KVO 通知在调用`-addObserver:forKeyPath:...`到时候也被触发。

当我们注册 KVO 通知的时候，我们可以添加`NSKeyValueObservingOptionPrior`选项，这能使我们在键值改变之前被通知。这和`-willChangeValueForKey:`被触发的时间相对应。
如果我们注册通知的时候附加了`NSKeyValueObservingOptionPrior`选项，我们将会收到两个通知：一个在值变更前，另一个在变更之后。变更前的通知将会在`change`字典中有不同的键。

```
- (void)registerAsObserver {
    /*
     Register 'inspector' to receive change notifications for the "openingBalance" property of
     the 'account' object and specify that both the old and new values of "openingBalance"
     should be provided in the observe… method.
     */
    [account addObserver:inspector
             forKeyPath:@"openingBalance"
                 options:(NSKeyValueObservingOptionNew |
                            NSKeyValueObservingOptionOld)
                    context:NULL];
}
```
context是一个指针，当`observeValueForKeyPath:ofObject:change:context:`方法执行时context会提供给观察者。context可以是C指针或者一个对象引用，既可以当作一个唯一的标识来分辨被观察的变更，也可以向观察者提供数据。

>注意，键值观察`addObserver:forKeyPath:options:context:`方法不会对观察者、被观察对象及context维持强引用。如果有必要，你应该维持强引用。

### Receiving Notification of a Change
当被观察的属性变更时，观察者会接到`observeValueForKeyPath:ofObject:change:context:`消息，所有的观察者都必须实现这个方法。
观察者会被提供触发通知的对象和`keyPath`，一个包含变更详细信息的字典，还有一个注册观察者时提供的`context`指针。

```
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {

    if ([keyPath isEqual:@"openingBalance"]) {
        [openingBalanceInspectorField setObjectValue:
            [change objectForKey:NSKeyValueChangeNewKey]];
    }
    /*
     Be sure to call the superclass's implementation *if it implements it*.
     NSObject does not implement the method.
     */
    [super observeValueForKeyPath:keyPath
                         ofObject:object
                           change:change
                           context:context];
}

```

关于change参数，它是一个字典，有五个常量作为它的键：
```
NSString *const NSKeyValueChangeKindKey;  
NSString *const NSKeyValueChangeNewKey;  
NSString *const NSKeyValueChangeOldKey;  
NSString *const NSKeyValueChangeIndexesKey;  
NSString *const NSKeyValueChangeNotificationIsPriorKey;
```

#### NSKeyValueChangeKindKey
指明了变更的类型，值为“NSKeyValueChange”枚举中的某一个，类型为NSNumber。
```
enum {
   NSKeyValueChangeSetting = 1,
   NSKeyValueChangeInsertion = 2,
   NSKeyValueChangeRemoval = 3,
   NSKeyValueChangeReplacement = 4
};
typedef NSUInteger NSKeyValueChange;
```

#### NSKeyValueChangeNewKey
如果`NSKeyValueChangeKindKey`的值为`NSKeyValueChangeSetting`，并且 `NSKeyValueObservingOptionNew`选项在注册观察者时也指定了，那么这个键的值就是属性变更后的新值。
对于`NSKeyValueChangeInsertion`或者`NSKeyValueChangeReplacement`，如果 `NSKeyValueObservingOptionNew`选项在注册观察者时也指定了，这个键的值是一个数组，其包含了插入或替换的对象。

#### NSKeyValueChangeOldKey
如果 `NSKeyValueChangeKindKey`的值为`NSKeyValueChangeSetting`，并且 `NSKeyValueObservingOptionOld`选项在注册观察者时也指定了，那么这个键的值就是属性变更前的旧值。
对于 `NSKeyValueChangeRemoval` 或者`NSKeyValueChangeReplacement`，如果 `NSKeyValueObservingOptionOld`选项在注册观察者时也指定了，这个键的值是一个数组，其包含了被移除或替换的对象。

#### NSKeyValueChangeIndexesKey
如果 `NSKeyValueChangeKindKey`的值为`NSKeyValueChangeInsertion`, `NSKeyValueChangeRemoval`, 或者 `NSKeyValueChangeReplacement`，这个键的值是一个`NSIndexSet`对象，包含了增加，移除或者替换对象的index。

#### NSKeyValueChangeNotificationIsPriorKey
如果注册观察者时`NSKeyValueObservingOptionPrior`选项被指明了，此通知会在变更发生前被发出。其类型为`NSNumber`，包含的值为`YES`。我们可以像以下这样区分通知是在改变之前还是之后被触发的：
```
if ([change[NSKeyValueChangeNotificationIsPriorKey] boolValue]) {
    // 改变之前
} else {
    // 改变之后
}
```
### Removing an Object as an Observer
你可以通过发送`removeObserver:forKeyPath:`消息来移除观察者，需要指明观察对象和路径。

```
- (void)unregisterForChangeNotification {
    [observedObject removeObserver:inspector forKeyPath:@"openingBalance"];
}
```

上面的代码将`openingBalance`属性的观察者`inspector`移除，移除后观察者再也不会收到`observeValueForKeyPath:ofObject:change:context:`消息。
在移除观察者之前，如果`context`是一个对象的引用，那么必须保持对它的强引用直到观察者被移除。

## KVO Compliance
>有两种方法可以保证变更通知被发出。
* 自动发送通知是`NSObject`提供的，并且一个类中的所有属性都默认支持，只要是符合KVO的。一般情况你使用自动变更通知，你不需要写任何代码。
* 人工变更通知需要些额外的代码，但也对通知发送提供了额外的控制。你可以通过重写子类`automaticallyNotifiesObserversForKey:`方法的方式控制子类一些属性的自动通知。

### Automatic Change Notification
```
// Call the accessor method.
[account setName:@"Savings"];

// Use setValue:forKey:.
[account setValue:@"Savings" forKey:@"name"];

// Use a key path, where 'account' is a kvc-compliant property of 'document'.
[document setValue:@"Savings" forKeyPath:@"account.name"];

// Use mutableArrayValueForKey: to retrieve a relationship proxy object.
Transaction *newTransaction = <#Create a new transaction for the account#>;
NSMutableArray *transactions = [account mutableArrayValueForKey:@"transactions"];
[transactions addObject:newTransaction];
```

### Manual Change Notification
```
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)theKey {

    BOOL automatic = NO;
    if ([theKey isEqualToString:@"openingBalance"]) {
        automatic = NO;
    }
    else {
        automatic = [super automaticallyNotifiesObserversForKey:theKey];
    }
    return automatic;
}
```

要实现人工观察者通知，你要执行在变更前执行`willChangeValueForKey:`方法，在变更后执行`didChangeValueForKey:`方法：
```
- (void)setOpeningBalance:(double)theBalance {
    [self willChangeValueForKey:@"openingBalance"];
    _openingBalance = theBalance;
    [self didChangeValueForKey:@"openingBalance"];
}
```

为了使不必要的通知最小化我们应该在变更前先检查一下值是否变了：

```
- (void)setOpeningBalance:(double)theBalance {
    if (theBalance != _openingBalance) {
        [self willChangeValueForKey:@"openingBalance"];
        _openingBalance = theBalance;
        [self didChangeValueForKey:@"openingBalance"];
    }
}
```

如果一个操作导致了多个键的变化，你必须嵌套变更通知：
```
- (void)setOpeningBalance:(double)theBalance {
    [self willChangeValueForKey:@"openingBalance"];
    [self willChangeValueForKey:@"itemChanged"];
    _openingBalance = theBalance;
    _itemChanged = _itemChanged+1;
    [self didChangeValueForKey:@"itemChanged"];
    [self didChangeValueForKey:@"openingBalance"];
}
```

在to-many关系操作的情形中，你不仅必须表明key是什么，还要表明变更类型和影响到的索引。变更类型是一个 `NSKeyValueChange`值，被影响对象的索引是一个 `NSIndexSet`对象。
下面的代码示范了在to-many关系`transactions`对象中的删除操作：
```
- (void)removeTransactionsAtIndexes:(NSIndexSet *)indexes {
    [self willChange:NSKeyValueChangeRemoval
        valuesAtIndexes:indexes forKey:@"transactions"];

    // Remove the transaction objects at the specified indexes.

    [self didChange:NSKeyValueChangeRemoval
        valuesAtIndexes:indexes forKey:@"transactions"];
}
```

## Registering Dependent Keys
>有一些属性的值取决于一个或者多个其他对象的属性值，一旦某个被依赖的属性值变了，依赖它的属性的变化也需要被通知。

### To-one Relationships
要自动触发 to-one关系，有两种方法：重写`keyPathsForValuesAffectingValueForKey:`方法或者定义名称为`keyPathsForValuesAffecting<Key>`的方法。
例如一个人的全名是由姓氏和名子组成的：
```
- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@",firstName, lastName];
}
```

一个观察`fullName`的程序在`firstName`或者`lastName`变化时也应该接收到通知。
一种解决方法是重写`keyPathsForValuesAffectingValueForKey:`方法来表明`fullname`属性是依赖于`firstname`和`lastname`的：
```
+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key {

    NSSet *keyPaths = [super keyPathsForValuesAffectingValueForKey:key];

    if ([key isEqualToString:@"fullName"]) {
        NSArray *affectingKeys = @[@"lastName", @"firstName"];
        keyPaths = [keyPaths setByAddingObjectsFromArray:affectingKeys];
    }
    return keyPaths;
}
```

相当于在影响`fullName`值的`keypath`中新加了两个key：`lastName`和`firstName`，很容易理解。
另一种实现同样结果的方法是实现一个遵循命名方式为`keyPathsForValuesAffecting<Key>`的类方法，`<Key>`是依赖于其他值的属性名（首字母大写），用上面代码的例子来重新实现一下：
```
+ (NSSet *)keyPathsForValuesAffectingFullName {
    return [NSSet setWithObjects:@"lastName", @"firstName", nil];
}
```

有时在`Caterory`中我们不能添加`keyPathsForValuesAffectingValueForKey:`方法，因为不能再`Category`中重写方法，所以这时可以实现`keyPathsForValuesAffecting<Key>`方法来代替。
> 你不能在`keyPathsForValuesAffectingValueForKey:`方法中设立to-many关系的依赖，相反，你必须观察在to-many集合中的每一个对象中相关的属性并通过亲自更新他们的依赖来回应变更。下一节将会讲述对付此情形的策略。

### To-many Relationships
`keyPathsForValuesAffectingValueForKey:`方法不支持包含to-many关系的`keypath`。比如，假如你有一个`Department`类，它有一个针对`Employee`类的to-many关系（雇员），`Employee`类有`salary`属性。你希望`Department`类有一个`totalSalary`属性来计算所有员工的薪水，也就是在这个关系中`Department`的`totalSalary`依赖于所有`Employee`的`salary`属性。你不能通过实现`keyPathsForValuesAffectingTotalSalary`方法并返回`employees.salary`。
有两种解决方法：
1. 你可以用KVO将`parent`（比如`Department`）作为所有`children`（比如`Employee`）相关属性的观察者。你必须在把`child`添加或删除到`parent`时也把`parent`作为`child`的观察者添加或删除。在`observeValueForKeyPath:ofObject:change:context:`方法中我们可以针对被依赖项的变更来更新依赖项的值：
```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {

    if (context == totalSalaryContext) {
        [self updateTotalSalary];
    }
    else
    // deal with other observations and/or invoke super...
}

- (void)updateTotalSalary {
    [self setTotalSalary:[self valueForKeyPath:@"employees.@sum.salary"]];
}

- (void)setTotalSalary:(NSNumber *)newTotalSalary {

    if (totalSalary != newTotalSalary) {
        [self willChangeValueForKey:@"totalSalary"];
        _totalSalary = newTotalSalary;
        [self didChangeValueForKey:@"totalSalary"];
    }
}

- (NSNumber *)totalSalary {
    return _totalSalary;
}
```

2. 如果你在使用`Core Data`，你可以在应用的`notification center`中将`parent`注册为它的`managed object context`的观察者，`parent`应该回应相应的变更通知，这些通知是`children`以类似KVO的形式发出的。
其实这也是Objective-C中利用Cocoa实现观察者模式的另一种途径：`NSNotificationCenter`

## Key-Value Observing Implementation Details
>自动键值观察通过`isa-swizzling`实现。

`isa`指针，从名称表明，指向持有分发表(dispatch table)的对象的类。这个分发表除了其他数据，包含类实现的方法的指针。
当一个观察者被注册到一个对象的属性，被观察的对象的`isa`指针就被转换，指向一个`过渡类`，而不是一个`真实的类`。结果`isa`指针的值没有映射实例真实的类。

>你不能依赖`isa`指针查明类关系。相反，你可以使用[Class](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Protocols/NSObject_Protocol/index.html#//apple_ref/occ/intfm/NSObject/class)方法查明对象实例的类。
```
-(Class)class
```

### 概览

这是怎么实现的呢？其实这都是通过Objective-C强大的运行时(runtime)实现的。当你第一次观察某个object时，runtime会创建一个新的继承原先class的subclass。在这个新的class中，它重写了所有被观察的key，然后将object的`isa`指针指向新创建的class（这个指针告诉Objective-C运行时某个object到底是哪种类型的object）。所以object神奇地变成了新的子类的实例。

这些被重写的方法实现了如何通知观察者们。当改变一个key时，会触发`setKey`方法，但这个方法被重写了，并且在内部添加了发送通知机制。（当然也可以不走setXXX方法，比如直接修改iVar，但不推荐这么做）。

有意思的是：苹果不希望这个机制暴露在外部。除了setters，这个动态生成的子类同时也重写了`-class`方法，依旧返回原先的class！如果不仔细看的话，被KVO过的object看起来和原先的object没什么两样。

### 深入探究
```Objective-C
// gcc -o kvoexplorer -framework Foundation kvoexplorer.m

#import <Foundation/Foundation.h>
#import <objc/runtime.h>


@interface TestClass : NSObject
{
    int x;
    int y;
    int z;
}
@property int x;
@property int y;
@property int z;
@end

@implementation TestClass
@synthesize x, y, z;
@end

static NSArray *ClassMethodNames(Class c)
{
    NSMutableArray *array = [NSMutableArray array];

    unsigned int methodCount = 0;
    Method *methodList = class_copyMethodList(c, &methodCount);
    unsigned int i;
    for(i = 0; i < methodCount; i++)
        [array addObject: NSStringFromSelector(method_getName(methodList[i]))];
    free(methodList);

    return array;
}

static void PrintDescription(NSString *name, id obj)
{
    NSString *str = [NSString stringWithFormat:
        @"%@: %@\n\tNSObject class %s\n\tlibobjc class %s\n\timplements methods <%@>",
        name,
        obj,
        class_getName([obj class]),
        class_getName(obj->isa),
        [ClassMethodNames(obj->isa) componentsJoinedByString:@", "]];
    printf("%s\n", [str UTF8String]);
}

int main(int argc, char **argv)
{
    [NSAutoreleasePool new];

    TestClass *x = [[TestClass alloc] init];
    TestClass *y = [[TestClass alloc] init];
    TestClass *xy = [[TestClass alloc] init];
    TestClass *control = [[TestClass alloc] init];

    [x addObserver:x forKeyPath:@"x" options:0 context:NULL];
    [xy addObserver:xy forKeyPath:@"x" options:0 context:NULL];
    [y addObserver:y forKeyPath:@"y" options:0 context:NULL];
    [xy addObserver:xy forKeyPath:@"y" options:0 context:NULL];

    PrintDescription(@"control", control);
    PrintDescription(@"x", x);
    PrintDescription(@"y", y);
    PrintDescription(@"xy", xy);

    printf("Using NSObject methods, normal setX: is %p, overridden setX: is %p\n",
          [control methodForSelector:@selector(setX:)],
          [x methodForSelector:@selector(setX:)]);
    printf("Using libobjc functions, normal setX: is %p, overridden setX: is %p\n",
          method_getImplementation(class_getInstanceMethod(object_getClass(control),
                                   @selector(setX:))),
          method_getImplementation(class_getInstanceMethod(object_getClass(x),
                                   @selector(setX:))));

    return 0;
}
```

首先定义了一个`TestClass`的类，它有3个属性。

然后定义了一些方便调试的方法。`ClassMethodNames`使用Objective-C运行时方法来遍历一个class，得到方法列表。注意，这些方法不包括父类的方法。`PrintDescription`打印object的所有信息，包括class信息（包括`-class`和通过运行时得到的class），以及这个class实现的方法。

然后创建了4个`TestClass`实例，每一个都使用了不同的观察方式。`x`实例有一个观察者观察`x`key，`y`, `xy`也类似。为了做比较，`z`key没有观察者。最后`control`实例没有任何观察者。

然后打印出4个objects的description。

之后继续打印被重写的setter内存地址，以及未被重写的setter的内存地址做比较。这里做了两次，是因为`-methodForSelector:`没能得到重写的方法。KVO试图掩盖它实际上创建了一个新的subclass这个事实！但是使用运行时的方法就原形毕露了。

### 运行代码
```
control: <TestClass: 0x104b20>
    NSObject class TestClass
    libobjc class TestClass
    implements methods <setX:, x, setY:, y, setZ:, z>
x: <TestClass: 0x103280>
    NSObject class TestClass
    libobjc class NSKVONotifying_TestClass
    implements methods <setY:, setX:, class, dealloc, _isKVOA>
y: <TestClass: 0x104b00>
    NSObject class TestClass
    libobjc class NSKVONotifying_TestClass
    implements methods <setY:, setX:, class, dealloc, _isKVOA>
xy: <TestClass: 0x104b10>
    NSObject class TestClass
    libobjc class NSKVONotifying_TestClass
    implements methods <setY:, setX:, class, dealloc, _isKVOA>
Using NSObject methods, normal setX: is 0x195e, overridden setX: is 0x195e
Using libobjc functions, normal setX: is 0x195e, overridden setX: is 0x96a1a550
```

首先，它输出了`control`object，没有任何问题，它的class是`TestClass`，并且实现了6个set/get方法。

然后是3个被观察的objects。注意`-class`仍然显示的是`TestClass`，使用`object_getClass`显示了这个object的真面目：它是`NSKVONotifying_TestClass`的一个实例。这个`NSKVONotifying_TestClass`就是动态生成的subclass！

注意，它是如何实现这两个被观察的setters的。你会发现，它很聪明，没有重写`-setZ:`，虽然它也是个setter，因为它没有被观察。同时注意到，3个实例对应的是同一个class，也就是说两个setters都被重写了，尽管其中的两个实例只观察了一个属性。这会带来一点效率上的问题，因为即使没有被观察的property也会走被重写的setter，但苹果显然觉得这比分开生成动态的subclass更好，我也觉得这是个正确的选择。

你会看到3个其他的方法。有之前提到过的被重写的`-class`方法，假装自己还是原来的class。还有`-dealloc`方法处理一些收尾工作。还有一个`_isKVOA`方法，看起来像是一个私有方法。

接下来，我们输出`-setX:`的实现。使用`-methodForSelector:`返回的是相同的值。因为`-setX:`已经在子类被重写了，这也就意味着`methodForSelector:`在内部实现中使用了`-class`，于是得到了错误的结果。

最后我们通过运行时得到了不同的输出结果。

作为一个优秀的探索者，我们进入debugger来看看这第二个方法的实现到底是怎样的：

```
(gdb) print (IMP)0x96a1a550
$1 = (IMP) 0x96a1a550 <_NSSetIntValueAndNotify>
```

看起来是一个内部方法，对`Foundation`使用`nm -a`得到一个完整的私有方法列表：

```
0013df80 t __NSSetBoolValueAndNotify
000a0480 t __NSSetCharValueAndNotify
0013e120 t __NSSetDoubleValueAndNotify
0013e1f0 t __NSSetFloatValueAndNotify
000e3550 t __NSSetIntValueAndNotify
0013e390 t __NSSetLongLongValueAndNotify
0013e2c0 t __NSSetLongValueAndNotify
00089df0 t __NSSetObjectValueAndNotify
0013e6f0 t __NSSetPointValueAndNotify
0013e7d0 t __NSSetRangeValueAndNotify
0013e8b0 t __NSSetRectValueAndNotify
0013e550 t __NSSetShortValueAndNotify
0008ab20 t __NSSetSizeValueAndNotify
0013e050 t __NSSetUnsignedCharValueAndNotify
0009fcd0 t __NSSetUnsignedIntValueAndNotify
0013e470 t __NSSetUnsignedLongLongValueAndNotify
0009fc00 t __NSSetUnsignedLongValueAndNotify
0013e620 t __NSSetUnsignedShortValueAndNotify
```

这个列表也能发现一些有趣的东西。比如苹果为每一种primitive type都写了对应的实现。Objective-C的object会用到的其实只有`__NSSetObjectValueAndNotify`，但需要一整套来对应剩下的，而且看起来也没有实现完全，比如`long dobule`或`_Bool`都没有。甚至没有为通用指针类型(generic pointer type)提供方法。所以，不在这个方法列表里的属性其实是不支持KVO的。

KVO是一个很强大的工具，有时候过于强大了，尤其是有了自动触发通知机制。现在你知道它内部是怎么实现的了，这些知识或许能帮助你更好地使用它，或在它出错时更方便调试。

可以看一下另一篇文章[Key-Value Observing Done Right](https://www.mikeash.com/pyblog/key-value-observing-done-right.html).




## 调试KVO
在 lldb 里查看一个被观察对象的所有观察信息。
```
(lldb) po [observedObject observationInfo]
```
这会打印出有关谁观察谁之类的很多信息。
这个信息的格式不是公开的，不能让任何东西依赖它，因为苹果随时都可以改变它。不过这是一个很强大的排错工具。

参考：

[Introduction to Key-Value Observing Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177-BCICJDHA)

[Objective-C中的KVC和KVO](http://yulingtianxia.com/blog/2014/05/12/objective-czhong-de-kvche-kvo/)

[KVO的内部实现](http://limboy.me/ios/2013/08/05/internal-implementation-of-kvo.html)

[Friday Q&A 2009-01-23](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)
