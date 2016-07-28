# Interview-runtime
## 为什么其他语言里叫函数调用， objective c里则是给对象发消息（或者谈下对runtime的理解）

先来看看怎么理解发送消息的含义：

曾经觉得Objc特别方便上手，面对着 Cocoa 中大量 API，只知道简单的查文档和调用。还记得初学 Objective-C 时把[receiver message]当成简单的方法调用，而无视了"发送消息"这句话的深刻含义。 于是[receiver message]会被编译器转化为： objc_msgSend(receiver, selector) 如果消息含有参数，则为： `objc_msgSend(receiver, selector, arg1, arg2, ...)`

如果消息的接收者能够找到对应的selector，那么就相当于直接执行了接收者这个对象的特定方法； 否则，消息要么被转发，或是临时向接收者动态添加这个selector对应的实现内容，要么就干脆玩完崩溃掉。

现在可以看出[receiver message]真的不是一个简简单单的方法调用。因为这只是在编译阶段确定了 要向接收者发送message这条消息，而receive将要如何响应这条消息，那就要看运行时发生的情况来决定了。

Objective-C 的 Runtime 铸就了它动态语言的特性，这些深层次的知识虽然平时写代码用的少一些， 但是却是每个 Objc 程序员需要了解的。

Objc Runtime使得C具有了面向对象能力，在程序运行时创建，检查，修改类、对象和它们的方法。 可以使用runtime的一系列方法实现。

顺便附上OC中一个类的数据结构 /usr/include/objc/runtime.h

```objc
struct objc_class {
    Class isa OBJC_ISA_AVAILABILITY; //isa指针指向Meta Class，
    因为Objc的类的本身也是一个Object，为了处理这个关系，runtime就创造了Meta Class，
    当给类发送[NSObject alloc]这样消息时，实际上是把这个消息发给了Class Object

    #if !__OBJC2__
    Class super_class OBJC2_UNAVAILABLE; // 父类
    const char *name OBJC2_UNAVAILABLE; // 类名
    long version OBJC2_UNAVAILABLE; // 类的版本信息，默认为0
    long info OBJC2_UNAVAILABLE; // 类信息，供运行期使用的一些位标识
    long instance_size OBJC2_UNAVAILABLE; // 该类的实例变量大小
    struct objc_ivar_list *ivars OBJC2_UNAVAILABLE; // 该类的成员变量链表
    struct objc_method_list **methodLists OBJC2_UNAVAILABLE; // 方法定义的链表
    struct objc_cache *cache OBJC2_UNAVAILABLE; // 方法缓存，对象接到一个消息会根据isa
    指针查找消息对象，这时会在method       Lists中遍历，如果cache了，常用的方法调用时就能够提高调用的效率。
    struct objc_protocol_list *protocols OBJC2_UNAVAILABLE; // 协议链表
    #endif

    } OBJC2_UNAVAILABLE;
```

OC中一个类的对象实例的数据结构（/usr/include/objc/objc.h）:

```objc
typedef struct objc_class *Class;

/// Represents an instance of a class.

struct objc_object {

    Class isa  OBJC_ISA_AVAILABILITY;

};

/// A pointer to an instance of a class.

typedef struct objc_object *id;
```

向object发送消息时，Runtime库会根据object的isa指针找到这个实例object所属于的类， 然后在类的方法列表以及父类方法列表寻找对应的方法运行。id是一个objc_object结构类型的指针， 这个类型的对象能够转换成任何一种对象。

然后再来看看消息发送的函数：objc_msgSend函数

在引言中已经对objc_msgSend进行了一点介绍，看起来像是objc_msgSend返回了数据，其实objc_msgSend 从不返回数据而是你的方法被调用后返回了数据。下面详细叙述下消息发送步骤：

检测这个 selector 是不是要忽略的。比如 Mac OS X 开发，有了垃圾回收就不理会 retain,release 这些函数了。 检测这个 target 是不是 nil 对象。ObjC 的特性是允许对一个 nil 对象执行任何一个方法不会 Crash，因为会被忽略掉。 如果上面两个都过了，那就开始查找这个类的 IMP，先从 cache 里面找，完了找得到就跳到对应的函数去执行。 如果 cache 找不到就找一下方法分发表。 如果分发表找不到就到超类的分发表去找，一直找，直到找到NSObject类为止。 如果还找不到就要开始进入动态方法解析了，后面会提到。

后面还有： 动态方法解析resolveThisMethodDynamically 消息转发forwardingTargetForSelector

## 对runtime的理解

1. 消息是如何转发的？ 动态解析过程大致是这样的：通过resolveInstanceMethod允许开发者决定是否动态添加方法，若返回NO， 就直接进入doesNotRecognizeSelector，流程结束，否则需要通过class_addMethod动态添加方法 并返回YES并进入下一步。forwardingTargetForSelector是第二步，允许开发者决定将由哪个对象响应这个selector， 如果返回nil，则直接进入doesNotRecognizeSelector，流程结束，否则需要返回一个对象，但不能是self。 进入第三步指定方法签名methodSignatureForSelector，若返回nil， 则直接进入doesNotRecognizeSelector且流程结束，否则指定签名，并进入下一步forwardInvocation。 forwardInvocation允许开发者修改响应者、方法实现等。若没有实现forwardInvocation， 则直接进入doesNotRecognizeSelector，流程结束。

2. 方法调用会被缓存吗？如何缓存过，又是如何查找的呢？ 方法是会缓存进来了，不然下次再调用又要重新查一次，效率是不高的。采用散列（哈希）的方式来缓存， 查询的效率是比较高的，因此内部会采用散列缓存起来。

3. 对象的内存是如何布局的？ 成员变量（包括父类）都保存在对象本身的存储空间内；本类的实例方法保存在类对象中， 本类的类方法保存在元类对象中；父类的实例方法保存在各级super class中， 父类的类方法保存在各级super meta class中。

4. runtime有哪些应用场景？

5. 给category添加属性

6. Method-Swizzling hook方法，然后交换方法实现来达到调用系统方法之前先做一些额外的处理

7. 埋点处理

8. 字典与模型互转

9. 模型自动获取所有属性并转换成SQL语句操作数据库

## oc是动态运行时语言是什么意思?

多态。 主要是将数据类型的确定由编译时，推迟到了运行时。 这个问题其实浅涉及到两个概念，运行时和多态。 简单来说，运行时机制使我们直到运行时才去决定一个对象的类别，以及调用该类别对象指定方法。 多态：不同对象以自己的方式响应相同的消息的能力叫做多态。意思就是假设生物类(life)都用有一个相同的方法-eat; 那人类属于生物，猪也属于生物，都继承了life后，实现各自的eat，但是调用是我们只需调用各自的eat方法。 也就是不同的对象以自己的方式响应了相同的消息(响应了eat这个选择器)。

### 关于多态性

多态，子类指针可以赋值给父类。 这个题目其实可以出到一切面向对象语言中， 因此关于多态，继承和封装基本最好都有个自我意识的理解，也并非一定要把书上资料上写的能背出来

## 对于语句NSString*obj = [[NSData alloc] init]; obj在编译时和运行时分别时什么类型的对象?

编译时是NSString的类型;运行时是NSData类型的对象
