## 数据持久化

1. 通过web服务，保存在服务器上
2. 通过NSCoder固化机制，将对象保存在文件中
3. 通过SQlite或CoreData保存在文件数据库中

### ios 平台怎么做数据的持久化?coredata 和sqlite有无必然联系？coredata是一个关系型数据库吗？

iOS 中可以有四种持久化数据的方式：属性列表(plist)、对象归档、 SQLite3 和 Core Data； core data 可以使你以图形界面的方式快速的定义 app 的数据模型，同时在你的代码中容易获取到它。 coredata 提供了基础结构去处理常用的功能，例如保存，恢复，撤销和重做，允许你在 app 中继续创建新的任务。 在使用 core data 的时候，你不用安装额外的数据库系统，因为 core data 使用内置的 sqlite 数据库。 core data 将你 app 的模型层放入到一组定义在内存中的数据对象。 coredata 会追踪这些对象的改变， 同时可以根据需要做相反的改变，例如用户执行撤销命令。当 core data 在对你 app 数据的改变进行保存的时候， core data 会把这些数据归档，并永久性保存。 mac os x 中sqlite 库，它是一个轻量级功能强大的关系数据引擎， 也很容易嵌入到应用程序。可以在多个平台使用， sqlite 是一个轻量级的嵌入式 sql 数据库编程。 与 core data 框架不同的是， sqlite 是使用程序式的， sql 的主要的 API 来直接操作数据表。 Core Data 不是一个关系型数据库，也不是关系型数据库管理系统 (RDBMS) 。虽然 Core Dta 支持SQLite 作为一种存储类型，但它不能使用任意的 SQLite 数据库。 Core Data 在使用的过程种自己创建这个数据库。 Core Data 支持对一、对多的关系。

### 和coredata一起有哪几种持久化存储机制?

存入到文件、 存入到NSUserDefaults(系统plist文件中)、存入到Sqlite文件数据库

### 什么是NSManagedObject模型?

NSManagedObject是NSObject的子类 ，也是coredata的重要组成部分，它是一个通用的类,实现了core data 模型层所需的基本功能，用户可通过子类化NSManagedObject，建立自己的数据模型。

### 什么是NSManagedobjectContext?

NSManagedobjectContext对象负责应用和数据库之间的交互。
### 你实现过多线程的Core Data么？NSPersistentStoreCoordinator，NSManagedObjectContext

和NSManagedObject中的哪些需要在线程中创建或者传递？你是用什么样的策略来实现的？ <https://onevcat.com/2014/03/common-background-practices/>
