# Interview-objc

## 谈谈instancetype和id的异同
1. 相同点
都可以作为方法的返回类型

2. 不同点
- instancetype可以返回和方法所在类相同类型的对象，id只能返回未知类型的对象；
- instancetype只能作为返回值，不能像id那样作为参数
