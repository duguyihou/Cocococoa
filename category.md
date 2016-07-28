## Category

Category用于向已经存在的类添加方法从而达到扩展已有类的目的，在很多情形下Category也是比创建子类更优的选择。 Category用于大型类有效分解。新添加的方法会被被扩展的类的所有子类自动继承。 Category也可以用于替代这个已有类中某个方法的实体，从而达到修复BUG的目的。 如此就不能去调用已有类中原有的那个被替换掉方法实体了。需要注意的是，当准备有Category来替换某一个方法的时候， 一定要保证实现原来方法的所有功能，否则这种替代就是没有意义而且会引起新的BUG。

Category的方法不一定非要在@implementation中实现，也可以在其他位置实现， 但是当调用Category的方法时，依据继承树没有找到该方法的实现，程序则会崩溃。Category理论上不能添加变量， 但是可以使用@dynamic 来弥补这种不足。 ```objc @implementation NSObject (Category) @dynamic variable;

- (id) variable { return objc_getAssociatedObject(self, externVariableKey); }
- (void)setVariable:(id) variable { objc_setAssociatedObject(self, externVariableKey, variable, OBJC_ASSOCIATION_RETAIN_NONATOMIC); }

````
和子类不同的是，Category不能用于向被扩展类添加实例变量。Category通常作为一种组织框架代码的工具来使用。
如果需要添加一个新的变量，则需添加子类。如果只是添加一个新的方法，用Category是比较好的选择。

### 在Category中实现属性
做开发时我们常常会需要在已经实现了的类中增加一些方法，这时候我们一般会用Category的方式来做。但是这样做我们也只能扩展一些方法，而有时候我们更多的是想给它增加一个属性。由于类已经是编译好的了，就不能静态的增加成员了，这样我们就需要自己来实现getter和setter方法了，在这些方法中动态的读写属性变量来实现属性。一种比较简单的做法是使用Objective-C运行时的这两个方法：
```objc
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
id objc_getAssociatedObject(id object, const void *key);
```
这两个方法可以让一个对象和另一个对象关联，就是说一个对象可以保持对另一个对象的引用，并获取那个对象。有了这些，就能实现属性功能了。
policy可以设置为以下这些值：
```objc
enum {
    OBJC_ASSOCIATION_ASSIGN = 0,
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,
    OBJC_ASSOCIATION_RETAIN = 01401,
    OBJC_ASSOCIATION_COPY = 01403
};
```
这些值跟属性定义中的nonatomic，copy，retain等关键字的功能类似。
下面是一个属性自定义getter和setter的例子：
```objc
NSString const * kExposeController = @"exposeController";

- (UIViewController *)exposeController {
    return (UIViewController *)objc_getAssociatedObject(self, kExposeController);
}

- (void)setExposeController:(UIViewController *)exposeController {
    objc_setAssociatedObject(self, kExposeController, exposeController, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```
可以看出使用objc_setAssociatedObject和objc_getAssociatedObject函数可以很方便的实现属性的getter和setter。
**一个很方便的宏**
为此，我特意写了一个Synthesize宏，可以提供@synthesize类似的功能。可以支持两种最常用的属性：非原子retain和assign属性（如果需要其他类型的属性可自行修改）。
```objc
#import <objc/runtime.h>
#define SYNTHESIZE_CATEGORY_OBJ_PROPERTY(propertyGetter, propertySetter)
- (id) propertyGetter {
    return objc_getAssociatedObject(self, @selector( propertyGetter ));
}
- (void) propertySetter (id)obj{
    objc_setAssociatedObject(self, @selector( propertyGetter ), obj, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}


#define SYNTHESIZE_CATEGORY_VALUE_PROPERTY(valueType, propertyGetter, propertySetter)
- (valueType) propertyGetter {
    valueType ret = {0};
    [objc_getAssociatedObject(self, @selector( propertyGetter )) getValue:&ret];
    return ret;
}
- (void) propertySetter (valueType)value{
    NSValue *valueObj = [NSValue valueWithBytes:&value objCType:@encode(valueType)];
    objc_setAssociatedObject(self, @selector( propertyGetter ), valueObj, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```
用这个宏只需要指定相关属性的类型，getter和setter就可以快速的实现一个属性。比如在UIAlert的Category实现一个非原子retain属性userInfo，以及一个assign的类型为CGRect的customArea属性：

UIAlertView+Ex.h
```objc
@interface UIAlertView (Ex)
@property(nonatomic, retain) id userInfo;
@property(nonatomic) CGRect customArea;
@end
```
UIAlertView+Ex.m
```objc
@implementation UIAlertView (Ex)
SYNTHESIZE_CATEGORY_OBJ_PROPERTY(userInfo, setUserInfo:)
SYNTHESIZE_CATEGORY_VALUE_PROPERTY(CGRect, customArea, setCustomArea:)
@end
```
**runtime对category的加载过程**
下面是runtime中category的结构：
```objc
struct _category_t {
const char *name; // 类的名字
struct _class_t *cls; // 要扩展的类对象，编译期间这个值是不会有的，
在app被runtime加载时才会根据name对应到类对象
const struct _method_list_t *instance_methods; // 实例方法
const struct _method_list_t *class_methods; // 类方法
const struct _protocol_list_t *protocols; // 这个category实现的protocol，
比较不常用在category里面实现协议，但是确实支持的
const struct _prop_list_t *properties; // 这个category所有的property，
这也是category里面可以定义属性的原因，不过这个property不会@synthesize实例变量，
一般有需求添加实例变量属性时会采用objc_setAssociatedObject和objc_getAssociatedObject方法绑定方法绑定，
不过这种方法生成的与一个普通的实例变量完全是两码事。
};
````

category动态扩展了原来类的方法，在调用者看来好像原来类本来就有这些方法似的， 不论有没有import category 的.h，都可以成功调用category的方法，都影响不到category的加载流程， import只是帮助了编译检查和链接过程。runtime加载完成后，category的原始信息在类结构里将不会存在。

objc runtime的加载入口是一个叫_objc_init的方法，在library加载前由libSystem dyld调用， 进行初始化操作。调用map_images方法将文件中的image map到内存。调用_read_images方法初始化map后的image， 这里面干了很多的事情，像load所有的类、协议和category，著名的+ load方法就是这一步调用的。 category的初始化，循环调用了_getObjc2CategoryList方法。

在调用完_getObjc2CategoryList后，runtime终于开始了category的处理，首先分成两拨， 一拨是实例对象相关的调用addUnattachedCategoryForClass，一拨是类对象相关的调用addUnattachedCategoryForClass， 然后会调到attachCategoryMethods方法，这个方法把一个类所有的category_list的所有方法取出来组成一个method_list_t ， 这里是倒序添加的，也就是说，新生成的category的方法会先于旧的category的方法插入。

生成了所有method的list之后，调用attachMethodLists将所有方法前序添加进类的方法的数组中， 也就是说，如果原来类的方法是a,b,c，类别的方法是1,2,3，那么插入之后的方法将会是1,2,3,a,b,c， 也就是说，原来类的方法被category的方法覆盖了，但被覆盖的方法确实还在那里。

```objc
static void attachCategoryMethods(class_t *cls, category_list *cats,
                  BOOL *inoutVtablesAffected)
{
if (!cats) return;
if (PrintReplacedMethods) printReplacements(cls, cats);

BOOL isMeta = isMetaClass(cls);
method_list_t **mlists = (method_list_t **)
    _malloc_internal(cats->count * sizeof(*mlists));

// Count backwards through cats to get newest categories first
int mcount = 0;
int i = cats->count;
BOOL fromBundle = NO;
while (i--) {
    method_list_t *mlist = cat_method_list(cats->list[i].cat, isMeta);
    if (mlist) {
        mlists[mcount++] = mlist;
        fromBundle |= cats->list[i].fromBundle;
    }
}

attachMethodLists(cls, mlists, mcount, NO, fromBundle, inoutVtablesAffected);

_free_internal(mlists);

}
```

这也即是我们上面说的Category修复Bug的原理。 **Extension** Extension非常像是没有命名的类别。扩展只是用来定义类的私有方法的，实现要在原始的.m里面。 还以用来改变原始属性的一些性质。一般的时候，Extension都是放在.m文件中@implementation的上方。 Extension中的方法必须在@implementation中实现，否则编译会报错。Category没有源代码的类添加方法， 格式：定义一对.h和.m。Extension作用于管理类的所有方法，格式：把代码写到原始类的.m文件中。

### 类别(category)的作用?继承和类别在实现中有何区别?

category 可以在不获悉，不改变原来代码的情况下往里面添加新的方法，只能添加，不能删除修改， 并且如果类别和原来类中的方法产生名称冲突，则类别将覆盖原来的方法，因为类别具有更高的优先级。

类别主要有3个作用：

1. 将类的实现分散到多个不同文件或多个不同框架中。
2. 创建对私有方法的前向引用。
3. 向对象添加非正式协议。 继承可以增加，修改或者删除方法，并且可以增加属性。

### 类别和类扩展的区别

  category和extensions的不同在于 后者可以添加属性。另外后者添加的方法是必须要实现的。 extensions可以认为是一个私有的Category。
