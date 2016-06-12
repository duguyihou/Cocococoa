# Protocol
> 简单来说，Protocol不属于任何一个类，它只是一个方法列表，任何类都可以对其中声明的方法进行实现。这种设计模式一般称为代理模式（delegation）。你可以通过Protocol定义各种行为，在不同的场景采用不同的实现方式。在iOS和OS X开发中，Apple采用了大量的代理模式来实现MVC中View和Controller的解耦。

## 使用方法
Protocol有两种声明的方式：
* 在单独的声明文件（.h文件）中声明。
* 在某个类的声明的文件中声明。

以上两种方式视具体情况而定，但是在代码规范上都是一致的：
```
@protocol HandleDeckDelegate <NSObject>

@required
- (void)ShuffleDeck;

@optional
- (void)CuttingDeck;

@end
```

上述代码中有两个关键字，`@required`和`@optional`，表示如果要实现这个协议，那么ShuffleDeck方法是必须要实现的，CuttingDeck则是可选的，如果不注明，那么方法默认是@required的，必须实现。

那么如何实现这个Protocol呢，很简单，创建一个普通的Objective-C类，如果Protocol使用单独的.h文件声明，那么在该类的.h声明文件中引入包含Protocol的.h文件，如果Protocol是声明在一个相关类中，那么就需要引入该类的.h文件。之后声明采用这个Protocol即可。
```
// Deck.h

#import <Foundation/Foundation.h>
#import "Card.h"
#import "HandleDeckDelegate.h"

@interface Deck : NSObject<HandleDeckDelegate>

- (Card *)randomDrawCard;

@end
```
用尖括号（<…>）括起来的HandleDeckDelegate就是我们创建的Protocol。如果要采用多个Protocol，可以在尖括号内引入多个Protocol名称，并用逗号隔开即可。例如<HandleDeckDelegate,xxxDelegate>。

```
// Deck.m

#import "Card.h"

@implementation Deck

- (Card *)drawCardFromTop
{
    //TODO.....
}

- (void)ShuffleDeck
{
    //TODO.....
}

@end
```
## 使用场景

* Objective-C里的Protocol和Java语言中的接口很类似，如果一些类之间没有继承关系，但是又具备某些相同的行为，则可以使用Protocol来描述它们的关系。
* 不同的类，可以遵守同一个Protocol，在不同的场景下注入不同的实例，实现不同的功能。
## 需要注意的问题
* 根据约定，框架中后缀为Delegate的都是Protocol，例如UIApplicationDelegate，UIWebViewDelegate等。
* Protocol本身是可以继承的，比如：
```
@protocol A
    -(void)methodA;
@end

@protocol B <A>
    -(void)methodB;
@end
```

如果你要实现B，那么methodA和methodB都需要实现。
* Protocol是与任何类都无关的，任何类都可以实现定义好的Protocol，如果我们想知道某个类是否实现了某个Protocol，那么我们可以用conformsToProtocol方法进行判断：
```
[obj conformsToProtocol:@protocol(ProcessDataDelegate)]

```

>Protocol最常用的就是委托代理模式，Cocoa框架中大量采用了这种模式实现数据和UI的分离。例如UIView产生的所有事件，都是通过委托的方式交给Controller完成。
