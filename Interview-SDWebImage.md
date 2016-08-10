## SDWebImage里面给UIImageView加载图片的逻辑是什么样的？

**[SDWebImage 学习笔记](https://everettjf.github.io/2016/04/03/learn-sdwebimage)**


**options所有选项：**
```objc
     //失败后重试
     SDWebImageRetryFailed = 1 << 0,

     //UI交互期间开始下载，导致延迟下载比如UIScrollView减速。
     SDWebImageLowPriority = 1 << 1,

     //只进行内存缓存
     SDWebImageCacheMemoryOnly = 1 << 2,

     //这个标志可以渐进式下载,显示的图像是逐步在下载
     SDWebImageProgressiveDownload = 1 << 3,

     //刷新缓存
     SDWebImageRefreshCached = 1 << 4,

     //后台下载
     SDWebImageContinueInBackground = 1 << 5,

     //NSMutableURLRequest.HTTPShouldHandleCookies = YES;

     SDWebImageHandleCookies = 1 << 6,

     //允许使用无效的SSL证书
     //SDWebImageAllowInvalidSSLCertificates = 1 << 7,

     //优先下载
     SDWebImageHighPriority = 1 << 8,

     //延迟占位符
     SDWebImageDelayPlaceholder = 1 << 9,

     //改变动画形象
     SDWebImageTransformAnimatedImage = 1 << 10,
```

**SDWebImage内部实现过程**

1. 入口 setImageWithURL:placeholderImage:options: 会先把 placeholderImage 显示，
然后 SDWebImageManager 根据 URL 开始处理图片。
2. 进入 SDWebImageManager-downloadWithURL:delegate:options:userInfo:，
交给 SDImageCache 从缓存查找图片是否已经下载        queryDiskCacheForKey:delegate:userInfo:.
3. 先从内存图片缓存查找是否有图片，如果内存中已经有图片缓存，SDImageCacheDelegate 回调
imageCache:didFindImage:forKey:userInfo: 到 SDWebImageManager。
4. SDWebImageManagerDelegate 回调 webImageManager:didFinishWithImage: 到
UIImageView+WebCache 等前端展示图片。
5. 如果内存缓存中没有，生成 NSInvocationOperation 添加到队列开始从硬盘查找图片是否已经缓存。
6. 根据 URLKey 在硬盘缓存目录下尝试读取图片文件。这一步是在 NSOperation 进行的操作，
所以回主线程进行结果回调 notifyDelegate:。
7. 如果上一操作从硬盘读取到了图片，将图片添加到内存缓存中（如果空闲内存过小，会先清空内存缓存）。
SDImageCacheDelegate 回调 imageCache:didFindImage:forKey:userInfo:。进而回调展示图片。
8. 如果从硬盘缓存目录读取不到图片，说明所有缓存都不存在该图片，需要下载图片，
回调 imageCache:didNotFindImageForKey:userInfo:。
9. 共享或重新生成一个下载器 SDWebImageDownloader 开始下载图片。
10. 图片下载由 NSURLConnection 来做，实现相关 delegate 来判断图片下载中、下载完成和下载失败。
11. connection:didReceiveData: 中利用 ImageIO 做了按图片下载进度加载效果。
12. connectionDidFinishLoading: 数据下载完成后交给 SDWebImageDecoder 做图片解码处理。
13. 图片解码处理在一个 NSOperationQueue 完成，不会拖慢主线程 UI。
如果有需要对下载的图片进行二次处理，最好也在这里完成，效率会好很多。
14. 在主线程 notifyDelegateOnMainThreadWithInfo: 宣告解码完成，imageDecoder:didFinishDecodingImage:userInfo: 回调给 SDWebImageDownloader。
15. imageDownloader:didFinishWithImage: 回调给 SDWebImageManager 告知图片下载完成。
16. 通知所有的 downloadDelegates 下载完成，回调给需要的地方展示图片。
17. 将图片保存到 SDImageCache 中，内存缓存和硬盘缓存同时保存。
写文件到硬盘也在以单独 NSInvocationOperation 完成，避免拖慢主线程。
18. SDImageCache 在初始化的时候会注册一些消息通知，在内存警告或退到后台的时候清理内存图片缓存，
应用结束的时候清理过期图片。
19. SDWI 也提供了 UIButton+WebCache 和 MKAnnotationView+WebCache，方便使用。
20. SDWebImagePrefetcher 可以预先下载图片，方便后续使用。

从上面流程可以看出，当你调用setImageWithURL:方法的时候，他会自动去给你干这么多事，
当你需要在某一具体时刻做事情的时候，你可以覆盖这些方法。比如在下载某个图片的过程中要响应一个事件，就覆盖这个方法：
```objc
    //覆盖方法，指哪打哪，这个方法是下载imagePath2的时候响应
    SDWebImageManager *manager = [SDWebImageManager sharedManager];

    [manager downloadImageWithURL:imagePath2 options:SDWebImageRetryFailed progress:^(NSInteger receivedSize, NSInteger expectedSize) {

        NSLog(@"显示当前进度");

    } completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {

        NSLog(@"下载完成");
    }];
```

### SDWebImage库的作用
通过对UIImageView的类别扩展来实现异步加载替换图片的工作。

主要用到的对象：
1. UIImageView (WebCache)类别，入口封装，实现读取图片完成后的回调
2. SDWebImageManager，对图片进行管理的中转站，记录那些图片正在读取。
向下层读取Cache（调用SDImageCache），或者向网络读取对象（调用SDWebImageDownloader） 。
实现SDImageCache和SDWebImageDownloader的回调。
3. SDImageCache，根据URL的MD5摘要对图片进行存储和读取（实现存在内存中或者存在硬盘上两种实现）
实现图片和内存清理工作。
4. SDWebImageDownloader，根据URL向网络读取数据（实现部分读取和全部读取后再通知回调两种方式）

其他类：
SDWebImageDecoder，异步对图像进行了一次解压⋯⋯
1. SDImageCache是怎么做数据管理的?

SDImageCache分两个部分，一个是内存层面的，一个是硬盘层面的。内存层面的相当是个缓存器，
以Key-Value的形式存储图片。当内存不够的时候会清除所有缓存图片。用搜索文件系统的方式做管理，
文件替换方式是以时间为单位，剔除时间大于一周的图片文件。当SDWebImageManager向SDImageCache要资源时，
先搜索内存层面的数据，如果有直接返回，没有的话去访问磁盘，将图片从磁盘读取出来，然后做Decoder，
将图片对象放到内存层面做备份，再返回调用层。

2. 为啥必须做Decoder?
由于UIImage的imageWithData函数是每次画图的时候才将Data解压成ARGB的图像，所以在每次画图的时候，
会有一个解压操作，这样效率很低，但是只有瞬时的内存需求。为了提高效率通过SDWebImageDecoder将包装在Data下的资源解压，
然后画在另外一张图片上，这样这张新图片就不再需要重复解压了。
这种做法是典型的空间换时间的做法。
