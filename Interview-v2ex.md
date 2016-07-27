# Interview-v2ex
>原链接：https://www.v2ex.com/t/289580

1. Declare an NS_OPTIONS type BCLayoutAxis with following values:
* none
* horizontal
* vertical
* all
```objc
typedef NS_OPTIONS(NSUInteger, BCLayoutAxis) {
BCLayoutAxisNone       = 1 << 0,
BCLayoutAxisHorizontal = 1 << 1,
BCLayoutAxisVertical   = 1 << 2,
BCLayoutAxisAll        = (BCLayoutAxisHorizontal | BCLayoutAxisVertical)
};
```
2. Create string representations for the values above
For debugging, we need string representations ("none", "horizontal", "vertical", "all") instead of original integer values. Use a graceful approach to represent them.
```objc
NSString * const BCLayoutAxisDescription[] = {
[BCLayoutAxisNone]                         = @"BCLayoutAxisNone",
[BCLayoutAxisHorizontal]                   = @"BCLayoutAxisHorizontal",
[BCLayoutAxisVertical]                     = @"BCLayoutAxisVertical",
[BCLayoutAxisAll]                          = @"BCLayoutAxisAll"
};
```
3. Declare a constant value (public/private)
Declare a constant named kBCMyConstant of NSString type with value of myConstantValue, public and private.
.h
```objc
extern NSString * const kBCMyConstant;
```
.m
```objc
NSString * const kBCMyConstant = @"XXXX";
```

4. Create variadic method
5. Create a singleton
```objc
+ (instancetype)sharedInstance {
    static NSObject *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [NSObject new];
    });
    return sharedInstance;
}
```
6. Concatenate string literals
```objc
NSString *string = [NSString stringWithFormat:@"%@%@", @"a", @"b"];
NSString *string = [@"a" stringByAppendingString:@"b"];
```
7. Percentage encoding and decoding
8. Reverse an array
```objc
NSArray *array = [@[@"a", @"b"] reverseObjectEnumerator].allObjects;
```
9. Filter objects in an array by value of a property
