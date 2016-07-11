# load方法的执行顺序
- 执行子类的 load 之前，当父类未加载时，先执行父类的 load 方法。
- 分类的 load 方法统一在最后执行
- 优先满足以上两条，再满足按 Compile Sources 的顺序执行 load 方法。

# initialize 方法的执行顺序
先执行了 ClassFather(category) initialize，再执行了 ClassSon(category) initialize，而 ClassSon load 在后面执行。
也就是说 load 方法还未执行也不会影响到这个类的使用。
另一个现象是执行子类 initialize 的时候会先执行其父类的 initialize。且 category 的覆写效应对 load 方法无效，但对 initialize 方法有效。且按 Complile Sources 的顺序，ClassSon(category2) 先覆写了 ClassSon 的 initialize 方法，接着 ClassSon(category) 覆写了 ClassSon(category2) 的 initialize。
