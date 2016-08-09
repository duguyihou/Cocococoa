# @property
## @property是什么
>@Property是声明属性的语法，它可以快速方便的为实例变量创建存取器，并允许我们通过点语法使用存取器。(存取器（accessor）：指用于获取和设置实例变量的方法。用于获取实例变量值的存取器是getter，用于设置实例变量值的存取器是setter。)

## 创建存取器
### 手工创建存取器
先看两段代码：
```
// Car.h

#import <Foundation/Foundation.h>

@interface Car : NSObject
{
    // 实例变量
    NSString *carName;
       NSString *carType;
}

// setter                                    
- (void)setCarName:(NSString *)newCarName;

// getter
- (NSString *)carName;

// setter
- (void)setCarType:(NSString *)newCarType;

// getter
- (NSString *)carType;

@end  
```

上面的代码中carName和carType就是Car的实例变量，并且可以看到分别对这两个实例变量声明了get/set方法，即存取器。
```
#import "Car.h"

@implementation Car

// setter
- (void)setCarName:(NSString *)newCarName
{
    carName = newCarName;
}

// getter
- (NSString *)carName
{
    return carName;
}

// setter
- (void)setCarType:(NSString *)newCarType
{
    carType = newCarType;
}

// getter
- (NSString *)carType
{
    return carType;
}

@end
```

上面代码是对实例变量存取器的实现。我们可以看到，存取器就是对实例变量进行赋值和取值。按照约定赋值方法以set开头，取值方法以实例变量名命名。
我们看看如何使用：
```
// main.m

#import "Car.h"

int main(int argc, char * argv[])
{
    @autoreleasepool {
        Car *car = [[Car alloc] init];
        [car setCarName:@"Jeep Cherokee"];
        [car setCarType:@"SUV"];
        NSLog(@"The car name is %@ and the type is %@",[car carName],[car carType]);      
       }
    return 0;
}
```

上面的代码中我们注意到，对象Car使用了消息语法，也就是使用方括号的语法给存取器发送消息。
返回结果为：
```
The car name is Jeep Cherokee and the type is SUV
```

### 使用@Property创建存取器
```
// Car.h

#import <Foundation/Foundation.h>

@interface Car : NSObject
{
    // 实例变量
    NSString *carName;
    NSString *carType;
}

@property(nonatomic,strong) NSString *carName;
@property(nonatomic,strong) NSString *carType;

@end
```
上面代码中，我们使用@property声明两个属性，名称与实例变量名称相同。
```
// Car.m

#import "Car.h"

@implementation Car

@synthesize carName;
@synthesize carType;

@end
```

在.m文件中我们使用@synthesize自动生成这两个实例变量的存取器，并且隐藏了存取器，虽然我们看不到存取器，但它们确实是存在的。
```
// main.m

int main(int argc, char * argv[])
{
    @autoreleasepool {
        Car *car = [[Car alloc] init];
        car.carName = @"Jeep Compass";
        car.carType = @"SUV";
        NSLog(@"The car name is %@ and the type is %@",car.carName,car.carType);
    }
    return 0;
}
```

在上面的代码中我们可以注意到，Car对象使用点语法给存取器发送消息，并且get与set的语法是相同的，所以这里的点语法可以根据语境判断我们是要赋值还是取值。

当然我们也依然可以使用消息语法来使用：
```
// main.m

int main(int argc, char * argv[])
{
    @autoreleasepool {
        Car *car = [[Car alloc] init];

        // 点语法
//        car.carName = @"Jeep Compass";
//        car.carType = @"SUV";
//        NSLog(@"The car name is %@ and the type is %@",car.carName,car.carType);

        // 消息语法
        [car setCarName:@"Jeep Compass"];
        [car setCarType:@"SUV"];
        NSLog(@"The car name is %@ and the type is %@",[car carName],[car carType]);       
    }
    return 0;
}
```
上面两段代码的执行结果都是：
```
The car name is Jeep Compass and the type is SUV
```

>总结：`@property`等同于在.h文件中声明实例变量的get/set方法，`@synthesize`等同于在.m文件中实现实例变量的get/set方法。使用`@property`和`synthesize`创建存取器要比手动声明两个存取方法（`getter`和`setter`）更简单。而且我们在使用属性时可以使用点语法赋值或取值，语法更简单，更符合面向对象编程。

### 不必单独声明实例变量
如果使用@Property，就不必单独声明实例变量了。因为在没有显示提供示例变量声明的前提下，系统会自动帮你生成实例变量。我们通过以下代码来说明：
```
// Car.h

#import <Foundation/Foundation.h>

@interface Car : NSObject

@property(nonatomic,strong) NSString *carName;
@property(nonatomic,strong) NSString *carType;

- (NSString *)carInfo;

@end
```
在.h文件中我们并没有声明实例变量，只是声明了carName和carType两个属性，以及一个`carInfo`方法，返回值为`NSString *`。
```
// Car.m

#import "Car.h"

@implementation Car

- (NSString *)carInfo
{
    return [NSString stringWithFormat:@"The car name is %@ and the type is %@",_carName,_carType];
}

@end
```

在.m文件中我们可以注意到，在carInfo方法中我们使用了_carName和_carType实例变量，这就是当我们没有显示声明实例变量时，系统为我们自动生成的。命名规则是以_为前缀，加上属性名，即_propertyName。

其实在.m文件中实际是存在@synthesize声明语句的，只是系统将其隐藏了：
```
@synthesize carName = _carName;
@synthesize carType = _carType;
```

那么如果我们不喜欢默认的实例变量命名方法，或者我们希望使用更有语义的名称.

```
// Car.m

#import "Car.h"

@implementation Car

@synthesize carName = i_am_car_name;
@synthesize carType = i_am_car_type;

- (NSString *)carInfo
{
    return [NSString stringWithFormat:@"The car name is %@ and the type is %@",i_am_car_name,i_am_car_type];
}

@end
```

通过上述代码可以看到，我们只需要通过@synthesize来声明我们希望的实例变量名。
> 总结：如果我们希望使用默认的实例变量命名方式，那么我们在.m文件中就不需要使用`@synthesize`声明，系统会帮我们自动完成。如果我们希望自己命名实例变量命，那么我们就使用`@synthesize`显示声明我们希望的实例变量名。

## @property的特性
@property还有一些关键字，它们都是有特殊作用的，比如上述代码中的nonatomic，strong：
```
@property(nonatomic,strong) NSString *carName;
@property(nonatomic,strong) NSString *carType;
```
>它们分为三类，分别是：
* 原子性
* 存取器控制
* 内存管理。

### 原子性
* atomic（默认）：atomic意为操作是原子的，意味着只有一个线程访问实例变量。atomic是线程安全的，至少在当前的存取器上是安全的。它是一个默认的特性，但是很少使用，因为比较影响效率，这跟ARM平台和内部锁机制有关。
* nonatomic：nonatomic跟atomic刚好相反。表示非原子的，可以被多个线程访问。它的效率比atomic快。但不能保证在多线程环境下的安全性，在单线程和明确只有一个线程访问的情况下广泛使用。

### 存取器控制
* readwrite（默认）：readwrite是默认值，表示该属性同时拥有setter和getter。
* readonly： readonly表示只有getter没有setter。
有时候为了语意更明确可能需要自定义访问器的名字：
```
@property (nonatomic, setter = mySetter:,getter = myGetter ) NSString *name;
```
最常见的是BOOL类型，比如标识View是否隐藏的属性hidden。可以这样声明：
```
@property (nonatomic,getter = isHidden ) BOOL hidden;
```

### 内存管理
@property有显示的内存管理策略。这使得我们只需要看一眼@property声明就明白它会怎样对待传入的值。
* assign（默认）：assign用于值类型，如int、float、double和NSInteger，CGFloat等表示单纯的复制。还包括不存在所有权关系的对象，比如常见的delegate。
```
@property(nonatomic) int running;
```

```
@property(nonatomic,assign) int running;

```
以上两段代码是相同的。

在`setter`方法中，采用直接赋值来实现设值操作：
```
-(void)setRunning:(int)newRunning{  
    _running = newRunning;  
}
```
* retain：在setter方法中，需要对传入的对象进行引用计数加1的操作。
简单来说，就是对传入的对象拥有所有权，只要对该对象拥有所有权，该对象就不会被释放。如下代码所示：
```
-(void)setName:(NSString*)_name{  
     //首先判断是否与旧对象一致，如果不一致进行赋值。  
     //因为如果是一个对象的话，进行if内的代码会造成一个极端的情况：当此name的retain为1时，使此次的set操作让实例name提前释放，而达不到赋值目的。  
     if ( name != _name){  
          [name release];  
          name = [_name retain];  
     }  
}
```
* strong：strong是在IOS引入ARC的时候引入的关键字，是retain的一个可选的替代。表示实例变量对传入的对象要有所有权关系，即强引用。strong跟retain的意思相同并产生相同的代码，但是语意上更好更能体现对象的关系。
* weak：在setter方法中，需要对传入的对象不进行引用计数加1的操作。
简单来说，就是对传入的对象没有所有权，当该对象引用计数为0时，即该对象被释放后，用weak声明的实例变量指向nil，即实例变量的值为0。

>注：weak关键字是IOS5引入的，IOS5之前是不能使用该关键字的。delegate 和 Outlet 一般用weak来声明。

* copy：与strong类似，但区别在于实例变量是对传入对象的副本拥有所有权，而非对象本身。

## assign 与weak、 __block 与 __weak、strong 与copy的区别

### assign 与weak区别
* assign适用于基本数据类型，weak是适用于NSObject对象，并且是一个弱引用。
>* assign其实也可以用来修饰对象。那么我们为什么不用它修饰对象呢？因为被assign修饰的对象（一般编译的时候会产生警告：Assigning retained object to unsafe property; object will be released after assignment）在释放之后，指针的地址还是存在的，也就是说指针并没有被置为nil，造成野指针。对象一般分配在堆上的某块内存，如果在后续的内存分配中，刚好分到了这块地址，程序就会崩溃掉。
>* 那为什么可以用assign修饰基本数据类型？因为基础数据类型一般分配在栈上，栈的内存会由系统自己自动处理，不会造成野指针。

> weak修饰的对象在释放之后，指针地址会被置为nil。所以现在一般弱引用就是用weak。weak使用场景：
>* 在ARC下,在有可能出现循环引用的时候，往往要通过让其中一端使用weak来解决，比如: delegate代理属性，通常就会声明为weak。
>* 自身已经对它进行一次强引用，没有必要再强引用一次时也会使用weak。比如：自定义 IBOutlet控件属性一般也使用weak，当然也可以使用strong。

### strong 与copy的区别

trong 与copy都会使引用计数加1，但strong是两个指针指向同一个内存地址，copy会在内存里拷贝一份对象，两个指针指向不同的内存地址

### __block与__weak的区别

* __block是用来修饰一个变量，这个变量就可以在block中被修改

__block：使用 __block修饰的变量在block代码块中会被retain（ARC下会retain，MRC下不会retain）
* _weak：使用__weak修饰的变量不会在block代码块中被retain
同时，在ARC下，要避免block出现循环引用 __weak typedof(self)weakSelf = self;

###  block变量定义时为什么用copy？block是放在哪里的？
block本身是像对象一样可以retain，和release。但是，block在创建的时候，它的内存是分配在栈(stack)上，可能被随时回收，而不是在堆(heap)上。他本身的作于域是属于创建时候的作用域，一旦在创建时候的作用域外面调用block将导致程序崩溃。通过copy可以把block拷贝（copy）到堆，保证block的声明域外使用。
>特别需要注意的地方就是在把block放到集合类当中去的时候，如果直接把生成的block放入到集合类中，是无法在其他地方使用block，必须要对block进行copy。
```
[array addObject:[[^{
    NSLog(@"hello!");
} copy] autorelease]];
```
### block 为什么不用strong？
block的循环引用并不是strong导致的…在ARC环境下，系统底层也会做一次copy操作使block从栈区复制一块内存空间到堆区…所以strong和copy在对block的修饰上是没有本质区别的，只不过copy操作效率高而已。
