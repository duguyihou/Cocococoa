使用NSURLSession时需要注意一个内存泄漏问题：
```
- (void)doNSURLSessionTest {
    NSURL *url = [NSURL URLWithString:@"https://www.github.com"];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession *session = [NSURLSession sessionWithConfiguration:config
                                                          delegate:self
                                                     delegateQueue:[NSOperationQueue mainQueue]];
    NSURLSessionTask *dataTask = [session dataTaskWithRequest:request];
    [dataTask resume];
}
```

初始化一个NSURLSession临时实例对象并由它发起一个网络请求。我们通过Instruments的Leaks工具会发现其存在内存泄漏和循环引用的地方.

通过NSURLSession的头文件我们发现，NSURLSession对于它的 delegate属性是强引用。这就意味着当session存在时，其delegate就不会被释放。另外，由session发起请求的缓存相关对象也会被其强引用并一直保留在内存中。

所以为了避免内存泄漏，根据Apple文档，当一个session不再使用时，我们应该调用finishTasksAndInvalidate或者invalidateAndCancel把session显式地置为无效(invalidated)，以释放对相关对象的引用。

最后，在一个 App 生命周期内，我们通常会初始化并配置好一个 NSURLSession对象，然后由它统一发起请求，一般不会显式把该session 置为无效（会在持有该session的类的dealloc方法里去释放它），所以建议采用单例的方式来使用NSURLSession(我们可以看到AFNetwoking的官方Demo也是通过单例来使用 AFSessionManager)，就不会出现上述内存泄漏问题。
- http://stackoverflow.com/questions/30106960/nsurlsession-memory-leak
- https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSession_class/#//apple_ref/occ/instm/NSURLSession/invalidateAndCancel
- http://stackoverflow.com/questions/28223345/memory-leak-when-using-nsurlsession-downloadtaskwithurl/35757989#35757989