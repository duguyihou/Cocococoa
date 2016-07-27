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

.h
```objc
-(void)yourMethods:(id)string1,...;
```
.m
```objc
-(void)yourMethods:(id)string1, ...{

    NSMutableArray *arguments=[[NSMutableArray alloc]initWithArray:nil];
    id eachObject;
    va_list argumentList;
    if (string1)
    {
        [arguments addObject: string1];
        va_start(argumentList, string1);
        while ((eachObject = va_arg(argumentList, id)))    
        {
             [arguments addObject: eachObject];
        }
        va_end(argumentList);        
     }
    NSLog(@"%@",arguments);
}
```
call the method
```objc
[self yourMethods:@"ab",@"cd",@"ef",@"gf",nil];
```
>remember to put nil at the end


```objc
- (NSString *) append:(NSString *)list, ...
{
    NSMutableString * res = [NSMutableString string];
    [res appendString:list];

    va_list args;
    va_start(args, list);
    id arg = nil;

    while(( arg = va_arg(args, id))){
        [res appendString:arg];
    }
    va_end(args);
    return res;
}

- (void) test_va_arg
{
    NSString * t = [self append:@"a", @"b", @"c", nil];
    STAssertEqualObjects(@"abc", t, @"");
}
```
参考：
http://stackoverflow.com/questions/4804674/how-to-create-variable-argument-methods-in-objective-c
http://stackoverflow.com/questions/12454408/variable-number-of-method-parameters-in-objective-c-need-an-example
http://www.cocoabuilder.com/archive/cocoa/125332-variadic-arguments-to-methods-in-objective-how.html
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
```objc
NSString *sampleUrl = @"http://www.google.com/search.jsp?params=Java Developer";
NSString* encodedUrl = [sampleUrl stringByAddingPercentEscapesUsingEncoding:
 NSUTF8StringEncoding];
```
```objc
[NSURL URLWithString:[string stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding]];
```
http://stackoverflow.com/questions/8086584/objective-c-url-encoding
http://stackoverflow.com/questions/8088473/how-do-i-url-encode-a-string
http://stackoverflow.com/questions/3418754/how-to-prepare-an-nsurl-from-an-nsstring-continaing-international-characters
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
# Runtime
1. Swizzle a method
```objc
- (void)myMethod;
- (void)bc_myMethod;
```

```objc
+ (void)load {
    static dispatch_once_t oncoToken;
    dispatch_once (&oncoToken, ^{
        Class class = [self class];

        SEL mySelector = @selector(myMethod);
        SEL bc_mySelector = @selector(bc_myMethod);

        Method myMethod = class_getInstanceMethod(class, mySelector);
        Method swizzledMethod = class_getInstanceMethod(class, bc_mySelector);

            // When swizzling a class method, use the following:
            // Class class = object_getClass((id)self);
            // ...
            // Method originalMethod = class_getClassMethod(class, originalSelector);
            // Method swizzledMethod = class_getClassMethod(class, swizzledSelector);
        BOOL didAddMethod = class_addMethod(class,
                                            mySelector,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
                                bc_mySelector,
                                method_getImplementation(myMethod),
                                method_getTypeEncoding(myMethod));
        } else {
            method_exchangeImplementations(myMethod, swizzledMethod);
        }
    });
}

# pragma mark - Method Swizzling
- (void)bc_myMethod {

}

```
2. Determine the type of a property
```objc
@property (nonatomic, copy) NSString *myProperty;
```
参考[How to detect a property return type in Objective-C](http://stackoverflow.com/questions/769319/how-to-detect-a-property-return-type-in-objective-c)
3. Determine the caller of a method
Stack, framework, address, class, function, line.
参考[Objective C find caller of method](http://stackoverflow.com/questions/1451342/objective-c-find-caller-of-method/1451437#1451437)
```objc
NSString *sourceString = [[NSThread callStackSymbols] objectAtIndex:1];
// Example: 1   UIKit                               0x00540c89 -[UIApplication _callInitializationDelegatesForURL:payload:suspended:] + 1163
NSCharacterSet *separatorSet = [NSCharacterSet characterSetWithCharactersInString:@" -[]+?.,"];
NSMutableArray *array = [NSMutableArray arrayWithArray:[sourceString  componentsSeparatedByCharactersInSet:separatorSet]];
[array removeObject:@""];

NSLog(@"Stack = %@", [array objectAtIndex:0]);
NSLog(@"Framework = %@", [array objectAtIndex:1]);
NSLog(@"Memory address = %@", [array objectAtIndex:2]);
NSLog(@"Class caller = %@", [array objectAtIndex:3]);
NSLog(@"Function caller = %@", [array objectAtIndex:4]);
```
