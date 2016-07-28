## sprintf,strcpy,memcpy使用上有什么要注意的地方。

1. sprintf是格式化函数。将一段数据通过特定的格式，格式化到一个字符串缓冲区中去。 sprintf格式化的函数的长度不可控，有可能格式化后的字符串会超出缓冲区的大小，造成溢出。
2. strcpy是一个字符串拷贝的函数，它的函数原型为strcpy(char _dst, const char_ src 将src开始的一段字符串拷贝到dst开始的内存中去，结束的标志符号为 '\0'，由于拷贝的长度不是由我们自己控制的， 所以这个字符串拷贝很容易出错。
3. memcpy是具备字符串拷贝功能的函数，这是一个内存拷贝函数，它的函数原型 为memcpy(char _dst, const char_ src, unsigned int len);将长度为len的一段内存， 从src拷贝到dst中去，这个函数的长度可控。但是会有内存叠加的问题。
