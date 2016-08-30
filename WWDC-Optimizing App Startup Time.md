# Session 406 Optimizing App Startup Time
> 一个App完整的启动时间应该保证400ms之内,而若超过20s后还未完全启动App,那么App进程就会被系统杀死.而如何Debug和优化应用启动的时间,官方提出一系列方法来关注应用启动时执行main()前究竟干了些什么.而通过这个Session,你会了解到以下内容:

## 测量Pre-main Time
一个App在执行main函数前包括app delegate的系列方法如`applicationWillFinishLaunching`时,会做许多系统级别的准备.而在iOS10之前,开发者很难清楚自己App为何启动加载慢.而通过在工程的scheme中添加环境变量**DYLD_PRINT_STATISTICS**,设置Value为1,App启动加载时就会有启动过程的日志输出. 现在(iOS 10之后)Apple对DYLD_PRINT_STATISTICS的日志输出结果进行了简化,使得更容易让开发者理解.

## 启动优化
- **对动态库加载的时间优化**.每个App都进行动态库加载,其中系统级别的动态库占据了绝大数,而针对系统级别的动态库都是经过系统高度优化的,不用担心时间的花费.开发者应该关注于自己集成到App的那些动态库,这也是最能消耗加载时间的地方.对此Apple建议**减少在App里开发者的动态库集成或者有可能地将其多个动态库最终集成一个动态库后进行导入, 尽量保证将App现有的非系统级的动态库个数保证在6个以内**.
- **减少Appp的Objective-C类,分类和的唯一Selector的个数**.这样做主要是为了加快程序的整个动态链接, 在进行动态库的重定位和绑定(Rebase/binding)过程中减少指针修正的使用,加快程序机器码的生成.
- **减少Objc运行初始化的时间花费**.主要是类的注册,分类的注册,唯一选择器的存在,以及涉及子父类内存布局的`Non Fragile ivars`偏移的更新,都会影响Objective-C运行时初始化的时间消耗.
- 使用`initialize`方法进行必要的初始化工作.用`+initialize`方法替换调用原先在OC的`+load`方法中执行初始代码工作,从而加快所有类文件的加载速度.

## 结尾
- 使用**DYLD_PRINT_STATISTICS**测试启动加载时间
- 减少自定义的动态库集成
- 精简原有的Objective-C类和代码
- 移除静态的初始化操作
- 使用更多的Swift代码
