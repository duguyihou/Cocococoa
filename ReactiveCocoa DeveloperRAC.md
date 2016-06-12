1. 神马是rac
	 Github 开源的一个应用于 iOS 和 OS X 开发的框架，
	 兼具 函数式编程 和 响应式编程 的特性

Mattt Thompson 大神 : 开启一个新Objective-C纪元

解决问题：
	传统iOS开发，消息传递、回调机制复杂等问题，使之清晰化，条理化



Github主页：
    https://github.com/ReactiveCocoa/ReactiveCocoa

体积庞大：
	https://github.com/ReactiveCocoa/ReactiveCocoa/tree/master/ReactiveCocoa/Objective-C

用法：
	https://github.com/ReactiveCocoa/ReactiveCocoa/tree/master/Documentation/Legacy

    最经典的示例：

    	登录/注销需求：
    	* 用户名、密码的长度均大于0，且当前尚未登录，登录按钮才可以点击（需要实时性地控制登录按钮是否可用的状态！）






在开始具体内容前 ---> (插播：https://github.com/DeveloperLx/LxDBAnything)






最常用的几招：（死记硬背都能会的）

target-action：

	文本框事件：

	UITextField * textField = [[UITextField alloc]init];
    textField.backgroundColor = [UIColor cyanColor];
    [self.view addSubview:textField];

    @weakify(self); //  __weak __typeof__(self) self_weak_ = self;

    [textField mas_makeConstraints:^(MASConstraintMaker *make) {

        @strongify(self);   //  __strong __typeof__(self) self = self_weak_;
        make.size.mas_equalTo(CGSizeMake(180, 40));
        make.center.equalTo(self.view);
    }];

    [textField addTarget:self action:@selector(textChanged:) forControlEvents:UIControlEventEditingChanged];

	- (void)textChanged:(UITextField *)textField
	{
	    LxDBAnyVar(textField);
	}



    [[textField rac_signalForControlEvents:UIControlEventEditingChanged]
    subscribeNext:^(id x) {

        LxDBAnyVar(x);
    }];

    [textField.rac_textSignal subscribeNext:^(id x) {

        LxDBAnyVar(x);
    }];

	Tip: id x -> NSString * text




	手势：(不待演示老式的写法了)

	self.view.userInteractionEnabled = YES;

    UITapGestureRecognizer * tap = [[UITapGestureRecognizer alloc]init];
    [[tap rac_gestureSignal] subscribeNext:^(UITapGestureRecognizer * tap) {

        LxDBAnyVar(tap);
    }];
    [self.view addGestureRecognizer:tap];




	通知：(最简单)

	[[[NSNotificationCenter defaultCenter] rac_addObserverForName:UIApplicationDidEnterBackgroundNotification object:nil] subscribeNext:^(NSNotification * notification) {

        LxDBAnyVar(notification);
    }];

	（不需要removeObserver：）



	定时器：

	常用两种：

	1. 延迟某个时间后再做某件事
	[[RACScheduler mainThreadScheduler]afterDelay:2 schedule:^{

        LxPrintAnything(rac);
    }];

    2. 每个一定长度时间做一件事
    [[RACSignal interval:1 onScheduler:[RACScheduler mainThreadScheduler]]subscribeNext:^(NSDate * date) {

        LxDBAnyVar(date);
    }];



代理：(有局限，只能取代没有返回值的代理方法)

    UIAlertView * alertView = [[UIAlertView alloc]initWithTitle:@"RAC" message:@"ReactiveCocoa" delegate:self cancelButtonTitle:@"Cancel" otherButtonTitles:@"Ensure", nil];
    [[self rac_signalForSelector:@selector(alertView:clickedButtonAtIndex:) fromProtocol:@protocol(UIAlertViewDelegate)] subscribeNext:^(RACTuple * tuple) {

        LxDBAnyVar(tuple);

        LxDBAnyVar(tuple.first);
        LxDBAnyVar(tuple.second);
        LxDBAnyVar(tuple.third);
    }];
    [alertView show];



    //	更简单的方式：
    [[alertView rac_buttonClickedSignal]subscribeNext:^(id x) {

        LxDBAnyVar(x);
    }];





KVO:

	UIScrollView * scrollView = [[UIScrollView alloc]init];
    scrollView.delegate = (id<UIScrollViewDelegate>)self;
    [self.view addSubview:scrollView];

    UIView * scrollViewContentView = [[UIView alloc]init];
    scrollViewContentView.backgroundColor = [UIColor yellowColor];
    [scrollView addSubview:scrollViewContentView];

    @weakify(self);

    [scrollView mas_makeConstraints:^(MASConstraintMaker *make) {

        @strongify(self);
        make.edges.equalTo(self.view).insets(UIEdgeInsetsMake(80, 80, 80, 80));
    }];

    [scrollViewContentView mas_makeConstraints:^(MASConstraintMaker *make) {

        @strongify(self);
        make.edges.equalTo(scrollView);
        make.size.mas_equalTo(CGSizeMake(CGRectGetWidth(self.view.frame), CGRectGetHeight(self.view.frame)));
    }];

    [RACObserve(scrollView, contentOffset) subscribeNext:^(id x) {

        LxDBAnyVar(x);
    }];

    （好处：写法简单，keypath有代码提示）

进阶：

1.	创建信号 & 激活信号 & 废弃信号 ：



```
- (RACSignal *)loginSignal {
        return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
            RACDisposable * schedulerDisposable = [[RACScheduler mainThreadScheduler]afterDelay:2 schedule:^{
                if (arc4random()%10 > 1) {
                    [subscriber sendNext:@"Login response"];
                    [subscriber sendCompleted];
                }
                else {
                    [subscriber sendError:[NSError errorWithDomain:@"LOGIN_ERROR_DOMAIN" code:444 userInfo:@{}]];
                }
            }];
            return [RACDisposable disposableWithBlock:^{
                [schedulerDisposable dispose];
            }];
        }];
    }
```
    经典示例：

    - (RACSignal *)rac_addObserverForName:(NSString *)notificationName object:(id)object {
        @unsafeify(object);
        return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
            @strongify(object);
            id observer = [self addObserverForName:notificationName object:object queue:nil usingBlock:^(NSNotification *note) {
                [subscriber sendNext:note];
            }];

            return [RACDisposable disposableWithBlock:^{
                [self removeObserver:observer];
            }];
        }] setNameWithFormat:@"-rac_addObserverForName: %@ object: <%@: %p>", notificationName, [object class], object];
    }






2.	信号的处理：

	map：
	filter:

	delay:

    startWith:

    timeout:

    [[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

        [[RACScheduler mainThreadScheduler]afterDelay:3 schedule:^{

            [subscriber sendNext:@"rac"];
            [subscriber sendCompleted];
        }];

        return nil;
    }] timeout:2 onScheduler:[RACScheduler mainThreadScheduler]]
    subscribeNext:^(id x) {

        LxDBAnyVar(x);
    } error:^(NSError *error) {

        LxDBAnyVar(error);
    } completed:^{

        LxPrintAnything(completed);
    }];

    take:
    skip:

    takeLast:
    takeUntil:
    takeWhileBlock:

    skipWhileBlock:
    skipUntilBlock:






    throttle:   (即时搜索优化)
    distinctUntilChanged:
    ignore:

    switchToLatest:

    UITextField * textField = [[UITextField alloc]init];
    textField.backgroundColor = [UIColor cyanColor];
    [self.view addSubview:textField];

    @weakify(self);

    [textField mas_makeConstraints:^(MASConstraintMaker *make) {

        @strongify(self);
        make.size.mas_equalTo(CGSizeMake(180, 40));
        make.center.equalTo(self.view);
    }];

    [[[[[[textField.rac_textSignal throttle:0.3]distinctUntilChanged]ignore:@""] map:^id(id value) {

        return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

            //  network request
            [subscriber sendNext:value];
            [subscriber sendCompleted];

            return [RACDisposable disposableWithBlock:^{

                //  cancel request
            }];
        }];
    }]switchToLatest] subscribeNext:^(id x) {

        LxDBAnyVar(x);
    }];

	repeat:
		[[[[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

	        [subscriber sendNext:@"rac"];
	        [subscriber sendCompleted];

	        return nil;
	    }]delay:1]repeat]take:3] subscribeNext:^(id x) {

	        LxDBAnyVar(x);
	    } completed:^{

	        LxPrintAnything(completed);
	    }];







	merge:

	RACSignal * signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            LxPrintAnything(a);
            [subscriber sendNext:@"a"];
            [subscriber sendCompleted];
        });

        return nil;
    }];

    RACSignal * signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            LxPrintAnything(b);
            [subscriber sendNext:@"b"];
            [subscriber sendCompleted];
        });

        return nil;
    }];

    [[RACSignal merge:@[signalA, signalB]]subscribeNext:^(id x) {

        LxDBAnyVar(x);
    }];

    combineLatest:reduce:

	concat:    （一个异步请求完成后，再启动另一个）

	RACSignal * signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            LxPrintAnything(a);
            [subscriber sendNext:@"a"];
            [subscriber sendCompleted];
        });

        return nil;
    }];

    RACSignal * signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            LxPrintAnything(b);
            [subscriber sendNext:@"b"];
            [subscriber sendCompleted];
        });

        return nil;
    }];

    [[signalA concat:signalB]subscribeNext:^(id x) {

        LxDBAnyVar(x);
    }];




	zipWith:/combineLatest:   （多个异步请求都完成后，再做某件事）

	RACSignal * signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            LxPrintAnything(a);
            [subscriber sendNext:@"a"];
            [subscriber sendCompleted];
        });

        return nil;
    }];

    RACSignal * signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            LxPrintAnything(b);
            [subscriber sendNext:@"b"];
            [subscriber sendCompleted];
        });

        return nil;
    }];

    [[signalA zipWith:signalB]subscribeNext:^(id x) {

        LxDBAnyVar(x);
    }];




RAC(<#TARGET, ...#>) 宏:

button setBackgroundColor:forState:

	UIButton * button = [UIButton buttonWithType:UIButtonTypeCustom];
    [self.view addSubview:button];

    @weakify(self);

    [button mas_makeConstraints:^(MASConstraintMaker *make) {

        @strongify(self);
        make.size.mas_equalTo(CGSizeMake(180, 40));
        make.center.equalTo(self.view);
    }];

    RAC(button, backgroundColor) = [RACObserve(button, selected) map:^UIColor *(NSNumber * selected) {

        return [selected boolValue] ? [UIColor redColor] : [UIColor greenColor];
    }];

    [[button rac_signalForControlEvents:UIControlEventTouchUpInside]subscribeNext:^(UIButton * btn) {

        btn.selected = !btn.selected;
    }];


用最少的代码写一个秒表：

	RAC(label, text) = [[RACSignal interval:1 onScheduler:[RACScheduler mainThreadScheduler]] map:^NSString *(NSDate * date) {

        return date.description;
    }];
