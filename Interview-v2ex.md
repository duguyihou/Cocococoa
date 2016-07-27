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
7. Percentage encoding and decoding(URL encoding)
8. Reverse an array
```objc
NSArray *array = [@[@"a", @"b"] reverseObjectEnumerator].allObjects;
```
9. Filter objects in an array by value of a property
```objc
@interface MyObject : NSObject
@property (nonatomic, assign, readonly) BOOL favorited;
@end
```
Given an NSArray instance containing several MyObject objects, put the objects of which favorited property are YES into a new NSArray instance.
```objc
NSArray<MyObject *> *array = @[@"a", @"b"];
    NSPredicate *predicate = [NSPredicate predicateWithBlock:^BOOL(id  _Nonnull evaluatedObject, NSDictionary<NSString *,id> * _Nullable bindings) {
        return evaluatedObject.favorited;
    }];
    NSArray *array2 = [array filteredArrayUsingPredicate:predicate];
```
10. Remove duplicated objects from an array
```objc
NSArray *myArray = @[@"a", @"b", @"c", @"a", @"d"];
```
Create a new NSArray instance from myArray containing @"a", @"b", @"c", @"d" only.
```objc
NSArray *myArray = @[@"a", @"b", @"c", @"d", @"a"];
NSArray *array = [array valueForKeyPath:@"@unionOfArray.self"];
NSArray *array1 = [NSOrderedSet orderedSetWithArray:duplicatedArray].array;  
```
11. Determine if an NSDate instance is in this month
```objc
NSDate *date = [NSDate date];
NSDate *date2 = [NSDate date];
NSCalendar *calendar = [NSCalendar currentCalendar];

NSDateComponents *components = [calendar component:NSCalendarUnitMonth fromDate:date];
NSDateComponents *components2 = [calendar component:NSCalendarUnitMonth fromDate:date2];

BOOL sameMonth = (components.month == components2.month);

```
