# 一道简单的iOS面试题
微信公众号找到的：[一道简单的iOS面试题](http://mp.weixin.qq.com/s?__biz=MzIxMjQ5NzcyMQ==&mid=2247483658&idx=1&sn=ac35fe51f05dd4f871e683b3a9e40034&scene=2&srcid=0721MylnHCei6HEA2C0iHGds&from=timeline&isappinstalled=0%23wechat_redirect)

```objc
- (init)findMaxNumber:(NSArray *)numbers {
  int maxNumber;
  [numbers enumerateObjectsUsingBlock:^(NSNumber* _Nonnull number, NSUInteger idx, BOOL * _Nonnull stop) {
    int intNumber = number.intValue;
    if (intNumber > maxNumber) {
      maxNumber = intNumber;
    }
  }];
  return maxNumber;
}
```

## 初级问题

1. maxNumber没有初始值。如果numbers里面都是负数的话，返回值为0，结果不对。
解释：这个问题是必须要回答出来的，读代码是最基本的能力。

2. maxNumber变量没有加 __block修饰。这其实是一段编译不过的代码。
解释：如果这个答不上也不要紧，经过提示让面试者答出block中不能修改变量值就行了。

## 进阶问题

1. 函数代码位置有问题
解释：一个求数组最大值的函数，是很有可能被复用的，这段代码应该写在项目的基础公共类库中。可以写成一个公共类的类函数，或者一个数组的category等等，题目里写在一个VC中就很容易出现重复造轮子的现象了。

2. 这个函数设计的有问题，返回值预期并不明确
解释：很多面试者都会提到，检查输入的numbers是不是为空，如果为空了怎么做呢？返回0还是-1？或者是NSIntegerMin？调用的人可能也会有此疑惑，所以我更希望面试者将这个函数改为
       `-(NSInteger)findMaxNumberIndex:(NSArray*)array`
   或者  
       `-(NSNumber*)findMaxNumber:(NSArray*)array`
这样的函数，可以让调用者更容易预测该函数的运作。


## 优化部分

1. 函数效率还可以优化
解释：关于这一点，其实很多人觉得iOS上即使有这种数组也不会很大的量，找个最大值也不会很慢。但是纯粹为了开发速度而且数组长度已知很少的业务场景下，我完全不介意使用[numbers valueForKeyPath:@"@max.intValue"]这样的方式，而且坑更少，甚至都不用额外写个函数了。但是如果要写一个单独的函数来处理还是用for循环吧，毕竟可以节省一个block对象。

2. 参数的处理的优化
解释：万一一堆int里面混了两个float怎么办?  这个float又是最大的怎么办?  其实只需要使用[NSNumber compare:]方法就好了，管他里面是什么类型呢。

3.其他比较散的优化点，例如这个数组来源是不是JSON啊，是不是支持字符串的数字啊，外部调用如何更好看等等，能答上的当然越多越好。

整理了以上思路，再权衡性能的情况下并未处理数字字符串，多线程等情况，修改后的代码如下，希望可以给大家带来一些帮助。
```objc
@interface NSArray(findMaxNumber)
@property (nonatomic, readOnly) NSNumber *maxNumber;
@end

@Implementation NSArray(findMaxNumber)

- (NSNumber *)maxNumber {
  NSNumber *maxNumber = nil;
  Class numberClass = [NSNumber class];
  for (NSNumber *num in self) {
    if (![num isKindOfClass:numberClass]) {
      continue;
    }
    if ([maxNumber compare:num] != NSOrderedDescending) {
      maxNumber = num;
    }
  }
  return maxNumber;
}

@end

int main () {
  NSArray *numbers = @[@1, @2, @10, @11.1, @-1000, @"Hello World"];
  NSLog(@"%@", numbers.maxNumber);

  return 0;
}
```
