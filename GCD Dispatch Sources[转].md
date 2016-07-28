# [转]GCD Dispatch Sources
## 何为Dispatch Sources
简单来说，dispatch source是一个监视某些类型事件的对象。当这些事件发生时，它自动将一个block放入一个dispatch queue的执行例程中。
说的貌似有点不清不楚。我们到底讨论哪些事件类型？
下面是GCD 10.6.0版本支持的事件：

```objc
Mach port send right state changes.
Mach port receive right state changes.
External process state change.
File descriptor ready for read.
File descriptor ready for write.
Filesystem node event.
POSIX signal.
Custom timer.
Custom event.
```
这是一堆很有用的东西，它支持所有kqueue所支持的事件（kqueue是什么？见http://en.wikipedia.org/wiki/Kqueue）以及mach（mach是什么？见http://en.wikipedia.org/wiki/Mach_(kernel)）端口、内建计时器支持（这样我们就不用使用超时参数来创建自己的计时器）和用户事件。

## 用户事件
这些事件里面多数都可以从名字中看出含义，但是你可能想知道啥叫用户事件。简单地说，这种事件是由你调用dispatch_source_merge_data函数来向自己发出的信号。
这个名字对于一个发出事件信号的函数来说，太怪异了。这个名字的来由是GCD会在事件句柄被执行之前自动将多个事件进行联结。你可以将数据“拼接”至dispatch source中任意次，并且如果dispatch queue在这期间繁忙的话，GCD只会调用该句柄一次（不要觉得这样会有问题，看完下面的内容你就明白了）。
用户事件有两种： `DISPATCH_SOURCE_TYPE_DATA_ADD` 和 `DISPATCH_SOURCE_TYPE_DATA_OR`.用户事件源有个 `unsigned long data`属性，我们将一个 `unsigned long`传入 `dispatch_source_merge_data`。当使用 `_ADD`版本时，事件在联结时会把这些数字相加。当使用 `_OR`版本时，事件在联结时会把这些数字逻辑与运算。当事件句柄执行时，我们可以使用`dispatch_source_get_data`函数访问当前值，然后这个值会被重置为0。
让我假设一种情况。假设一些异步执行的代码会更新一个进度条。因为主线程只不过是GCD的另一个`dispatch queue`而已，所以我们可以将GUI更新工作push到主线程中。然而，这些事件可能会有一大堆，我们不想对GUI进行频繁而累赘的更新，理想的情况是当主线程繁忙时将所有的改变联结起来。
用`dispatch source`就完美了，使用`DISPATCH_SOURCE_TYPE_DATA_ADD`，我们可以将工作拼接起来，然后主线程可以知道从上一次处理完事件到现在一共发生了多少改变，然后将这一整段改变一次更新至进度条。
啥也不说了，上代码：
```objc
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0, dispatch_get_main_queue());
dispatch_source_set_event_handler(source, ^{
    [progressIndicator incrementBy:dispatch_source_get_data(source)];
});
dispatch_resume(source);

dispatch_apply([array count], globalQueue, ^(size_t index) {
    // do some work on data at index
    dispatch_source_merge_data(source, 1);
});
```
（对于这段代码，我很想说点什么，我第一次用dispatch source时，我纠结了很久很久，真让人蛋疼：**Dispatch source启动时默认状态是挂起的，我们创建完毕之后得主动恢复，否则事件不会被传递，也不会被执行**）

假设你已经将进度条的min/max值设置好了，那么这段代码就完美了。数据会被并发处理。当每一段数据完成后，会通知dispatch source并将dispatch source data加1，这样我们就认为一个单元的工作完成了。事件句柄根据已完成的工作单元来更新进度条。若主线程比较空闲并且这些工作单元进行的比较慢，那么事件句柄会在每个工作单元完成的时候被调用，实时更新。如果主线程忙于其他工作，或者工作单元完成速度很快，那么完成事件会被联结起来，导致进度条只在主线程变得可用时才被更新，并且一次将积累的改变更新至GUI。
现在你可能会想，听起来倒是不错，但是要是我不想让事件被联结呢？有时候你可能想让每一次信号都会引起响应，什么后台的智能玩意儿统统不要。啊。。其实很简单的，别把自己绕进去了。如果你想让每一个信号都得到响应，那使用dispatch_async函数不就行了。实际上，使用的dispatch source而不使用dispatch_async的唯一原因就是利用联结的优势。

## 内建事件
上面就是怎样使用用户事件，那么内建事件呢？看看下面这个例子，用GCD读取标准输入：
```objc
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_source_t stdinSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ,
                                                       STDIN_FILENO,
                                                       0,
                                                       globalQueue);
dispatch_source_set_event_handler(stdinSource, ^{
    char buf[1024];
    int len = read(STDIN_FILENO, buf, sizeof(buf));
    if(len > 0)
        NSLog(@"Got data from stdin: %.*s", len, buf);
});
dispatch_resume(stdinSource);
```

简单的要死！因为我们使用的是全局队列，句柄自动在后台执行，与程序的其他部分并行，这意味着对这种情况的提速：事件进入程序时，程序正在处理其他事务。

这是标准的UNIX方式来处理事务的好处，不用去写loop。如果使用经典的 `read`调用，我们还得万分留神，因为返回的数据可能比请求的少，还得忍受无厘头的“errors”，比如 `EINTR` (系统调用中断)。使用GCD，我们啥都不用管，就从这些蛋疼的情况里解脱了。如果我们在文件描述符中留下了未读取的数据，GCD会再次调用我们的句柄。
对于标准输入，这没什么问题，但是对于其他文件描述符，我们必须考虑在完成读写之后怎样清除描述符。对于dispatch source还处于活跃状态时，我们决不能关闭描述符。如果另一个文件描述符被创建了（可能是另一个线程创建的）并且新的描述符刚好被分配了相同的数字，那么你的dispatch source可能会在不应该的时候突然进入读写状态。de这个bug可不是什么好玩的事儿。
适当的清除方式是使用 `dispatch_source_set_cancel_handler`，并传入一个block来关闭文件描述符。然后我们使用 `dispatch_source_cancel`来取消dispatch source，使得句柄被调用，然后文件描述符被关闭。
使用其他dispatch source类型也差不多。总的来说，你提供一个source（mach port、文件描述符、进程ID等等）的区分符来作为diapatch source的句柄。mask参数通常不会被使用，但是对于 `DISPATCH_SOURCE_TYPE_PROC` 来说mask指的是我们想要接受哪一种进程事件。然后我们提供一个句柄，然后恢复这个source（前面我加粗字体所说的，得先恢复），搞定。dispatch source也提供一个特定于source的data，我们使用 `dispatch_source_get_data`函数来访问它。例如，文件描述符会给出大致可用的字节数。进程source会给出上次调用之后发生的事件的mask。具体每种source给出的data的含义，看man page吧。

## 定时器
计时器事件稍有不同。它们不使用handle/mask参数，计时器事件使用另外一个函数 `dispatch_source_set_timer` 来配置计时器。这个函数使用三个参数来控制计时器触发：

 * `start`参数控制计时器第一次触发的时刻。参数类型是 `dispatch_time_t`，这是一个opaque类型，我们不能直接操作它。我们得需要 `dispatch_time` 和  `dispatch_walltime` 函数来创建它们。另外，常量`DISPATCH_TIME_NOW`和 `DISPATCH_TIME_FOREVER`通常很有用。
 * `interval`参数没什么好解释的。
 * `leeway`参数比较有意思。这个参数告诉系统我们需要计时器触发的精准程度。所有的计时器都不会保证100%精准，这个参数用来告诉系统你希望系统保证精准的努力程度。如果你希望一个计时器没五秒触发一次，并且越准越好，那么你传递0为参数。另外，如果是一个周期性任务，比如检查email，那么你会希望每十分钟检查一次，但是不用那么精准。所以你可以传入60，告诉系统60秒的误差是可接受的。
这样有什么意义呢？简单来说，就是降低资源消耗。如果系统可以让cpu休息足够长的时间，并在每次醒来的时候执行一个任务集合，而不是不断的醒来睡去以执行任务，那么系统会更高效。如果传入一个比较大的leeway给你的计时器，意味着你允许系统拖延你的计时器来将计时器任务与其他任务联合起来一起执行。

>总结：现在你知道怎样使用GCD的dispatch source功能来监视文件描述符、计时器、联结的用户事件以及其他类似的行为。由于dispatch source完全与dispatch queue相集成，所以你可以使用任意的dispatch queue。你可以将一个dispatch source的句柄在主线程中执行、在全局队列中并发执行、或者在用户队列中串行执行（执行时会将程序的其他模块的运算考虑在内）。
下一篇我会讨论如何对dispatch queue进行挂起、恢复、重定目标操作；如何使用dispatch semaphore；如何使用GCD的一次性初始化功能。
