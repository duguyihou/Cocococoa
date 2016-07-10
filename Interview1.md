## 不用临时变量实现swap(a, b)
### 位运算
通过位运算实现交换，首先将a = a ^^ b，再执行第二条b = a ^^ b就可以得到b = a，但是此时a还不是了，再执行a = a ^^ b，那么a就变成b了
```objc
NSInteger a = 19;
NSInteger b = 99;

a = a ^ b;
b = a ^ b;
a = a ^ b;

// a = 99, b = 19
NSLog(@"a = %ld, b = %ld", a, b);
```
### 算术运算
实现原理：将a、b看作数轴上的两个点，利用差值来计算两者的距离并保存到其中一个变量中，再利用这个距离与a、b的差值或者和计算出交换后的a、b值：
```objc
NSInteger a = 19;
NSInteger b = 99;

// 计算a、b两者之间的距离并保存到a
a = labs(a - b);
// a、b两者之间的距离再减去b就是最初a的值，此时就交换给了b
b = labs(b - a);
// 当前的a就是最初a、b之间的距离，再加上b（也就是原来的a的值），就是b的值
// 将赋值给a就完成了a与b的交换
a = a + b;

NSLog(@"a = %ld, b = %ld", a, b);
```
这种方式也不需要借助临时变量，但是这种方式有很大的局限性，比如
a、b本是不溢出的，但是a+b后可能会溢出。
## 二维有序数组查找数字

```
[
  [1,   3,  5,  7],
  [10, 11, 16, 20],
  [23, 30, 34, 50]
]

```

假设，我们查找数字34，那么我们可以这么找：

先查找第一行最后一个元素，若等于34，则查找成功；若>34，则必定就是这一行或者不存在；若<34，则说明必定是后面的行中或者根本不存在。
若第一步最后一个元素>34，则可以采用折半查找这一行。
若第一步最后一个元素<34，则说明不在第一行，则继续查找下一行最后一个元素，直到找到>34为止或者直接查找结束。
当查找到最后一个元素>34的行时，折半查找这一行即可。
时间复杂度分析：我们查找第一行最后一个元素，最坏情况下是column次，而折半查找是log2^^n
，那么最终时间复杂度为：column * log2^^n
```objc
const int rows = 3;
const int cols = 4;
int array[rows][cols] = {{1, 3, 5, 7}, {10, 11, 16, 20}, {23, 30, 34, 50}};
int searchNum = 34;
// 是否查找成功
int found = 0;

// 查找第一行最后一个元素
for (int i = 0; i < rows; ++i) {
  int last = array[i][cols - 1];

  if (last == searchNum) {
    NSLog(@"找到了，位置为：(%d, %d)", i, cols - 1);
    break;
  }
  // 说明待查找的元素就在这一行，或者根本不存在
  else if (last > searchNum) {
    int mid = 0;
    int low = 0;
    int high = cols - 1;

    while (low <= high) {
      mid = (low + high) / 2;

      if (array[i][mid] == searchNum) {
        found = YES;
        NSLog(@"找到了，位置为：(%d, %d)", i, cols - 1);
        return;
      } else if (array[i][mid] > searchNum) {
        high = mid - 1;
      } else {
        low = mid + 1;
      }
    }

    if (!found) {
      NSLog(@"查找失败了，元素并不在二维数组中");
    }
  }
}
```
## 亿级日志中，查找登陆次数最多的十个用户
假设只有一亿级日志，每条日志占用255字节，那么1亿就有1.0/4K 10000  10000 == 2500  10000M=>2.44  10000M == 24400M ~= 23.8G，可想而知是不能同时放入内存中的。
1. 我们可以通过分割成很多份，通过hash(key)对1024取模，分成1024个份，每份占用约23.8M，完全可以放内存中。
2. 然后分别对每一份操作，以登录次数作为hash的key，若key存在，则只是使key加1，若不存在则key值设置为1，只保留key值最大的10条记录，其余都丢掉（释放内存）
3. 当执行完毕后，内存中共有10 1024条记录，共10 0.25K * 1024 = 2.5M，因此内存使用上没有问题
4. 对这10 * 1024条记录排序，采用最小堆排序，或者采用红黑树（平衡二叉树）排序，提取前值最大的10条。

## 简述排序算法
* 快速排序：利用分而治之的思想，每一趟比较中保证左边的比基准数小，右边的都比基准数大。每一趟排序时，需要划分出基准数，通过partition函数来划分，得到基准后，再递归排序基准数左边的部分，递归排序基准数右边的部分。

* 堆排序：堆分为最大堆和最小堆。最大堆是每次调整堆后，保证堆顶元素是最大元素，然后将当前未有序的元素中的最后一个元素与之交换，保证右边不断有序，而左边无序元素越来越少，因此最大堆排序后是升序序列；最小堆是每次调整堆后，保证堆顶元素是最小元素，然后将当前未有序的元素中的最后一个元素与之交换，保证右边不断有序，而左边无序元素越来越少，因此最小堆排序后得到的是降序序列。

- 归并排序：采用分治法，通过将序列中分成子序列，递归归并成有序的若干个子序列，然后再二路归并成各个更大的子序列，最后再二路归并成最终序列。

<!-- - 冒泡排序：所谓冒泡，就像水中冒泡一样，将值小的不断将上浮，值大的不断往下沉。每一趟都将值最大的交换到这一趟的最后，保证待查找序列部分最后一个元素是最大值。 -->

插入排序 插入排序分为直接插入排序和折半插入排序。对于直接插入排序，每一趟都寻找a[i-1] > a[i]的，说明这时候是无序的，记录待排序值a[i]，然后将前面已有序的部分，移动位置，使a[i]值插入后，已有序部分依然有序；对于折半插入排序，每一趟都通过折半查找的方式来查找元素，然后移动位置，将之插入，使之有序，不过折半插入排序需要一个哨兵位置a[0]。

##  给一个字符串，如何判断它是否是合法的IP地址，比如 “192.168.1.1” 就是合法的。
首先，我们需要明确一个合法的IP格式应该是怎么样的，每个值是1-255，因此，我们可以通过字符串分割后，分别判断值是否在1-255即可。
### Objective-C实现

```objc
/**
 *    Judge whether the specified string is a valid form of IP or not.
 *
 *    @param ipString    The specified string to be checked.
 *
 *    @return YES if is a valid form of IP, otherwise NO.
 */
 - (BOOL)isValidIP:(NSString *)ipString {

 if (ipString.length < 7) {
 return NO;
 }

 NSArray *tempArray = [ipString componentsSeparatedByString:@"."];
 if (tempArray.count != 4) {
 return NO;
 }

 __block BOOL isValid = YES;
 [tempArray enumerateObjectsUsingBlock:^(NSString *_Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
 if (obj.integerValue < 0 || obj.integerValue > 255) {
 isValid = NO;
 *stop = YES;
 }

 NSString *string = [NSString stringWithFormat:@"%d",obj.intValue];
 if (![string isEqualToString:obj]) {
 isValid = NO;
 *stop = YES;
 }
 }];

 return isValid;
 }

```

### C实现
```c
/**
 *    Input：a string，judge whether it is a valid form of IP string
 *
 *    @param ipString
 *
 *    @return 1 if it is a valid form of IP string, otherwise 0
 */
int isValidIP(char *ipString) {
  int len = (int)strlen(ipString);

  if (len < 7 || len > 15) {
    return 0;
  }

  int itemValue = 0;
  int index = 1;
  for (int i = len - 1; i >= 0; --i) {
    char ch = ipString[i];

    if (ch >= '0' && ch <= '9') {
      itemValue += (ch - '0') * pow(10, index - 1);
      index++;
    } else if (ch == '.') {
      index = 1;
      if (itemValue < 1 || itemValue > 255) {
        return 0;
      }

      itemValue = 0;
    } else {
      return 0;
    }
  }

  if (itemValue < 1 || itemValue > 255) {
    return 0;
  }

  return 1;
}
```

## 大数相加的思路，动手写代码实现
大数相加的关键点是通过字符串来实现相加，以串最长的作为基准，将串短的高位补0，然后对位相加，并做好进位处理。
```objc
/**
 *    做两个超大数相加算法，采用0补高位的方式再做加法运算
 *
 *    @param lhsSource    大数1
 *    @param rhsSource    大数2
 *    @param result        接收结果，确保长度足够
 */
void addBigNumbers(char *lhsSource, char *rhsSource, char *result) {
  int lhsLen = (int)strlen(lhsSource);
  int rhsLen = (int)strlen(rhsSource);
  int len = lhsLen > rhsLen ? lhsLen : rhsLen;

  char *temp = malloc(sizeof(char *) * (len + 2));
  int i = lhsLen - 1;
  int j = rhsLen - 1;
  int k = 0;
  char lhsChar = '0';
  char rhsChar = '0';

  // 进位
  int carryBit = 0;
  int z = 0;

  while (i >= 0 || j >= 0) {
    // 串短的就以'0'补位，用于做加法运算
    if (i < 0) {
      lhsChar = '0';
    } else {
      lhsChar = lhsSource[i];
    }

    // 串短的就以'0'补位，用于做加法运算
    if (j < 0) {
      rhsChar = '0';
    } else {
      rhsChar = rhsSource[j];
    }

    // 对位相加，再加上进位值
    z = lhsChar - '0' + rhsChar - '0' + carryBit;
    // 有可能>=10，需要取做进位处理
    temp[k++] = z % 10 + '0';
    // 更新进位
    carryBit = z / 10;

    i--;
    j--;
  }

  // 全部相加完之后，有可能还有进位，需要将进位顶到高位
  while (carryBit > 0) {
    temp[k++] = carryBit % 10 + '0';
    carryBit /= 10;
  }

  // 我们借助了临时字符数组来存储计算结果，但是计算结果是倒序的，
  // 我们需要将计算结果变成正序
  k--;
  i = 0;
  while (k >= 0) {
    result[i++] = temp[k--];
  }

  // 别忘了添加上字符串结束标记符
  result[i] = '\0';

  // temp是自己在堆上申请的内存，记得释放
  free(temp);
}

```
写两个字符串相加：

```objc
char *lhsSource = "12368102369126318236218391231231232132132";
char *rhsSource = "9999999999999991232399999999999999999999999";
char result[100];
addBigNumbers(lhsSource, rhsSource, result);
NSLog(@"result = %s", result);

// print
// result = 10012368102369117550636218391231231232132131
```

## 简述TCP建立和关闭连接时，握手的过程。为什么前者是三次握手，后者需要四次？

TCP建立连接时，握手的过程大概如下：

- 客户端发送SYN到服务端
- 服务端发布SYN/ACK到客户端，此时开始建立连接
- 客户端发布ACK到服务端，此时正式建立好连接

客户端发送SYN到服务端，而服务端返回了客户端发过来的SYN，同时也返回ACK，那么客户端接收到之后，就可以确定服务端收到了SYN信号，而客户端接收到服务端返回来的ACK信号后，再将ACK信号发送到服务端，服务端就明确客户端收到了服务端发过去的信号。因此，这三次握手就可以确定了双方的身份。
TCP关闭连接时，握手的过程大致如下：
- 客户端发送FIN包到服务端：此时客户端进入FIN_WAIT_1等待对方确认状态
- 服务端返回ACK包到客户端：此时客户端结束FIN_WAIT_1状态，并进入FIN_WAIT_2状态，等待服务端的发过来的关闭请求
- 服务端发送FIN包到客户端：此时服务端进入CLOSE_WAIT状态，等待客户端确认关闭请求
- 客户端返回ACK包到服务端：此时服务端正式关闭，结束CLOSE_WAIT状态

TCP关闭连接之所以需要四次握手，是因为TCP连接是全双工，是双向的。

## 假设有10W条电话号码，如何通过输入电话号码的某一段内容，快速搜索出来。比如输入234，以下两个号码都会显示在搜索结果中：
```
123456789000
188888823400
```
> 用predicate正则

## 什么是Binary search tree? search的时间复杂度是多少？
Binary search tree:二叉搜索树。
主要由四个方法：（用C语言实现或者Python）
1. search：时间复杂度为O(h)，h为树的高度
2. traversal：时间复杂度为O(n)，n为树的总结点数。
3. insert：时间复杂度为O(h)，h为树的高度。
4. delete：最坏情况下，时间复杂度为O(h)+指针的移动开销。

可以看到，二叉搜索树的dictionary operation的时间复杂度与树的高度h相关。所以需要尽可能的降低树的高度，由此引出平衡二叉树Balanced binary tree。它要求左右两个子树的高度差的绝对值不超过1，并且左右两个子树都是一棵平衡二叉树。这样就可以将搜索树的高度尽量减小。常用算法有红黑树、AVL、Treap、伸展树等。

## 反转二叉树，不用递归
```
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
```
递归方式：
```swift
public class Solution {
public TreeNode invertTree(TreeNode root) {
    if (root == null) {
        return null;
    }
    root.left = invertTree(root.left);
    root.right = invertTree(root.right);
    TreeNode tmp = root.left;
    root.left = root.right;
    root.right = tmp;
    return root;
}
}
```
object-c实现：
```objc
/**
 * 翻转二叉树（又叫：二叉树的镜像）
 *
 * @param rootNode 根节点
 *
 * @return 翻转后的树根节点（其实就是原二叉树的根节点）
 */
 + (BinaryTreeNode *)invertBinaryTree:(BinaryTreeNode *)rootNode {
    if (!rootNode) {  return nil; }
    if (!rootNode.leftNode && !rootNode.rightNode) {  return rootNode; }
    [self invertBinaryTree:rootNode.leftNode];
    [self invertBinaryTree:rootNode.rightNode];
    BinaryTreeNode *tempNode = rootNode.leftNode;
    rootNode.leftNode = rootNode.rightNode;
    rootNode.rightNode = tempNode;
    return rootNode;
  }
```
非递归方式：
```
+ (BinaryTreeNode *)invertBinaryTree:(BinaryTreeNode *)rootNode {
if (!rootNode) {  return nil; }
if (!rootNode.leftNode && !rootNode.rightNode) {  return rootNode; }
NSMutableArray *queueArray = [NSMutableArray array]; //数组当成队列
[queueArray addObject:rootNode]; //压入根节点
while (queueArray.count > 0) {
    BinaryTreeNode *node = [queueArray firstObject];
    [queueArray removeObjectAtIndex:0]; //弹出最前面的节点，仿照队列先进先出原则
    BinaryTreeNode *pLeft = node.leftNode;
    node.leftNode = node.rightNode;
    node.rightNode = pLeft;

    if (node.leftNode) {
        [queueArray addObject:node.leftNode];
    }
    if (node.rightNode) {
        [queueArray addObject:node.rightNode];
    }

}

return rootNode;
}
```
