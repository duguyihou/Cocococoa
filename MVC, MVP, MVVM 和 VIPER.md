## MV(X) 的基本要素
现在我们面对架构设计模式的时候有了很多选择：
- MVC
- MVP
- MVVM
- VIPER

首先前三种模式都是把所有的实体归类到了下面三种分类中的一种：
- Models（模型） — 数据层，或者负责处理数据的 数据接口层。比如 Person 和 PersonDataProvider 类
- Views（视图） - 展示层(GUI)。对于 iOS 来说所有以 UI 开头的类基本都属于这层。
- Controller/Presenter/ViewModel（控制器/展示器/视图模型） - 它是 Model 和 View 之间的胶水或者说是中间人。一般来说，当用户对 View 有操作时它负责去修改相应 Model；当 Model 的值发生变化时它负责去更新对应 View。
将实体进行分类之后我们可以：
- 更好的理解
- 重用（主要是 View 和 Model）
- 对它们独立的进行测试
- 让我从 MV(X) 系列开始讲起，最后讲 VIPER。

## VIPER - 把搭建乐高积木的经验应用到 iOS 应用的设计上
VIPER从另一个角度对职责进行了划分，这次划分了 五层。
- Interactor（交互器） - 包括数据（Entities）或者网络相关的业务逻辑。比如创建新的 entities 或者从服务器上获取数据；要实现这些功能，你可能会用到一些服务和管理（Services and Managers）：这些可能会被误以为成是外部依赖东西，但是它们就是 VIPER 的 Interactor 模块。
- Presenter（展示器） - 包括 UI（but UIKit independent）相关的业务逻辑，可以调用 Interactor 中的方法。
- Entities（实体） - 纯粹的数据对象。不包括数据访问层，因为这是 Interactor 的职责。
- Router（路由） - 负责 VIPER 模块之间的转场

实际上 VIPER 模块可以只是一个页面（screen），也可以是你应用里整个的用户使用流程（the whole user story）- 比如说「验证」这个功能，它可以只是一个页面，也可以是连续相关的一组页面。你的每个「乐高积木」想要有多大，都是你自己来决定的。

如果我们把 VIPER 和 MV(X) 系列做一个对比的话，我们会发现它们在职责划分上面有下面的一些区别：

- Model（数据交互）的逻辑被转移到了 Interactor 里面，Entities 只是一个什么都不用做的数据结构体。
- Controller/Presenter/ViewModel 的职责里面，只有 UI 的展示功能被转移到了 Presenter 里面。Presenter 不具备直接更改数据的能力。
- VIPER 是第一个把导航的职责单独划分出来的架构模式，负责导航的就是 Router 层。

>如何正确的使用导航（doing routing）对于 iOS 应用开发来说是一个挑战，MV(X) 系列的架构完全就没有意识到（所以也不用处理）这个问题。
