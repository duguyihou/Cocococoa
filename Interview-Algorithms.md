## 不用临时变量实现swap(a, b)
**位运算**
通过位运算实现交换，首先将a = a ^^ b，再执行第二条b = a ^^ b就可以得到b = a，但是此时a还不是了，
再执行a = a ^^ b，那么a就变成b了
```objc
NSInteger a = 19;
NSInteger b = 99;

a = a ^ b;
b = a ^ b;
a = a ^ b;

// a = 99, b = 19
NSLog(@"a = %ld, b = %ld", a, b);
```
**算术运算**
实现原理：将a、b看作数轴上的两个点，利用差值来计算两者的距离并保存到其中一个变量中，再利用
这个距离与a、b的差值或者和计算出交换后的a、b值：
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

先查找第一行最后一个元素，若等于34，则查找成功；若>34，则必定就是这一行或者不存在；若<34，
则说明必定是后面的行中或者根本不存在。
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
假设只有一亿级日志，每条日志占用255字节，那么1亿就有1.0/4K 10000  10000 == 2500  
10000M=>2.44  10000M == 24400M ~= 23.8G，可想而知是不能同时放入内存中的。
1. 我们可以通过分割成很多份，通过hash(key)对1024取模，分成1024个份，每份占用约23.8M，完全可以放内存中。
2. 然后分别对每一份操作，以登录次数作为hash的key，若key存在，则只是使key加1，若不存在则key值设置为1，
只保留key值最大的10条记录，其余都丢掉（释放内存）
3. 当执行完毕后，内存中共有10 1024条记录，共10 0.25K * 1024 = 2.5M，因此内存使用上没有问题
4. 对这10 * 1024条记录排序，采用最小堆排序，或者采用红黑树（平衡二叉树）排序，提取前值最大的10条。

## 简述排序算法
* 快速排序：利用分而治之的思想，每一趟比较中保证左边的比基准数小，右边的都比基准数大。每一趟排序时，
需要划分出基准数，通过partition函数来划分，得到基准后，再递归排序基准数左边的部分，递归排序基准数右边的部分。

* 堆排序：堆分为最大堆和最小堆。最大堆是每次调整堆后，保证堆顶元素是最大元素，
然后将当前未有序的元素中的最后一个元素与之交换，保证右边不断有序，而左边无序元素越来越少，
因此最大堆排序后是升序序列；最小堆是每次调整堆后，保证堆顶元素是最小元素，然后将当前未有序的
元素中的最后一个元素与之交换，保证右边不断有序，而左边无序元素越来越少，因此最小堆排序后得到的是降序序列。

- 归并排序：采用分治法，通过将序列中分成子序列，递归归并成有序的若干个子序列，然后再二路归并成各个更大的子序列，
最后再二路归并成最终序列。

<!-- - 冒泡排序：所谓冒泡，就像水中冒泡一样，将值小的不断将上浮，值大的不断往下沉。
每一趟都将值最大的交换到这一趟的最后，保证待查找序列部分最后一个元素是最大值。 -->

插入排序 插入排序分为直接插入排序和折半插入排序。对于直接插入排序，每一趟都寻找a[i-1] > a[i]的，
说明这时候是无序的，记录待排序值a[i]，然后将前面已有序的部分，移动位置，使a[i]值插入后，
已有序部分依然有序；对于折半插入排序，每一趟都通过折半查找的方式来查找元素，然后移动位置，
将之插入，使之有序，不过折半插入排序需要一个哨兵位置a[0]。

##  给一个字符串，如何判断它是否是合法的IP地址，比如 “192.168.1.1” 就是合法的。
首先，我们需要明确一个合法的IP格式应该是怎么样的，每个值是1-255，因此，我们可以通过字符串分割后，
分别判断值是否在1-255即可。
**Objective-C实现**

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
 [tempArray enumerateObjectsUsingBlock:^(NSString *_Nonnull obj, NSUInteger idx,
   BOOL * _Nonnull stop) {
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

**C实现**
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

## 假设有10W条电话号码，如何通过输入电话号码的某一段内容，快速搜索出来。比如输入234，
以下两个号码都会显示在搜索结果中：
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

可以看到，二叉搜索树的dictionary operation的时间复杂度与树的高度h相关。所以需要尽可能的降低树的高度，
由此引出平衡二叉树Balanced binary tree。它要求左右两个子树的高度差的绝对值不超过1，
并且左右两个子树都是一棵平衡二叉树。这样就可以将搜索树的高度尽量减小。常用算法有红黑树、AVL、Treap、伸展树等。

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
**递归方式：**
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
**object-c实现：**
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
**非递归方式：**
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

## 一工人给老板打7天工要求一块金条 这金条只能切2次 工人每天要1/7金条 怎么分?
这道题解决的主要难点在于：不是给出去的就收不回来了，可以用交换的方法。　　

把金条分成三段（就是分两次，或者切两刀），分别是整根金条的1/7、2/7、 4/7。　　

第一天：给1/7的， 第二天：给2/7的，收回1/7的； 第三天，给1/7的； 第四天：给4/7的，
收回1/7和2/7的 ；第五天：给1/7的 ；第六天：给2/7的，收回1/7的；第七天发1/7。

## 有一栋楼共100层，一个鸡蛋从第N层及以上的楼层落下来会摔破， 在第N层以下的楼层落下不会摔破。
给你2个鸡蛋，设计方案找出N，并且保证在最坏情况下， 最小化鸡蛋下落的次数。(假设每次摔落时，
  如果没有摔碎，则不会给鸡蛋带来损耗)
[一道有趣的面试题](http://www.cricode.com/3558.html)

## 1-1000个连续自然数，然后从中随机去掉两个，再打乱顺序，要求只遍历一次，求出被去掉的两个数。

**求方程组的解**
遍历被打乱的数组时，计算value的累加值和value平方的累加值。结合未打乱之前的数组，
这样就能得出x+y = m与xx+yy = n两个方程，解这组方程即可算出被去掉的两个数。
这种方法比较容易理解，实现起来也比较简单

**使用异或**
先来说说异或的定义：两个二进制位不同的取1。再来说说异或的两个特性：顺序无关 / 对一个数异或两次等于没有异或。
顺序无关就是说异或的元素可以随意交换顺序，而不会影响结果。异或两次可以理解为+x和-x。
**计算出x^y的值**
首先，这两个数组(打乱前和打乱后)各自异或，也就是1^2^…^1000，得到两个异或值。
再对这两个异或值进行一次异或，这样就得到了x^y的指(重复部分互相抵消了)。
```objc
// 其实就是把数组的所有元素进行异或，重复部分互相抵消
result = 1^2^...^1000^1^2...^1000;
result = 1^1^2^2...^x...^y...^1000^1000;
result = x^y;
```
**获取计算出的异或值的1所在的位置，并继续异或**
因为x和y是两个不同的整数，所以这两个数的异或结果，转化为二进制的话，一定在某位是1，假设在第3位。
也就是说如果把原始数组按第3位是否为0进行划分，就可以分成两个数组，每个数组各包含一个被抽取的数。
如果打乱后的数组也按这个规则划分为两个数组，这样就得到了4个数组，其中两组是第3位为0，
另外两组是第3位为1。把第3位为0的两个数组所有元素进行异或就能得到被抽取的一个数，
同理也就能获得另外一个被抽取的数，于是问题解决。

**PHP的实现**
```objc
<?php
// 起始长度
$length = 10;

$arr = $arr_copy = range(1, $length);
// 将要被移除的两个数
$num1 = $num2 = 0;
// 两个数组异或再互相异或的结果
$num1_num2_xor = 0;
// 存放被pos分割的数字
$arr_0 = $arr_1 = $arr_copy_0 = $arr_copy_1 = array();

// 获取一个数字转化为二进制后1所在的位置
function get_pos($num)
{
	for($i=0 ;$i<10; $i++)
	{
		$b = pow(2, $i);
		$rs = $num&$b;
		if($rs % 2 == 0 && $rs != 0)
		{
			return $i;
		}
	}
	return 0;
}

// 进行异或计算
function do_xor($x, $y)
{
	return $x^$y;
}

function init()
{
	global $arr, $arr_copy, $num1, $num2, $num1_num2_xor, $length;

	$rand_index_1 = mt_rand(1, $length/2);
	$rand_index_2 = mt_rand($length/2+1, $length-1);

	// 获取两个随机数，然后去掉从数组中去掉它们
	$num1 = $arr[$rand_index_1];
	$num2 = $arr[$rand_index_2];

	unset($arr[$rand_index_1]);
	unset($arr[$rand_index_2]);

	cacl_num1_num2_xor();
	divide_by_pos(get_pos($num1_num2_xor));
	get_num();

}

// 获取两个数组各自异或再互相异或的结果
function cacl_num1_num2_xor()
{
	global $arr, $arr_copy, $num1_num2_xor;
	$arr_xor = array_reduce($arr, 'do_xor');
	$arr_copy_xor = array_reduce($arr_copy, 'do_xor');

	$num1_num2_xor = $arr_xor ^ $arr_copy_xor;
}

// 根据pos将两个数组再各自细分成两个数组
// 其中$arr_copy_0和$arr_copy_1各自包含了一个被抽取的数
function divide_by_pos($pos)
{
	global $arr, $arr_copy, $arr_0, $arr_1, $arr_copy_0, $arr_copy_1;
	$b = pow(2, $pos);

	foreach($arr as $val)
	{
		$rs = $val&$b;
		if($rs == 0)
		{
			$arr_0[] = $val;
		}
		else
		{
			$arr_1[] = $val;
		}
	}

	foreach($arr_copy as $val)
	{
		$rs = $val&$b;
		if($rs == 0)
		{
			$arr_copy_0[] = $val;
		}
		else
		{
			$arr_copy_1[] = $val;
		}
	}
}

// 对这4个数组进行对应的异或操作，就出结果了
function get_num()
{
	global $arr_0, $arr_1, $arr_copy_0, $arr_copy_1, $num1, $num2;
	$arr_0_xor = array_reduce($arr_0, 'do_xor');
	$arr_copy_0_xor = array_reduce($arr_copy_0, 'do_xor');
	$cacl_num1 = $arr_0_xor^$arr_copy_0_xor;

	$arr_1_xor = array_reduce($arr_1, 'do_xor');
	$arr_copy_1_xor = array_reduce($arr_copy_1, 'do_xor');
	$cacl_num2 = $arr_1_xor^$arr_copy_1_xor;

	echo $cacl_num1.' / '.$cacl_num2. PHP_EOL;
	echo $num1.' / '.$num2;
}

init();
```
```
二分查找 θ(logn)

递归方法
int binarySearch1(int a[] , int low , int high , int findNum)
{    
      int mid = ( low + high ) / 2;       
      if (low > high)        
            return -1;   
     else   
     {        
              if (a[mid] > findNum)          
                    return binarySearch1(a, low, mid - 1, findNum);        
              else if (a[mid] < findNum)            
                    return binarySearch1(a, mid + 1, high, findNum);                    
              else            
                    return mid;   
    }
}

非递归方法
int binarySearch2(int a[] , int low , int high , int findNum)
{    
       while (low <= high)
      {
            int mid = ( low + high) / 2;   //此处一定要放在while里面
            if (a[mid] < findNum)           
                low = mid + 1;        
            else if (a[mid] > findNum)            
                high = mid - 1;       
             else           
                return mid;    
    }       
    return  -1;
}
```

```
冒泡排序   θ(n^2)
void bubble_sort(int a[], int n)
{
    int i, j, temp;
    for (j = 0; j < n - 1; j++)
        for (i = 0; i < n - 1 - j; i++) //外层循环每循环一次就能确定出一个泡泡（最大或者最小），所以内层循环不用再计算已经排好的部分
        {
            if(a[i] > a[i + 1])
            {
                temp = a[i];
                a[i] = a[i + 1];
                a[i + 1] = temp;
            }
        }
}

快速排序  调用方法  quickSort(a,0,n);  θ(nlogn)
void quickSort (int a[] , int low , int high)
{
    if (high < low + 2)
        return;
    int start = low;
    int end = high;
    int temp;

    while (start < end)
    {
        while ( ++start < high && a[start] <= a[low]);//找到第一个比a[low]数值大的位子start

        while ( --end  > low  && a[end]  >= a[low]);//找到第一个比a[low]数值小的位子end

        //进行到此，a[end] < a[low] < a[start],但是物理位置上还是low < start < end，因此接下来交换a[start]和a[end],于是[low,start]这个区间里面全部比a[low]小的，[end,hight]这个区间里面全部都是比a[low]大的

        if (start < end)
        {
            temp = a[start];
            a[start]=a[end];
            a[end]=temp;
        }
        //在GCC编译器下，该写法无法达到交换的目的，a[start] ^= a[end] ^= a[start] ^= a[end];编译器的问题
    }
    //进行到此，[low,end]区间里面的数都比a[low]小的,[end,higt]区间里面都是比a[low]大的，把a[low]放到中间即可

    //在GCC编译器下，该写法无法达到交换的目的，a[low] ^= a[end] ^= a[low] ^= a[end];编译器的问题

    temp = a[low];
    a[low]=a[end];
    a[end]=temp;

    //现在就分成了3段了，由最初的a[low]枢纽分开的
    quickSort(a, low, end);
    quickSort(a, start, high);
}
```

[面试时该死的排序算法](http://coderyi.com/posts/sort_algorithm/)
# 阿里一面

1. MVC 具有什么样的优势，各个模块之间怎么通信，比如点击 Button 后 怎么通知 Model？
2. 两个无限长度链表（也就是可能有环） 判断有没有交点
3. UITableView 的相关优化
4. KVO、Notification、delegate 各自的优缺点，效率还有使用场景
5. 如何手动通知 KVO
6. Objective-C 中的 copy 方法
7. runtime 中，SEL 和 IMP 的区别
8. autoreleasepool 的使用场景和原理
9. RunLoop 的实现原理和数据结构，什么时候会用到
10. block 为什么会有循环引用
11. 使用 GCD 如何实现这个需求：A、B、C 三个任务并发，完成后执行任务 D。
12. NSOperation 和 GCD 的区别
13. CoreData 的使用，如何处理多线程问题
14. 如何设计图片缓存？
15. 有没有自己设计过网络控件？
## 阿里二面：

1. 怎么判断某个 cell 是否显示在屏幕上
2. 进程和线程的区别
3. TCP 与 UDP 区别
4. TCP 流量控制
5. 数组和链表的区别
6. UIView 生命周期
7. 如果页面 A 跳转到 页面 B，A 的 viewDidDisappear 方法和 B 的 viewDidAppear 方法哪个先调用？
8. block 循环引用问题
9. ARC 的本质
10. RunLoop 的基本概念，它是怎么休眠的？
11. Autoreleasepool 什么时候释放，在什么场景下使用？
12. 如何找到字符串中第一个不重复的字符
13. 哈希表如何处理冲突

## 数据结构
- 数组，链表，哈希表，二叉树的区别？数组索引和查找方便。链表插入和删除方便，
链表一般运用在堆栈（后进先出）和队列中（先进先出），哈希表方便查找，插入和删除。二叉树方便查找和排序
- 链表的插入是O(1)还是O(n)？是O(1)
- 写个反转二叉树的代码？递归左右子树交换
- 求二叉树相距最远的两个叶子节点？

## 基础算法题
- 如何以最快时间找到与给定点最近的点算法
- 写个 aabbbccaabddeffcc 化为abcdef
- 0(1)时间求栈中最大元素的算法
- 什么是贪婪算法
- 背包容量150，7个物品，每个物品重量价值不同，要求装入包中物品价值最大。
- n个人预约网球场，时间不同，求最少需要多少个网球场。
- 亿级数据里查找相同的字符以及出现次数
- 设计一种算法求出算法复杂度
- 两个字符串的最大公共子串
