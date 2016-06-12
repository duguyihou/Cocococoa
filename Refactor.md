# Refactor
Xcode提供了以下几个重构功能：
- Rename
- Extract
- Create Superclass
- Move Up
- Move Down
- Encapsulate

## Rename：重命名
几乎可以试用所有symbol：类名，方法名，函数名，属性名等。使用起来非常简单，选中一个要命名的符号后，选择rename，就会弹出一个输入框让你输入要想要的新名称，输入完成后点击preview可以预览一下。
## Extract：将代码抽取为一个单独的方法或函数
选中一段代码后（可以包括注释），选择Extract，会分析你选择的代码段后自动生成方法签名。你可以修改方法名，如果参数、返回值不正确也可以自己再修改。
如果选择了function，就是另外一种格式：
## Create Superclass：创建父类
创建一个当前类的父类。要注意的是要选中类名的时候才能成功触发。
需要注意的是预览界面最左边的导航区，选择中间一个是这次重构会影响到文件列表。可以点击到这个tab下查看其它类的改动。
## Move Up & Move Down
Move Up：可以将一个方法、实例变量移动到父类中去。触发时和重命名一样，要选中实例名或者方法名后才能正常使用。在category中不适用。

Move Down：相反，将选中的实例变量移动到子类中。是的，方法就不能移到子类了。逻辑上很难理解为什么是这样。但是苹果就是这么任性。
## Encapsulate：封装
这是一个令人怀念的词，多年后看到还是会想起期末考试里面向对象三大特点的填空题。
这个的作用是在你选中一个变量后，会自动帮你生成get、set方法。
因为在声明property时就已经自动生成了get、set方法。所以这个功能应该是有点过时了。

[相关链接:Refactoring: General Workflow](https://developer.apple.com/library/ios/recipes/xcode_help-source_editor/chapters/RefactorWorkflow.html#//apple_ref/doc/uid/TP40009975-CH17-SW1)
