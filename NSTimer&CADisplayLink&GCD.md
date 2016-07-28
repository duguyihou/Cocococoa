# NSTimer
## 创建方法
```objc
NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(action:) userInfo:nil repeats:NO];
```
TimerInterval: 执行之前等待的时间。比如设置成1.0，就代表1秒后执行方法    
target: 需要执行方法的对象。    
selector: 需要执行的方法    
repeats: 是否需要循环
## 停止方法
```objc
[timer invalidate];     
```
## 特点
- 存在延迟：不管是一次性的还是周期性的timer的实际触发事件的时间，都会与所加入的RunLoop和RunLoop Mode有关，如果此RunLoop正在执行一个连续性的运算，timer就会被延时出发。重复性的timer遇到这种情况，如果延迟超过了一个周期，则会在延时结束后立刻执行，并按照之前指定的周期继续执行。    
- 必须加入Runloop：使用上面的创建方式，会自动把timer加入MainRunloop的NSDefaultRunLoopMode中。如果使用以下方式创建定时器，就必须手动加入Runloop:
```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:5 target:self selector:@selector(timerAction) userInfo:nil repeats:YES];   
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
```       

## CADisplayLink
>CADisplayLink是一个能让我们以和屏幕刷新率相同的频率将内容画到屏幕上的定时器。我们在应用中创建一个新的 CADisplayLink 对象，把它添加到一个runloop中，并给它提供一个 target 和selector在屏幕刷新的时候调用。

一但 CADisplayLink 以特定的模式注册到runloop之后，每当屏幕需要刷新的时候，runloop就会调用CADisplayLink绑定的target上的selector，这时target可以读到 CADisplayLink 的每次调用的时间戳，用来准备下一帧显示需要的数据。例如一个视频应用使用时间戳来计算下一帧要显示的视频数据。在UI做动画的过程中，需要通过时间戳来计算UI对象在动画的下一帧要更新的大小等等。在添加进runloop的时候我们应该选用高一些的优先级，来保证动画的平滑。可以设想一下，我们在动画的过程中，runloop被添加进来了一个高优先级的任务，那么，下一次的调用就会被暂停转而先去执行高优先级的任务，然后在接着执行CADisplayLink的调用，从而造成动画过程的卡顿，使动画不流畅。

duration属性提供了每帧之间的时间，也就是屏幕每次刷新之间的的时间。我们可以使用这个时间来计算出下一帧要显示的UI的数值。但是 duration只是个大概的时间，如果CPU忙于其它计算，就没法保证以相同的频率执行屏幕的绘制操作，这样会跳过几次调用回调方法的机会。frameInterval属性是可读可写的NSInteger型值，标识间隔多少帧调用一次selector方法，默认值是1，即每帧都调用一次。如果每帧都调用一次的话，对于iOS设备来说那刷新频率就是60HZ也就是每秒60次，如果将 frameInterval 设为2 那么就会两帧调用一次，也就是变成了每秒刷新30次。我们通过pause属性开控制CADisplayLink的运行。当我们想结束一个CADisplayLink的时候，应该调用-(void)invalidate从runloop中删除并删除之前绑定的target跟selector。另外CADisplayLink 不能被继承。
### CADisplayLink 与 NSTimer 有什么不同
iOS设备的屏幕刷新频率是固定的，CADisplayLink在正常情况下会在每次刷新结束都被调用，精确度相当高。
NSTimer的精确度就显得低了点，比如NSTimer的触发时间到的时候，runloop如果在阻塞状态，触发时间就会推迟到下一个runloop周期。并且 NSTimer新增了tolerance属性，让用户可以设置可以容忍的触发的时间的延迟范围。
CADisplayLink使用场合相对专一，适合做UI的不停重绘，比如自定义动画引擎或者视频播放的渲染。NSTimer的使用范围要广泛的多，各种需要单次或者循环定时处理的任务都可以使用。在UI相关的动画或者显示内容使用
CADisplayLink比起用NSTimer的好处就是我们不需要在格外关心屏幕的刷新频率了，因为它本身就是跟屏幕刷新同步的。
### 给非UI对象添加动画效果
动画效果就是一个属性的线性变化，比如UIView 动画的`EasyInEasyOut`。通过数值按照不同速率的变化我们能生成更接近真实世界的动画效果。我们也可以利用这个特性来使一些其他属性按照我们期望的曲线变化。比如**当播放视频时关掉视频的声音我可以通过CADisplayLink来实现一个EasyOut的渐出效果:先快速的降低音量,在慢慢的渐变到静音.**
>注意通常来讲：iOS设备的刷新频率事60HZ也就是每秒60次。那么每一次刷新的时间就是1/60秒 大概16.7毫秒。当我们的frameInterval值为1的时候我们需要保证的是 CADisplayLink调用的｀target｀的函数计算时间不应该大于 16.7否则就会出现严重的丢帧现象。在mac应用中我们使用的不是CADisplayLink而是CVDisplayLink.它是基于C接口的用起来配置有些麻烦但是用起来还是很简单的。
### 创建方法
```objc
displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(handleDisplayLink:)];    
[displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
```
### 停止方法
```objc
[displayLink invalidate];   
displayLink = nil;  
```
当把CADisplayLink对象add到runloop中后，selector就能被周期性调用，类似于重复的NSTimer被启动了；执行invalidate操作时，CADisplayLink对象就会从runloop中移除，selector调用也随即停止，类似于NSTimer的invalidate方法。
### 特点
- 屏幕刷新时调用：CADisplayLink是一个能让我们以和屏幕刷新率同步的频率将特定的内容画到屏幕上的定时器类。CADisplayLink以特定模式注册到runloop后，每当屏幕显示内容刷新结束的时候，runloop就会向CADisplayLink指定的target发送一次指定的selector消息， CADisplayLink类对应的selector就会被调用一次。所以通常情况下，按照iOS设备屏幕的刷新率60次/秒   
- 延迟
    1. iOS设备的屏幕刷新频率是固定的，CADisplayLink在正常情况下会在每次刷新结束都被调用，精确度相当高。但如果调用的方法比较耗时，超过了屏幕刷新周期，就会导致跳过若干次回调调用机会。
    2. 如果CPU过于繁忙，无法保证屏幕60次/秒的刷新率，就会导致跳过若干次调用回调方法的机会，跳过次数取决CPU的忙碌程度。
- 使用场景：从原理上可以看出，CADisplayLink适合做界面的不停重绘，比如视频播放的时候需要不停地获取下一帧用于界面渲染。
### 重要属性    
- `frameInterval`
NSInteger类型的值，用来设置间隔多少帧调用一次selector方法，默认值是1，即每帧都调用一次。
- `duration`
readOnly的CFTimeInterval值，表示两次屏幕刷新之间的时间间隔。
需要注意的是，该属性在target的selector被首次调用以后才会被赋值。
selector的调用间隔时间计算方式是：调用间隔时间 = duration × frameInterval。

## GCD
### 执行一次
```objc
double delayInSeconds = 2.0;    
dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, delayInSeconds * NSEC_PER_SEC);   
dispatch_after(popTime, dispatch_get_main_queue(), ^(void){    //执行事件    });
```
### 重复执行
```objc
NSTimeInterval period = 1.0; //设置时间间隔    
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);    dispatch_source_t _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);    dispatch_source_set_timer(_timer, dispatch_walltime(NULL, 0), period * NSEC_PER_SEC, 0); //每秒执行   dispatch_source_set_event_handler(_timer, ^{    //在这里执行事件    });
dispatch_resume(_timer);
```

```objc
#import "ViewController.h"

NSInteger count = 0;

@interface ViewController ()

//不需要*号。
@property (nonatomic, strong)dispatch_source_t time;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
}

-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    //获得队列
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    //创建一个定时器
    self.time = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    //设置开始时间
    dispatch_time_t start = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3.0 * NSEC_PER_SEC));
    //设置时间间隔
    uint64_t interval = (uint64_t)(2.0* NSEC_PER_SEC);
    //设置定时器
    dispatch_source_set_timer(self.time, start, interval, 0);
    //设置回调
    dispatch_source_set_event_handler(self.time, ^{

        NSLog(@"Run------ ");
        //设置当执行五次是取消定时器
        count++;
        if(count == 5){

            dispatch_cancel(self.time);

        }
    });
    //由于定时器默认是暂停的所以我们手动启动
    //启动定时器
    dispatch_resume(self.time);

}
@end
```

## NSTimer创建后，会在哪个线程运行。

用scheduledTimerWithTimeInterval创建的，在哪个线程创建就会被加入哪个线程的RunLoop中就运行在哪个线程。

自己创建的Timer，加入到哪个线程的RunLoop中就运行在哪个线程。

## 如何让计时器(NSTimer)调用一个类方法

计时器只能调用实例方法，但是可以在这个实例方法里面调用静态方法。

使用计时器需要注意，计时器一定要加入RunLoop中，并且选好model才能运行。scheduledTimerWithTimeInterval 方法创建一个计时器并加入到RunLoop中所以可以直接使用。

如果计时器的repeats选择YES说明这个计时器会重复执行，一定要在合适的时机调用计时器的invalid。 不能在dealloc中调用，因为一旦设置为repeats 为yes，计时器会强持有self，导致dealloc永远不会被调用， 这个类就永远无法被释放。比如可以在viewDidDisappear中调用，这样当类需要被回收的时候就可以正常进入dealloc中了。
