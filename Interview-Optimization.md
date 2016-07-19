## UITableView的调优
通常来说，在开发中注意以下问题，可以使列表滚动比较流畅，但是对于特别复杂的列表就需要做额外的优化处理：
- 重用cell，设置好cellIdentifier
- 重用header、footer view，设置好identifier
- 若高度固定，直接使用rowHight；若不固定则使用heightForRowAtIndexPath代理方法
- 缓存cell的高度、header/footer view的高度
- 不要修改view的opaque，默认就是YES,表示不透明度
- 不要动态添加子view到cell上，直接在初始时创建，然后做显示与隐藏操作
- 尽量不要直接使用cornerRadius，采用镂空图或者Core Graphics API来绘制
- 将I/O操作、复杂运算放到子线程中处理，再回到主线程更新UI

如果列表比较复杂，对于上面的做好后，还是不够流畅，就需要通过线程s工具来检测哪些地方可以优化了。
中文:[UITableView 的完美平滑滚动](http://ios.jobbole.com/84360/)
英文:[perfect-smooth-scrolling-in-uitableviews](https://medium.com/ios-os-x-development/perfect-smooth-scrolling-in-uitableviews-fd609d5275a5#.6fnfw5gkm)

## [8 Patterns to Help You Destroy Massive View Controller](http://khanlou.com/2014/09/8-patterns-to-help-you-destroy-massive-view-controller/?utm_source=wanqu.co&utm_campaign=Wanqu+Daily&utm_medium=website)

## 用Instrument优化动画性能的经历
[iOS App性能优化](http://www.hrchen.com/2013/05/performance-with-instruments/)

1. Separate by Thread: 每个线程应该分开考虑。只有这样你才能揪出那些大量占用CPU的"重"线程。
2. Invert Call Tree: 从上倒下跟踪堆栈,这意味着你看到的表中的方法,将已从第0帧开始取样,
这通常你是想要的,只有这样你才能看到CPU中话费时间最深的方法.也就是说FuncA{FunB{FunC}}
勾选此项后堆栈以C->B-A 把调用层级最深的C显示在最外面。
3. Hide System Libraries: 勾选此项你会显示你app的代码,这是非常有用的.
因为通常你只关心cpu花在自己代码上的时间不是系统上的。
4. Flatten Recursion: 递归函数, 每个堆栈跟踪一个条目。
5. Top Functions: 一个函数花费的时间直接在该函数中的总和，以及在函数调用该函数所花费的时间的总时间。
因此，如果函数A调用B，那么A的时间报告在A花费的时间加上B.花费的时间,这非常真有用，
因为它可以让你每次下到调用堆栈时挑最大的时间数字，归零在你最耗时的方法。

内存泄漏有两种泄漏。第一个是真正的内存泄漏，一个对象尚未被释放，但是不再被引用的了。
因此，存储器不能被重新使用。第二类泄漏是比较麻烦一些。这就是所谓的“无界内存增长”。
这发生在内存继续分配，并永远不会有机会被释放。如果永远这样下去你的程序占用的内存会无限大,
当超过一定内存的话 会被系统的看门狗给kill掉。

内存警告是ios处理app最好的方式，尤其是在内存越来越吃紧的时候,你需要清除一些内存。
内存一直增长其实也不一定是你的代码出了问题,也有可能是UIKit 系统框架本身导致的。

**内存泄露**
这一类泄漏是前面提到的 - 当一个对象不再被引用时出现的那种,检测泄漏可以理解为一个很复杂的事情，
但泄漏的工具记得已分配的所有对象，通过定期扫描每个对象以确定是否有任何不能从任何其他对象访问的。

## **大量数据表的优化方案**
1. 对查询进行优化，要尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。

2. 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描，如：

 `select id from t where num is null`
最好不要给数据库留NULL，尽可能的使用 NOT NULL填充数据库.

备注、描述、评论之类的可以设置为 NULL，其他的，最好不要使用NULL。

不要以为 NULL 不需要空间，比如：char(100) 型，在字段建立时，空间就固定了，
不管是否插入值（NULL也包含在内），都是占用 100个字符的空间的，如果是varchar这样的变长字段， null 不占用空间。

可以在num上设置默认值0，确保表中num列没有null值，然后这样查询：

select id from t where num=0
3. 应尽量避免在 where 子句中使用 != 或 <> 操作符，否则将引擎放弃使用索引而进行全表扫描。

4. 应尽量避免在 where 子句中使用 or 来连接条件，如果一个字段有索引，一个字段没有索引，
将导致引擎放弃使用索引而进行全表扫描，如：

 select id from t where num=10 or Name='admin'
可以这样查询：

 select id from t where num=10 union all select id from t where Name='admin'
5. in 和 not in 也要慎用，否则会导致全表扫描，如：

select id from t where num in (1,2,3)
对于连续的数值，能用 between 就不要用 in 了：

 select id from t where num between 1 and 3
很多时候用 exists 代替 in 是一个好的选择：

 select num from a where num in (select num from b)
用下面的语句替换：

 select num from a where exists (select 1 from b where num=a.num)
6. 下面的查询也将导致全表扫描：

select id from t where name like ‘%abc%’
若要提高效率，可以考虑全文检索。

7. 如果在 where 子句中使用参数，也会导致全表扫描。因为SQL只有在运行时才会解析局部变量，
但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时进行选择。然而，
如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。
如下面语句将进行全表扫描：

select id from t where num=@num
可以改为强制查询使用索引：

select id from t with (index(索引名)) where num=@num
应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。如：

select id from t where num/2=100
应改为:

select id from t where num=100*2
8. 应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：

 select id from t where substring(name,1,3)=’abc’ -–name以abc开头的id
 select id from t where datediff(day,createdate,’2015-11-30′)=0 -–‘2015-11-30’ --生成的id
应改为:

select id from t where name like'abc%'
select id from t where createdate>='2005-11-30' and createdate<'2005-12-1'
9. 不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。

10. 在使用索引字段作为条件时，如果该索引是复合索引，
那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使用，
并且应尽可能的让字段顺序与索引顺序相一致。

11. 不要写一些没有意义的查询，如需要生成一个空表结构：

select col1,col2 into #t from t where1=0
这类代码不会返回任何结果集，但是会消耗系统资源的，应改成这样：

create table #t(…)
12. Update 语句，如果只更改1、2个字段，不要Update全部字段，否则频繁调用会引起明显的性能消耗，同时带来大量日志。

13. 对于多张大数据量（这里几百条就算大了）的表JOIN，要先分页再JOIN，否则逻辑读会很高，性能很差。

14. select count(*) from table；这样不带任何条件的count会引起全表扫描，并且没有任何业务意义，是一定要杜绝的。

15. 索引并不是越多越好，索引固然可以提高相应的 select 的效率，
但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，
所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过6个，
若太多则应考虑一些不常使用到的列上建的索引是否有 必要。

16. 应尽可能的避免更新 clustered 索引数据列，因为 clustered 索引数据列的顺序就是表记录的物理存储顺序，
一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新 clustered 索引数据列，
那么需要考虑是否应将该索引建为 clustered 索引。

17. 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，
并会增加存储开销。这是因为引擎在处理查询和连 接时会逐个比较字符串中每一个字符，
而对于数字型而言只需要比较一次就够了。

18. 尽可能的使用 varchar/nvarchar 代替 char/nchar ，因为首先变长字段存储空间小，
可以节省存储空间，其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。

19. 任何地方都不要使用

  select * from t
用具体的字段列表代替“*”，不要返回用不到的任何字段。

20. 尽量使用表变量来代替临时表。如果表变量包含大量数据，请注意索引非常有限（只有主键索引）。

21. 避免频繁创建和删除临时表，以减少系统表资源的消耗。临时表并不是不可使用，
适当地使用它们可以使某些例程更有效，例如，当需要重复引用大型表或常用表中的某个数据集时。
但是，对于一次性事件， 最好使用导出表。

22. 在新建临时表时，如果一次性插入数据量很大，那么可以使用 select into 代替 create table，
避免造成大量 log ，以提高速度；如果数据量不大，为了缓和系统表的资源，应先create table，然后insert。

23. 如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先 truncate table ，
然后 drop table ，这样可以避免系统表的较长时间锁定。

24. 尽量避免使用游标，因为游标的效率较差，如果游标操作的数据超过1万行，那么就应该考虑改写。

25. 使用基于游标的方法或临时表方法之前，应先寻找基于集的解决方案来解决问题，基于集的方法通常更有效。

26. 与临时表一样，游标并不是不可使用。对小型数据集使用 FAST_FORWARD 游标通常要优于其他逐行处理方法，
尤其是在必须引用几个表才能获得所需的数据时。在结果集中包括“合计”的例程通常要比使用游标执行的速度快。
如果开发时 间允许，基于游标的方法和基于集的方法都可以尝试一下，看哪一种方法的效果更好。

27. 在所有的存储过程和触发器的开始处设置 SET NOCOUNT ON ，在结束时设置 SET NOCOUNT OFF 。
无需在执行存储过程和触发器的每个语句后向客户端发送 DONE_IN_PROC 消息。

28. 尽量避免大事务操作，提高系统并发能力。

29. 尽量避免向客户端返回大数据量，若数据量过大，应该考虑相应需求是否合理。

实际案例分析：拆分大的 DELETE 或INSERT 语句，批量提交SQL语句

如果你需要在一个在线的网站上去执行一个大的 DELETE 或 INSERT 查询，你需要非常小心，
要避免你的操作让你的整个网站停止相应。因为这两个操作是会锁表的，表一锁住了，别的操作都进不来了。

Apache 会有很多的子进程或线程。所以，其工作起来相当有效率，而我们的服务器也不希望有太多的子进程，
线程和数据库链接，这是极大的占服务器资源的事情，尤其是内存。

如果你把你的表锁上一段时间，比如30秒钟，那么对于一个有很高访问量的站点来说，这30秒所积累的访问进程/线程，
数据库链接，打开的文件数，可能不仅仅会让你的WEB服务崩溃，还可能会让你的整台服务器马上挂了。

所以，如果你有一个大的处理，你一定把其拆分，使用 LIMIT oracle(rownum),
sqlserver(top)条件是一个好的方法。下面是一个mysql示例：

## 界面卡顿产生的原因和解决方案
> iOS界面处理是在主线程下进行的，系统图形服务通过 CADisplayLink 等机制通知 App，App 主
线程开始在 CPU 中计算显示内容，比如视图的创建、布局计算、图片解码、文本绘制等。随后 CPU 会
将计算好的内容提交到 GPU 去，由 GPU 进行变换、合成、渲染。随后 GPU 会把渲染结果提交到帧缓冲区去，
等待下一次刷新信号到来时显示到屏幕上。显示器通常以固定频率进行刷新，如果在一个刷新时间内，CPU 或者
GPU 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。
这就是界面卡顿的原因。CPU 和 GPU 不论哪个阻碍了显示流程，都会造成掉帧现象。
### CPU 资源消耗原因和解决方案
#### 对象创建
对象的创建会分配内存、调整属性、甚至还有读取文件等操作，比较消耗 CPU 资源。尽量用轻量的对象代替重量的对象，
可以对性能有所优化。比如 CALayer 比 UIView 要轻量许多，那么不需要响应触摸事件的控件，
用 CALayer 显示会更加合适。如果对象不涉及 UI 操作，则尽量放到后台线程去创建，但可惜的是包含有
CALayer 的控件，都只能在主线程创建和操作。通过 Storyboard 创建视图对象时，
其资源消耗会比直接通过代码创建对象要大非常多，在性能敏感的界面里，Storyboard 并不是一个好的技术选择。

尽量推迟对象创建的时间，并把对象的创建分散到多个任务中去。尽管这实现起来比较麻烦，并且带来的优势并不多，
但如果有能力做，还是要尽量尝试一下。如果对象可以复用，并且复用的代价比释放、创建新对象要小，
那么这类对象应当尽量放到一个缓存池里复用。

#### 对象调整
对象的调整也经常是消耗 CPU 资源的地方。这里特别说一下 CALayer：CALayer 内部并没有属性，
当调用属性方法时，它内部是通过运行时 resolveInstanceMethod 为对象临时添加一个方法，
并把对应属性值保存到内部的一个 Dictionary 里，同时还会通知 delegate、创建动画等等，非常消耗资源。
UIView 的关于显示相关的属性（比如 frame/bounds/transform）等实际上都是 CALayer 属性映射来的，
所以对 UIView 的这些属性进行调整时，消耗的资源要远大于一般的属性。对此你在应用中，
应该尽量减少不必要的属性修改。当视图层次调整时，UIView、CALayer 之间会出现很多方法调用与通知，
所以在优化性能时，应该尽量避免调整视图层次、添加和移除视图。

#### 对象销毁
对象的销毁虽然消耗资源不多，但累积起来也是不容忽视的。通常当容器类持有大量对象时，
其销毁时的资源消耗就非常明显。同样的，如果对象可以放到后台线程去释放，那就挪到后台线程去。
这里有个小 Tip：把对象捕获到 block 中，然后扔到后台队列去随便发送个消息以避免编译器警告，
就可以让对象在后台线程销毁了。
```objc
NSArray *tmp = self.array;
self.array = nil;
dispatch_async(queue, ^{
[tmp class];
});
```
#### 布局计算
视图布局的计算是 App 中最为常见的消耗 CPU 资源的地方。如果能在后台线程提前计算好视图布局、
并且对视图布局进行缓存，那么这个地方基本就不会产生性能问题了。

不论通过何种技术对视图进行布局，其最终都会落到对 UIView.frame/bounds/center 等属性的调整上。
上面也说过，对这些属性的调整非常消耗资源，所以尽量提前计算好布局，在需要时一次性调整好对应属性，
而不要多次、频繁的计算和调整这些属性。

#### Autolayout
Autolayout 是苹果本身提倡的技术，在大部分情况下也能很好的提升开发效率，但是 Autolayout
对于复杂视图来说常常会产生严重的性能问题。随着视图数量的增长，Autolayout 带来的 CPU 消耗会呈指数级上升。
如果你不想手动调整 frame 等属性，你可以用一些工具方法替代（比如常见的 left/right/top/bottom/width/height 快捷属性，
或者使用 ComponentKit、AsyncDisplayKit 等框架。

#### 文本计算
如果一个界面中包含大量文本（比如微博微信朋友圈等），文本的宽高计算会占用很大一部分资源，并且不可避免。
如果你对文本显示没有特殊要求，
可以参考下 UILabel 内部的实现方式：用 [NSAttributedString boundingRectWithSize:options:context:]
来计算文本宽高，用 -[NSAttributedString drawWithRect:options:context:] 来绘制文本。
尽管这两个方法性能不错，但仍旧需要放到后台线程进行以避免阻塞主线程。如果你用 CoreText 绘制文本，
那就可以先生成 CoreText 排版对象，然后自己计算了，并且 CoreText 对象还能保留以供稍后绘制使用。

#### 文本渲染
屏幕上能看到的所有文本内容控件，包括 UIWebView，在底层都是通过 CoreText 排版、绘制为 Bitmap 显示的。
常见的文本控件 （UILabel、UITextView 等），其排版和绘制都是在主线程进行的，当显示大量文本时，
CPU 的压力会非常大。对此解决方案只有一个，那就是自定义文本控件，用 TextKit 或最底层的 CoreText 对文本异步绘制。
尽管这实现起来非常麻烦，但其带来的优势也非常大，CoreText 对象创建好后，能直接获取文本的宽高等信息，
避免了多次计算（调整 UILabel 大小时算一遍、UILabel 绘制时内部再算一遍）；CoreText 对象占用内存较少，
可以缓存下来以备稍后多次渲染。

#### 图片的解码
当你用 UIImage 或 CGImageSource 的那几个方法创建图片时，图片数据并不会立刻解码。
图片设置到UIImageView 或者 CALayer.contents 中去，并且 CALayer 被提交到 GPU 前，
CGImage 中的数据才会得到解码。这一步是发生在主线程的，并且不可避免。如果想要绕开这个机制，
常见的做法是在后台线程先把图片绘制到 CGBitmapContext 中，然后从 Bitmap 直接创建图片。
目前常见的网络图片库都自带这个功能。

#### 图像的绘制
图像的绘制通常是指用那些以 CG 开头的方法把图像绘制到画布中，然后从画布创建图片并显示这样一个过程。
这个最常见的地方就是 [UIView drawRect:] 里面了。由于 CoreGraphic 方法通常都是线程安全的，
所以图像的绘制可以很容易的放到后台线程进行。一个简单异步绘制的过程大致如下（实际情况会比这个复杂得多，但原理基本一致）：
```objc
- (void)display {
dispatch_async(backgroundQueue, ^{
    CGContextRef ctx = CGBitmapContextCreate(...);
    // draw in context...
    CGImageRef img = CGBitmapContextCreateImage(ctx);
    CFRelease(ctx);
    dispatch_async(mainQueue, ^{
        layer.contents = img;
    });
});
}
```
### GPU资源消耗原因和解决方案
相对于 CPU 来说，GPU 能干的事情比较单一：接收提交的纹理（Texture）和顶点描述（三角形），
应用变换（transform）、混合并渲染，然后输出到屏幕上。通常你所能看到的内容，主要也就是纹理（图片）
和形状（三角模拟的矢量图形）两类。

#### 纹理的渲染
所有的 Bitmap，包括图片、文本、栅格化的内容，最终都要由内存提交到显存，绑定为 GPU Texture。
不论是提交到显存的过程，还是 GPU 调整和渲染 Texture 的过程，都要消耗不少 GPU 资源。
当在较短时间显示大量图片时（比如 TableView 存在非常多的图片并且快速滑动时），CPU 占用率很低，
GPU 占用非常高，界面仍然会掉帧。避免这种情况的方法只能是尽量减少在短时间内大量图片的显示，
尽可能将多张图片合成为一张进行显示。

当图片过大，超过 GPU 的最大纹理尺寸时，图片需要先由 CPU 进行预处理，这对 CPU 和 GPU
都会带来额外的资源消耗。目前来说，iPhone 4S 以上机型，纹理尺寸上限都是 4096x4096，所以，
尽量不要让图片和视图的大小超过这个值。

#### 视图的混合 (Composing)
当多个视图（或者说 CALayer）重叠在一起显示时，GPU 会首先把他们混合到一起。如果视图结构过于复杂，
混合的过程也会消耗很多 GPU 资源。为了减轻这种情况的 GPU 消耗，应用应当尽量减少视图数量和层次，
并在不透明的视图里标明 opaque 属性以避免无用的 Alpha 通道合成。当然，这也可以用上面的方法，
把多个视图预先渲染为一张图片来显示。

#### 图形的生成
CALayer 的 border、圆角、阴影、遮罩（mask），CASharpLayer 的矢量图形显示，通常会触发离屏渲染
（offscreen rendering），而离屏渲染通常发生在 GPU 中。当一个列表视图中出现大量圆角的 CALayer，
并且快速滑动时，可以观察到 GPU 资源已经占满，而 CPU 资源消耗很少。这时界面仍然能正常滑动，
但平均帧数会降到很低。为了避免这种情况，可以尝试开启 CALayer.shouldRasterize 属性，
但这会把原本离屏渲染的操作转嫁到 CPU 上去。对于只需要圆角的某些场合，
也可以用一张已经绘制好的圆角图片覆盖到原本视图上面来模拟相同的视觉效果。最彻底的解决办法，
就是把需要显示的图形在后台线程绘制为图片，避免使用圆角、阴影、遮罩等属性。

## 如何追踪app崩溃率，如何解决线上闪退
当iOS设备上的App应用闪退时，操作系统会生成一个crash日志，保存在设备上。crash日志上有很多有用的信息，
比如每个正在执行线程的完整堆栈跟踪信息和内存映像，这样就能够通过解析这些信息进而定位crash发生时的代码逻辑，
从而找到App闪退的原因。通常来说，crash产生来源于两种问题：违反iOS系统规则导致的crash和App代码逻辑BUG导致的crash，
下面分别对他们进行分析。

### 违反iOS系统规则产生crash的三种类型
1. 内存报警闪退
当iOS检测到内存过低时，它的VM系统会发出低内存警告通知，尝试回收一些内存；如果情况没有得到足够的改善，
iOS会终止后台应用以回收更多内存；最后，如果内存还是不足，那么正在运行的应用可能会被终止掉。
在Debug模式下，可以主动将客户端执行的动作逻辑写入一个log文件中，这样程序童鞋可以将内存预警的逻辑写入该log文件，
当发生如下截图中的内存报警时，就是提醒当前客户端性能内存吃紧，可以通过Instruments工具中的Allocations
和 Leaks模块库来发现内存分配问题和内存泄漏问题。
2. 响应超时
当应用程序对一些特定的事件（比如启动、挂起、恢复、结束）响应不及时，苹果的Watchdog机制会把应用程序干掉，
并生成一份相应的crash日志。这些事件与下列UIApplicationDelegate方法相对应，当遇到Watchdog日志时，
可以检查上图中的几个方法是否有比较重的阻塞UI的动作。　
```objc
application:didFinishLaunchingWithOptions:　
applicationWillResignActive:
applicationDidEnterBackground:　
applicationWillEnterForeground:
applicationDidBecomeActive:
applicationWillTerminate:
```
3. 用户强制退出
一看到“用户强制退出”，首先可能想到的双击Home键，然后关闭应用程序。不过这种场景一般是不会产生crash日志的，
因为双击Home键后，所有的应用程序都处于后台状态，而iOS随时都有可能关闭后台进程，
当应用阻塞界面并停止响应时这种场景才会产生crash日志。这里指的“用户强制退出”场景，
是稍微比较复杂点的操作：先按住电源键，直到出现“滑动关机”的界面时，再按住Home键，
这时候当前应用程序会被终止掉，并且产生一份相应事件的crash日志。

### 应用逻辑的Bug
大多数闪退崩溃日志的产生都是因为应用中的Bug，这种Bug的错误种类有很多，比如　　
```objc
SEGV：（Segmentation Violation，段违例），无效内存地址，比如空指针，未初始化指针，栈溢出等；
  SIGABRT：收到Abort信号，可能自身调用abort()或者收到外部发送过来的信号；
  SIGBUS：总线错误。与SIGSEGV不同的是，SIGSEGV访问的是无效地址（比如虚存映射不到物理内存），
  而SIGBUS访问的是有效地址，但总线访问异常（比如地址对齐问题）；
  SIGILL：尝试执行非法的指令，可能不被识别或者没有权限；
  SIGFPE：Floating Point Error，数学计算相关问题（可能不限于浮点计算），比如除零操作；
  SIGPIPE：管道另一端没有进程接手数据；
```
常见的崩溃原因基本都是代码逻辑问题或资源问题，比如数组越界，访问野指针或者资源不存在，或资源大小写错误等。

### crash的收集

如果是在windows上你可以通过itools或pp助手等辅助工具查看系统产生的历史crash日志，
然后再根据app来查看。如果是在Mac 系统上，只需要打开xcode->windows->devices，选择device logs进行查看，
如下图，这些crash文件都可以导出来，然后再单独对这个crash文件做处理分析。
市场上已有的商业软件提供crash收集服务，这些软件基本都提供了日志存储，日志符号化解析和服务端可视化管理等服务：
Crashlytics (www.crashlytics.com)
Crittercism (www.crittercism.com)
Bugsense (www.bugsense.com)　　
HockeyApp (www.hockeyapp.net)　　
Flurry(www.flurry.com)
开源的软件也可以拿来收集crash日志，比如Razor,QuincyKit（git链接）等，
这些软件收集crash的原理其实大同小异，都是根据系统产生的crash日志进行了一次提取或封装，
然后将封装后的crash文件上传到对应的服务端进行解析处理。很多商业软件都采用了Plcrashreporter这个
开源工具来上传和解析crash，比如HockeyApp,Flurry和crittercism等。

由于自己的crash信息太长，找了一张示例：　　
1. crash标识是应用进程产生crash时的一些标识信息，
它描述了该crash的唯一标识（E838FEFB-ECF6-498C-8B35-D40F0F9FEAE4），
所发生的硬件设备类型（iphone3,1代表iphone4），以及App进程相关的信息等；
2. 基本信息描述的是crash发生的时间和系统版本；　　
3. 异常类型描述的是crash发生时抛出的异常类型和错误码；　　
4. 线程回溯描述了crash发生时所有线程的回溯信息，每个线程在每一帧对应的函数调用信息（这里由于空间限制没有全部列出）；　　5. 二进制映像是指crash发生时已加载的二进制文件。以上就是一份crash日志包含的所有信息，
接下来就需要根据这些信息去解析定位导致crash发生的代码逻辑， 这就需要用到符号化解析的过程（洋名叫：symbolication)。

## 支付宝SDK使用
使用支付宝进行一个完整的支付功能，大致有以下步骤：向支付宝申请, 与支付宝签约，
获得商户ID（partner）和账号ID（seller）和私钥(privateKey)。下载支付宝SDK，生成订单信息,
签名加密调用支付宝客户端，由支付宝客户端跟支付宝安全服务器打交道。支付完毕后,
支付宝客户端会自动跳回到原来的应用程序，在原来的应用程序中显示支付结果给用户看。
**集成之后可能遇到的问题**
1. 集成SDK编译时找不到 openssl/asn1.h 文件
解决方案：Build Settings --> Search Paths --> Header Search paths : $(SRCROOT)/支付宝集成/Classes/Alipay
2. 链接时：找不到 SystemConfiguration.framework 这个库
解决方案：打开支付宝客户端进行支付(用户没有安装支付宝客户端,直接在应用程序中添加一个WebView,
  通过网页让用户进行支付)
// 注意:如果是通过网页支付完成,那么会回调该block:callback
```objc
 [[AlipaySDK defaultService] payOrder:orderString fromScheme:@"jingdong" callback:^(NSDictionary *resultDic) { }];
```
在AppDelegate.m
```objc
// 当通过别的应用程序,将该应用程序打开时,会调用该方法
- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<NSString *,id> *)options{ // 当用户通过支付宝客户端进行支付时,会回调该block:standbyCallback
[[AlipaySDK defaultService] processOrderWithPaymentResult:url standbyCallback:^(NSDictionary *resultDic) { NSLog(@"result = %@",resultDic); }]; return YES;}
```
## iOS可执行文件瘦身方法
- [iOS微信安装包瘦身](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207986417&idx=1&sn=77ea7d8e4f8ab7b59111e78c86ccfe66&3rd=MzA3MDU4NTYzMw==&scene=6#rd)
- [iOS APP可执行文件的组成](http://blog.cnbang.net/tech/2296/)
- [iOS可执行文件瘦身方法](http://blog.cnbang.net/tech/2544/)

## IPV6
- [iOS应用支持IPV6，就那点事儿](http://www.jianshu.com/p/a6bab07c4062)

**DNS64**
DNS64说白了是用来帮助host获取IPv6地址的，传统的DNS服务器可以把域名转换成IPv4地址，但我们的iPhone设备如果处于IPv6环境下，只能去获取IPv6的地址。DNS64就像一个中间代理，把传统服务器返回的IPv4地址通过特殊的映射方式转换成一个看着像IPv6地址的地址（IPv4的核，IPv6的壳），转换其实很简单，用公式可以这样表达：
>64:ff9b::IPv4 = IPv6

**NAT64**
DNS64帮助拿到IPv6的地址后，接下来就是NAT64登场，帮助IPv6的Packet顺利接入IPv4的公网当中。IPv4的公网环境路由器只认识IPv4的地址，所有这里当然也需要一个中间设备来做协议转换。NAT64就扮演这个角色。
## UIScrollView调优
- [UIScrollView调优——节省超过50%内存](http://www.jianshu.com/p/a7698be04d3f)

## iOS 保持界面流畅的技巧
- [保持界面流畅的技巧](http://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
- [Designing for iOS: Graphics &amp; Performance](https://robots.thoughtbot.com/designing-for-ios-graphics-performance)
