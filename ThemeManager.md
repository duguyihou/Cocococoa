>OS中实现主题更换的核心思路为：
- 资源按主题放置：相同功能的资源名称相同，放在不同的主题路径或者前缀使用主题名。
- 增加中间层，隔离不同主题相同功能资源使用的变化。

## 1. 主题管理
主题的特性导致代码不关心资源的表现是什么，只关心资源的功能，而主题是易变化的，因此需要将易变化的部分抽离出来，整合到一个管理者中，主题的变化在管理者中完成，而不影响资源使用的地方。而且**这个管理者是全局唯一的，因此使用单例**。
```objc
+ (ThemeManager *)sharedInstance
{
    static ThemeManager *sharedInstance = nil;
    if (sharedInstance == nil)
    {
        sharedInstance = [[ThemeManager alloc] init];
    }
    return sharedInstance;
}
```
主题中的资源使用plist进行存储，颜色的RGBA值跟字体的信息可以直接存入plist，而图片则可以存入图片的位置。按主题命名plist文件，ThemeManager的初始化跟主题更换就从main bundle中按主题名字读取对应的plist文件。
```objc
- (id)init
{
    if (self = [super init])
    {
        NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
        NSString *themeName = [defaults objectForKey:@"theme"] ?: @"default";

        NSString *path = [[NSBundle mainBundle] pathForResource:themeName ofType:@"plist"];
        self.theme = [NSDictionary dictionaryWithContentsOfFile:path];
    }
    return self;
}
```
通过ThemeManager得到对应主题下的资源。
```objc
/ 直接使用资源：
UIImage *image = [UIImage imageNamed:@"xxx_btn_background"];
// 通过主题管理器使用资源：
NSDictionary *theme = [ThemeManager sharedInstance].theme;
NSString *imageName = [theme objectForKey:@"xxx_btn_background"];
UIImage *image = [UIImage imageNamed:imageName];
```
上面的代码在使用时还是有些复杂，代码只关心资源的功能，不关系也不应该关心取资源的细节，因此应在ThemeManager对取资源进行如下封装：
```objc
- (UIImage *)imageForKey:(NSString *)key;
```
在使用主题中的资源时，代码就变成:
```objc
UIImage *image = [[ThemeManager sharedInstance] imageForKey:@"xxx_btn_background"];
```

## 2. 资源的放置
当系统将主题相关的资源文件部署到ios设备中时，在默认情况下，系统会将所有的资源plat平铺拷贝到mainBundle目录下，即使你的资源是按文件夹来组织的。（我们可以在模拟器中查看Bundle的情况，模拟器的路径是：~/Library/Application Support/iPhone Simulator）

因此，在将资源文件加入到工程时，不要选默认的”Recursively create groups for any add folders”，要选择“Create Folder Reference for any add folders”，这样才能保证资源文件按照原有文件夹的组织格式被拷贝到mainBundle中。

关于上述的两个选项，就涉及到Xcode的Group（黄色）跟Folder Reference（蓝色）的概念了，参见从别处摘抄来的理解：
>XCode项目中的文件夹分成两类: group 和 directory reference, 分别是虚结构和实结构. 黄颜色的 group 是默认的格式, 它的结构和磁盘上的文件夹毫无关系, 仅仅表示资源的逻辑组织结构, 这在管理源文件是非常方便. 同一段代码可以被很多项目使用, 也可能只使用一个目录的部分文件, 它不需要被拷贝到当前项目中, 但可以在当前项目中保持一个清晰的逻辑结构. 而且引用头文件时不需要指明复杂的层次结构, 因为这些文件在XCode看来是 flat 的, 即它们处在同一层文件夹里.
但是 group 带来便利的同时也导致更加棘手的麻烦, 文件重名冲突问题; 尤其当你要使用上千个资源文件时, 这种问题已经极难避免; 而且, 资源文件一般是要拷贝到目标程序中的, 虽然它们在项目中可以有结构的组织, 但是复制到程序中时将会 flat 地输出到程序的根目录中, 这将是怎样的一个灾难! 同时, 如果你在外部向文件夹中加入了上百幅图片, 你不得不把它们再向xcode中加入一遍. 归根结底, 还要求助于我们传统的蓝色的 directory reference。

## 3. 主题更换通知
对于没有显示的界面，更换主题是不需要通知的，因为在取资源时是根据当前主题取的，但是对于正在显示的界面，更换主题时就需要进行通知，让界面重新取资源后再重绘。由于这类通知是全局性的，因此应该使用NSNotification实现通知机制。

在ThemeManager的`changeTheme`中调用`[NSNotificaitonCenter defaultCenter]`的`postNotificationName:object:`发出通知，而在各个涉及到主题更换的ViewController中使用`addObserver:selector:name:object:`监听通知事件。

## 4. 总结
其实主题的设计思路跟类簇很像，例如对于NSNumber，不同类型的数据其实真正返回的是NSNumber相对于此类型的子类，但是对于NSNumber的使用者而言，其并不关心NSNumber返回的具体子类是什么，只要满足NSNumber定义的接口就行。设计总是类似的，针对易变化的部分，增加一个中间层（接口）将易变化的部分封装起来，提供给使用者稳定不易变的服务。

整理自[主题更换的设计思路](http://joeshang.github.io/2014/12/22/theme-change-architecture/)
