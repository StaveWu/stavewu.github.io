---
title: leetcode刷题笔记
date: 2024/02/05 23:00:00
cover: /images/covers/03-computer.jpg
thumbnail: /images/covers/03-computer.jpg
toc: true
categories: 
 - 力扣
---

本文旨在记录刷题过程中的思考，以及在给定刷题路径上做一些变式补充。

刷题路径遵循：<https://github.com/youngyangyang04/leetcode-master?tab=readme-ov-file>

<!-- more -->

## 数组

### 题704：二分查找

二分查找重点关注边界取舍，这里讨论left、right为闭区间的情况：

*   外层循环为left <= right，具有等号
*   里层赋值left = mid + 1 或 right = mid - 1，偏移1个单位

为什么？

外层循环取等号由内层赋值控制推导，存在加1减1时left == right的可能，且所指向的值未曾遍历。

里层赋值偏移1个单位原因是，前一次的\[mid]值判断已经在条件中得到结论，故下一个\[mid]无需再遍历。

> \[数值]符号含义：指取该数值所指向的值。

**变式题34：找边界**

找出指定数值的起始位置和终止位置。仍然是二分查找，唯一要处理的点是：

在匹配到指定数值后，不要停：

*   起始位置对应：右边界 - 1
*   终止位置对应：左边界 + 1
*   暂存当前的位置

最终循环会因为left <= right不满足打破。

**变式题162：找峰值**

题目要求只要找到任意一个峰值即可，为此可以使用二分。

假设-1，n均为负无穷，所以原始数组就变为\[-infinity, 1, 2, 1, -infinity]

基于此取查找就可找到答案。

因为数组不支持-1和n索引，所以可以定义一个匿名函数支持它，这里也利用了cpp中的pair\<int,int>特性。

```cpp
auto get = [&](int i) -> pair<int, int> {
	if (i == -1 || i == n) {  // 如果是-1或n，pair第一个值比较起作用，一定是最小的
        return {0, 0};
    }
    return {1, nums[i]};  // pair的第二值起作用，依据具体nums[i]
};

get(idx) > get(idx + 1) && get(idx) > get(idx - 1)  // 这就是峰值
```

### 题27：O1空间内处理数组删除

方式1：快慢指针，快指针负责搜索不相等元素，慢指针负责等候快指针给的值进行存储。

方式2：c++可以使用iterator，在`*it`匹配到相等元素后，通过erase删除v.erase(it)，并保证it不向下走，因为这里erase的实现上会讲数组后续元素都向前shift 1个单位，所以此时的it仍然指向下一个新值。

当然，方式2比方式1慢很多（1ms vs 5ms），时间复杂度后者为O(n2)。

### 题977：On时间内平方数组

双指针解法，利用原有数组的有序特性 + 平方后的排序是两端大中间小的特性，推导出两个指针从两头开始，往中间遍历。

### 题209：最小连续子数组

使用滑动窗口，本质是双指针，维护一个双指针区间内部的sum，一边加一边卸，返回区间长度。

**变式题121：股票价值最大化**

思路：题目要求只能1次买入卖出，所以可以设想：每一天，都想一个问题，我在历史最低点买入，今天卖出，会盈利多少（也就是保存最小价格，和当天价格相减，取max，得到结果）

**变式题122：支持多次买入卖出**

思路：动态规划，二维数组做状态记录

> 其实只用2个数值做记录也够用，因为我们总是只回看前一个数。

一列是未持股时手上的现金，一列是持股后手上的现金

下一列的值等于：

*   今日未持股现金 = 昨日未持股现金，昨日持股后今日卖掉的现金 取最大&#x20;
*   今日持股现金 = 昨日持股现金，昨日未持股现金买今日股票后的现金 取最大

最后取未持股时的现金返回。

### 题59：螺旋矩阵

螺旋矩阵没什么算法，纯粹是边界处理，顺着方向写就行。

right -> down -> left -> up

每转一圈，边长减1。

## 链表

### 题203：移除链表元素

两个基本注意点：

*   要创建一个dummy节点，作为起始遍历辅助
*   结束时要返回dummy.next，而不是head，因为head有可能已被删除

### 题707：设计链表

核心方法：addAtIndex，deleteAtIndex，get

关键点是要初始化一个dummy节点，从该节点开始处理链表的新增和删除。其本质和上述移除链表元素是一样的，必须得有个假的头节点来处理index为0的情况。

### 题206：翻转链表

观察翻转前后的变化，只有相邻节点的next箭头变了，因此只需要prev、current两个变量：

*   先暂存next，翻转current->prev
*   然后prev、current往下平移1个单位

最后prev就是头了。

当然这道题的另一种思路是拷贝到新链表，弊端是占用了额外内存空间。

### 题24：两两节点交换

![](https://assets.leetcode.com/uploads/2020/10/03/swap_ex1.jpg)

最小可重复交换操作为：

```bash
# 三个数一组
[dummy 1 2] 3 4
[dummy 2 1] 3 4
# 指针偏移2个单位
dummy 2 [1 3 4]
dummy 2 [1 4 3]
# 再偏移2个单位
dummy 2 1 4 [3 null null]  # 无需交换

# 不满两组的情况
[dummy 1 2] 3
[dummy 2 1] 3
dummy 2 [1 3 null]  # 无需交换
```

所以截止条件为：只要有null，就无需再操作。

这种最小可重复操作怎么发现的？

观察一次有多少元素需要参与运算，上述1 2参与了，2的下一个元素3属于下一次运算的范畴，可以不处理。下一次运算时，3 4交换，此时前面的1箭头也要变，所以参与运算的就有3个。为了抽象为通用操作，第一次运算1 2时，在其前补充dummy元素。

### 题19：删除链表的倒数第N个节点

要求一次遍历计算出来：

*   链表长度怎么获取？

    *   只能遍历完。
*   遍历完后推算出倒数第n的位置，怎么返回获取前后节点？

    *   遍历时有意识的保存

如何实现有意识的保存？思路：构造2个有距离的指针，同时shift遍历，距离就是n。

### 题02.07：链表相交

思路1：普通解法

*   分别遍历每条链表，计算长度，得到长度之差
*   挑长链表磨平差值
*   此时两条链表并排，开始遍历比较，第一个出现相等的元素就是了

思路2：数学算法

链表A不相交部分长度a，链表A相交部分长度c

链表A不相交部分长度a，链表A相交部分长度c

```bash
a + c = m
b + c = n
a + c + b = b + c + a
```

可以得到，不相交部分长度走完，一定是交点。

### 题142：环形链表

思路1：将所有被遍历的节点压栈set集合，如果有重复节点，set长度将不会增长。

*   缺点：需要额外内存空间存储节点地址

思路2：设置快慢指针，快指针1次2步，慢指针1次1步，如果快指针追上慢指针，则满足条件。

一定能相遇吗？

*   一定能的，即便快指针在固定节点循环，慢指针属于遍历，一定能踩到。

相遇的点一定是交点吗？

*   不是，但可以通过以下公式推导：假设a是环外长度，b是环内长度，n表示在环内转了n圈，f是快指针走的里程，s是慢指针走的里程

```bash
# 已知快指针比慢指针多走一倍里程
f = 2s
# 已知快慢相遇时，在相遇节点开始走n圈 + s就是快指针的里程
f = s + nb
# 推导得到
s = nb
# 已知从head走到入口节点需要：
a + nb
# 慢指针已经走了nb步，还需再走a步，a步怎么来？和head开始的指针一起走，直至相遇，得到结论。
```

## hash表

### 题242：有效的字母异位词

思路1：最直接的方法是定义一个hashmap，第一轮迭代填充charMap，第二轮迭代减少charMap计数，如果出现-1的则return false。

思路2：定义1个具有26长度的数组，每个位置依据字符串的字符位置进行++--，最后check一遍得到结果。

### 题1002：查找共用字符

这里有个坑是，不是说一个字母统计一次就行，而是如相同字母在多个words里分别重复出现了，那就都得输出出来。

思路：一个当前word的freq数组，一个全局words的最小freq数组，分别统计每个词的字母出现频率。

全局word在的foreach每个词时，总是取当前word每个槽位的最小值。

### 题349：两个数组的交集

这道题跟上面一道题有点类似，这道题要求一个字母只统计1次，并且只有两个数组之间的比较，所以可以认为是上一道题的简单版本。

思路：set存储第一个数组内容，set.count(第二个数组中的每一个元素)，如果存在，则return

### 题202：快乐数

首先理解题意：

*   拆解的数的每一位做平方
*   希望得到1
*   可能永远都得不到
*   得不到的原因是无限循环（在某个圈内循环转）

解题关键：找到循环的入口

具体思路：记录历史算数，如果重复或为1，则退出

### 题1：两数之和

思路：排序，二分，要寻找的对象是：find(和 - 当前数值)。

寻找对象的方法还可以是hashset，即hashset + find，这里要注意，find到值不能是自己，并且还要有数组下标返回，所以通过hashmap更合适。

### 题454：四数相加

四个数组，长度一致；每个数组挑1个数，和其他数相加；求和要等于0；返回有多少种可能。

要把所有组合都遍历一遍，n的4次方。减少复杂度方法是，分为2组，每组n的2次方，从等于0改为ab组 = -cd组。ab组的求和存入hash表，值为等于该和的组数；cd组查询，这样就得到最终结果。

### 题383：赎金信

数组2中的字符作为候选，数组1消费。

给1个hashmap对候选计数，数组1遍历时消费，如果找不到或value出现小于0的情况，则为false。

### 题15：三数之和

思路1：选target，剩下的元素求“两数之和”，因为本题要求返回所有元组，而两数之和给了1个只有1个元组答案的约束，所以，本题最后还得有一层去重处理。

思路2：组合问题，从数组中选3，求和。

组合问题是无序的，相比有序的数组还是差了点效率。所以

思路3：这里可以基于排序后的数组，使用双指针解决

*   遍历数组，从first = 0，选target
*   定义left = first + 1、right == n - 1，双指针两方向往中间靠近，找所有解

second和third就是双指针。

### 题18：四数之和

本质是在三数之和上套一层循环，怎么理解？

*   遍历数组，从first = 0，选target
*   遍历数组，从second = first + 1
*   定义left = second + 1、right = n - 1，双指针两方向往中间靠近，找所有解

相关剪枝细节（其实就是哪些可以跳过不做的）这里不展开。

## 字符串

### 题344：反转字符串

这道题要求原地反转。

思路：双指针从两端到中间依次反转字符对，截止条件：指针相遇或错过。

**变式题541：反转字符串2**

相比上一道字符串增加了步长，其余逻辑一致。

### 题kama54：替换数字

这道题也是要做到原地变更字符串。利用C++里的s.resize方法。

核心思路是：预留卡位，从后往前填。

### 题151：反转字符串中的单词

这题考察对string api的熟悉程度。

更高要求：原地修改字符串，大致步骤：

*   字符串reverse
*   删除多余空格
*   单词reverse

### 题kama55：右旋字符串

思路上跟上道题类似，右旋只是表象，本质还是2个操作：

*   字符串reverse
*   区间字符串reverse（等同于单词reverse）

### 题459：重复的子字符串

特殊例子：ababdababd、acdfdacdfd，这也是重复的

思路1：比较朴素的解法

*   遍历各种长度（满足size % 长度 == 0）的子串
*   始终满足是前一个子串的前缀：s\[j] == s\[j - 1]，j++，j < size

思路2：特殊性质

*   如果是重复子串，则abab + abab，去掉头尾， = bababa 包含ab
*   如果不是，则absab + absab = bsababsa 不包含 absab

## 双指针

### 题27：移除元素

原地操作移除元素，可使用快慢指针，慢指针指向当前要赋值的位置，快指针指向非目标元素的位置，如此迭代。

## 栈与队列

### 题232：用栈实现队列

队列FIFO，栈LIFO，用栈实现队列核心要解决如何将栈头元素弹出问题。

让第一次进栈的元素保持在栈尾？双栈思路。

在pop/top时操作，将当前所有元素都转移到新栈上。

注意这里有个tracky，新栈总是在为空时才从旧栈中补充元素。

### 题225：用队列实现栈

push时操作，将新push进队列的元素置于front位置，可通过循环将其他元素pop再push的方式。

### 题20：有效的括号

校验括号的方法：左括号进栈，右括号遇到匹配的左括号，则左括号弹栈，否则右括号进栈。

### 题1047：删除字符串中的所以相邻重复项

和上述题一样，相同的消掉，不同的入栈，最后反转下栈内元素输出即可。

### 题150：逆波兰表达式求值

思路：如果时数字，压栈，如果是符号，弹出两个数字，计算，再压栈。

### 题239：滑动窗口最大值

看着简单，操作起来有点难度。

思路：优先队列，它能解决区间最大值的问题。但还要考虑：

*   如何删除区间的第一个值问题

这里就有一个位置信息需要在优先队列里保留，方法：

*   设置队列节点为\<nums\[i], \[i]>的pair对

那么必须要始终确保优先队列的长度为窗口大小吗？不用。其实只要确保当前队列里的top仍然在窗口内，那么top就不需要摘除，保留即可，其他比top小的窗口外元素不影响。

### 题347：前k个高频元素

前k个 --> 优先队列，返回高频元素，所以队列元素设计为<频率,元素>pair，为了得到这种pair，还需要使用map收集。

这块还需要重写优先队列的排序方式，只用频率排序。

## 二叉树

### 题144/145/94：二叉树遍历

前序：中左右

中序：左中右

后序：左右中

可以看出，前后是基于树的高来讲，root在树顶点，那么它就是前，树的叶子节点就是后。

### 题102：层序遍历

从上到下输出，前序

遍历是深度的，所以还需要辅助信息，也就是层高

层高对应到数组索引

所以思路1：深度优先搜索

```cpp
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector<vector<int>> res;
        traversal(root, 0, res);
        return res;
    }

    void traversal(TreeNode* node, int level, vector<vector<int>>& res) {
        if (node == nullptr) {
            return;
        }
        while (res.size() <= level) {
            vector<int> ele;
            res.push_back(ele);
        }
        res[level].push_back(node->val);  // 深度遍历到对应层级时加入对应的数组内
        traversal(node->left, level + 1, res);
        traversal(node->right, level + 1, res);
    }
};
```

思路2：广度优先搜索的层序版

通过辅助的队列来实现。队列的职责是保存每一层的节点：

*   检查队列是否为空
*   **全部遍历**，依次弹出，取这些节点的左右节点重新入队列
*   如此循环

```cpp
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector<vector<int>> res;
        if (root == nullptr) {
            return res;
        }
        queue<TreeNode*> q;
        q.push(root);
        while (!q.empty()) {
            int levelsize = q.size();
            res.push_back(vector<int>());
            for (int i = 0; i < levelsize; i++) {  // 广度优先搜索不需要这一层
                TreeNode* cur = q.front();
                q.pop();
                res.back().push_back(cur->val);
                if (cur->left != nullptr) {
                    q.push(cur->left);
                }
                if (cur->right != nullptr) {
                    q.push(cur->right);
                }
            }
        }
        return res;
    }
};
```

> 这里要注意一个细节，即层序遍历不是广度优先搜索。广度优先是1个节点1个节点逐个往下走，它没有层的信息，而层序遍历是将层的信息显性化出来。

**变式题107：要求返回从底向上的层序遍历**

思路：基于常规层序遍历将res翻转下。

有其他思路吗？没有，官方题解本质仍然是翻转，只是在while循环中翻转和循环完翻转的问题。

**变式题199：二叉树的右视图**

思路1：层序，保留每一层的最优节点值。

思路2：层序或dfs，在最后res中取back值。

**变式题637：二叉树的层平均值**

同样的做法，层序遍历时求平均。

**变式题429：N叉树的层序遍历**

只需要改造left、right为for children即可。

**变式题515：二叉树的层最大值**

同样的做法，层序遍历求最大。

**变式题116：填充每个节点的下一个右侧节点**

层序，在遍历时做好前后节点的next关联。

**变式题104：二叉树的最大深度**

思路1：层序，为层计数。

思路2：深度DFS，可以不用辅助函数。

**变式题559：N叉树的最大深度**

思路类似，深度遍历，每下钻一层加1。

**变式题111：二叉树的最小深度**

思路：层序，当left、right都为空时直接返回。

### 题226：翻转二叉树

思路：深度遍历，无需辅助函数，直接翻转（这里可以选择前序，也可以选择后序）。

### 题101：对称二叉树

思路：层序遍历，弹出单层数组，判断是否对称。这里有个处理技巧，就是单层元素不按从左到右加入队列，而是两端对称加入；每次循环弹出2个节点进行比较。

### 题222：完全二叉树的节点个数

参考二叉树的最大深度解法，通过DFS来解。

因为二叉树的性质，这里还有一种二分查找的解法，先算出有多少层，然后再二分叶子节点。

### 题110：平衡二叉树

平衡的性质是左右子树的高相差小于等于1

*   首先要能算出高
*   其次是判断是否满足小于等于1的条件

### 题257：二叉树的所有路径

深度优先搜索，本质也是使用回溯的方法，设计好输入、截止条件即可。

### 题513：找树左下角的值

左下角，高度最大，最左

深度遍历需要带上这两个状态，同时要带上answer（当前高度，当前值）

如果新遍历的比当前高度高，则刷新answer。

处理细节：是否为左节点的状态可以去掉，转为由height > answer.curHeight代替，原因：height递增时的首次遍历的节点一定是左节点（通过由先遍历左子树再遍历右子树的方式控制）。

### 题112：路径之和

求路径和，判断是否等于目标值。

路径和 = DFS + sum累加，遇到叶子节点后截止。

### 题106：已知中后序，构造二叉树

特征：后序的最后1个元素总是根节点。

步骤：

*   后序数组倒序遍历，选取的元素拿到中序遍历中做边界，分出左右子树数组
*   递归构造左子树数组
*   递归构造右子树数组
*   左右子树拼接到当前的根节点

因为涉及元素到index的查询，所以，构造1个（元素，index）的map加快索引。

### 题654：最大二叉树

这道题跟题106类似，递归，选出根节点，构造左右子树，拼接到根节点即可。

### 题617：合并二叉树

合并二叉树的逻辑也是构造左右子树，拼接到根节点。

### 题700：二叉搜索树

深度遍历即可，小于往左，大于往右。

### 题98：验证二叉搜索树

需满足两条规则：

*   当前节点比左节点大，比右节点小；
*   当前节点要小于或大于整体子树的区间范围，比如\[5,4,7,null,null,3,9]，3比4小，false

> 搜索树有个特征：不存在两个节点的值重复。

### 题530：二叉搜索树的最小差值

注意题目要求是任意两节点，非相邻节点。但实际根据树的性质，最小值只会存在于**中序遍历**下相邻的两个节点（因为中序遍历将得到1个递增序列）。

中序遍历需具有以下状态：前一个节点的值，当前的最小值。

### 题501：二叉搜索树的众数

这道题的二叉树定义为节点值可重复。

仍然按中序遍历，因为是递增序列，所以在遍历时保存前一个节点值preval，该值的count，当前的最大count，以及当前的最大countVals（因为要返回具体的val值）

官方解答对首次处理中序有一个比较晦涩的点：

```cpp
int preval, int curCount, int maxCount

dfs // 前序
// 中序，注意看这里省略了对preval未初始化情况的处理
if (node->val == preval) {
  curCount++; 
} else {
  curCount = 1;
  preval = node->val;
}
```

更直观的写法：

```cpp
int preNode, int curCount, int maxCount
dfs // 前序
// 中序
if (preNode == nullptr) {
  curCount = 1;
} else if (preNode->val == node->val) {
  curCount++;
} else {
  curCount = 1;
}
preNode = node;
```

可以看到，直观的写法中，if elseif块内的处理是一致的，故官方解法直接将二者合并为1个，无论preval是什么。

### 题236：二叉树的最近公共祖先

最近公共祖先的特征：

*   该祖先的下一个node->left和node->right必有一方不是祖先

所以在搜索时，可以先探查左右子树，如果同时为空，则说明该节点不包含目标；如果同时不为空，则说明该节点就是二者的公共祖先；如果一方为空、一方不为空，为空的子树没有包含目标，往不为空的搜索。

```cpp
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if (root == nullptr || root == p || root == q) return root;

        TreeNode* left = lowestCommonAncestor(root->left, p, q);
        TreeNode* right = lowestCommonAncestor(root->right, p, q);

        if (left == nullptr && right == nullptr) return nullptr;
        if (left != nullptr && right != nullptr) return root;
        if (left == nullptr) return right;
        if (right == nullptr) return left;
        return root;
    }
};
```

### 题701：二叉树的插入操作

思路：找到节点要插入的位置，挂上去，这种适合前序遍历。

首先判断节点是否为null，是则构造一个当前val的节点返回；否则对left、right进行遍历，并对返回值重新挂载到root。

### 题450：二叉树的删除操作

类似插入操作，同样是找到节点，然后执行删除：

*   如果左右子树均为空，返回空
*   如果左子树为空，返回右子树
*   如果右子树为空，返回左子树
*   如果都不为空，则找到右子树的最左节点，把当前左子树挂到这个节点上，然后返回右子树

### 题669：修剪二叉搜索树

区间描述要保留的节点，中序遍历，一路trim节点，trim规则：

*   小于左边界的，trim左子树
*   大于右边界的，trim右子树
*   满足区间内的，左子树的trim结果挂到左节点，右子树的trim结果挂到右节点（整体结构上只会有边界节点会变化）

```cpp
class Solution {
public:
    TreeNode* trimBST(TreeNode* root, int low, int high) {
        if (root == nullptr) {
            return root;
        }
        if (root->val < low) {
            return trimBST(root->right, low, high);
        }
        if (root->val > high) {
            return trimBST(root->left, low, high);
        }
        root->left = trimBST(root->left, low, high);
        root->right = trimBST(root->right, low, high);
        return root;
    }
};
```

### 题108：有序数组转平衡二叉树

二分查找，每个被发现的中间元素作为root节点，而二分本身能够确保树平衡。

### 题538：二叉树转换为累加树

二叉树转为递减数组，每次遍历累加前一个节点的值

如果实现递减遍历？通过颠倒的中序遍历来做。

遍历过程需要维护一个[currentSum]()值。

## 回溯

### 题77：组合

这道题有两种写法，第一种是带有for循环的：

```cpp
class Solution {
public:
    vector<vector<int>> combine(int n, int k) {
        vector<int> curVec;
        vector<vector<int>> res;
        backtracking(n, k, 1, curVec, res);
        return res;
    }

    void backtracking(int n, int k, int start, vector<int>& curVec, vector<vector<int>>& res) {
        if (curVec.size() == k) {
            res.emplace_back(curVec);
            return;
        }
        for (int i = start; i <= n; i++) {
            curVec.push_back(i);
            backtracking(n, k, i + 1, curVec, res);
            curVec.pop_back();
        }
    }
};
```

第二种是不带for循环的：

```cpp
class Solution {
public:
    vector<vector<int>> combine(int n, int k) {
        vector<int> curVec;
        vector<vector<int>> res;
        backtracking(n, k, 1, curVec, res);
        return res;
    }

    void backtracking(int n, int k, int cur, vector<int>& curVec, vector<vector<int>>& res) {
        if (curVec.size() + n - cur + 1 < k) {  // 这里做了剪枝处理，表示如果当前数组长度+剩余待遍历区间长度小于k时，跳过不遍历，因为没必要
            return;
        }
        if (curVec.size() == k) {
            res.emplace_back(curVec);
            return;
        }
        if (cur == n + 1) {
            return;
        }
        curVec.push_back(cur);
        backtracking(n, k, cur + 1, curVec, res);
        curVec.pop_back();
        backtracking(n, k, cur + 1, curVec, res);
    }
};
```

以上存在一个细节，在回收结果处，使用了emplace\_back来copy数组

其他方案有：

*   入参不带&，而使用copy：backtracking(int n, int k, int cur, vector\<int> curVec, vector\<vector\<int>>& res)
*   入参带&，在回收结果时，自行构造一个新vector拷贝curVec

实验证明，上述两种都不高效，且第一种方案内存使用异常高，是最优方案的10倍。

**变式题216：组合总和3**

这道题同样求组合，加了1个约束，要求组合的总和要等于指定值。

思路：遍历所有组合，维护当前总和值，如果等于指定值且组合的size等于指定size，则返回，否则回溯。

剪枝细节：同上道题，vec长度+区间长度 < 指定长度或者 vec长度大于指定长度的，无需遍历，直接返回。

**变式题39：组合总和**

这道题的约束为，不限制给定数组中的某个数被选择几次，找出所有能够求和等于target值的组合。

类似题目：爬楼梯，可选的一步台阶范围类比这里给定的数组。

解法（更适合不具有for循环的递归）

```cpp
    void backtracking(vector<int>& candidates, int index, int target, vector<int>& curVec, vector<vector<int>>& res) {
        if (index == candidates.size()) {
            return;
        }
        if (target == 0) {
            res.emplace_back(curVec);
            return;
        }

        // 跳过不使用candidates[index]
        backtracking(candidates, index + 1, target, curVec, res);

        // 使用candicates[index]
        if (target - candidates[index] >= 0) {
            curVec.emplace_back(candidates[index]);
            backtracking(candidates, index, target - candidates[index], curVec, res);
            curVec.pop_back();
        }
    }
```

**变式题40：组合总和2**

这道题放宽了对给定数组的约束，允许数组内有元素重复，并且，结果要求不可重复，且每个元素只能被用一次。

对于只能被用一次这条，正常组合解法已覆盖。

对于结果要求不可重复这条，需要做两个步骤：

*   对给定数组排序
*   带for循环递归时，检查前后两个元素是否相同，是则跳过

剪枝细节：如果target - 当前元素小于0，可以直接break，无需继续遍历。

### 题17：电话号码的字母组合

这道题求的是多组字母内分别取1的组合，之前的题型则为1组数内取k的组合。

多组取1，对上述1组取多进行变式：

```cpp
    void backtracking(vector<string>& digitLetters /* 多组取1 */, int index /* 指向哪一组 */, string& curStr, vector<string>& res) {
        if (curStr.size() == digitLetters.size()) {
            res.emplace_back(curStr);
            return;
        }
        for (int i = 0; i < digitLetters[index].size(); i++) {  // 遍历每一组内的所有字母
            curStr.push_back(digitLetters[index][i]);
            backtracking(digitLetters, index + 1, curStr, res);
            curStr.pop_back();
        }
    }
```

### 题131：回文字符串

遍历字符串，选择步长，可以是1，2，3等，检查该子串是否符合，如果符合，加入结果；如果不符合，继续往大步长遍历。

每次处理完当前子串后，对剩余的子串做相同动作（回溯）

将这个过程具象化为一张横纵回溯图，如下所示：

![image](https://camo.githubusercontent.com/0e9d7097c5f8a06d6ee1a6fd0ab110e981442669d0dbcdeb6d5e25ef615d5f1a/68747470733a2f2f636f64652d7468696e6b696e672e63646e2e626365626f732e636f6d2f706963732f3133312e2545352538382538362545352538392542322545352539422539452545362539362538372545342542382542322e6a7067)

横向对当前子串的区间放大，并分别判断

纵向对剩余子串做同等处理

### 题93：复原IP地址

和回文串类似，对当前子串长度进行遍历，对剩余子串进行回溯。

这里唯一要注意下对子串的合法性进行校验。

### 题78：子集

和爬楼梯类似，无需for循环，分考虑当前位置、不考虑当前位置两种场景进行回溯。

```cpp
class Solution {
public:
    vector<vector<int>> subsets(vector<int>& nums) {
        vector<int> subset;
        vector<vector<int>> res;
        backtracking(nums, 0, subset, res);
        return res;
    }

    void backtracking(vector<int>& nums, int start, vector<int>& subset, vector<vector<int>>& subsets) {
        if (start >= nums.size()) {
            subsets.emplace_back(subset);
            return;
        }
        subset.push_back(nums[start]);
        backtracking(nums, start + 1, subset, subsets);
        subset.pop_back();
        backtracking(nums, start + 1, subset, subsets);
    }
};
```

思考：什么时候回溯需要for循环，什么时候不用？

其实这道题也可以套for循环：

```cpp
class Solution {
public:
    vector<vector<int>> subsets(vector<int>& nums) {
        vector<int> subset;
        vector<vector<int>> res;
        backtracking(nums, 0, subset, res);
        return res;
    }

    void backtracking(vector<int>& nums, int startIndex, vector<int>& subset, vector<vector<int>>& subsets) {
        subsets.push_back(subset); // 收集子集，要放在终止添加的上面，否则会漏掉自己
        if (startIndex >= nums.size()) { // 终止条件可以不加
            return;
        }
        for (int i = startIndex; i < nums.size(); i++) {
            subset.push_back(nums[i]);
            backtracking(nums, i + 1, subset, subsets);
            subset.pop_back();
        }
    }
};
```

相比第一种没有for循环写法，具有for循环的写法明显不那么直观。

**变式题90：子集2**

相比上道题，这道题给定的数组元素重复，求不重复的子集。

解法和**组合求和2**一致，先排序，再回溯，检测前后重复的跳过。

### 题491：非递减子序列

给定数组，带重复元素，找所有非递减子序列。

这道题特殊之处在于需要给子序列回溯加限定条件：

*   当前元素比序列中最后1个元素大的，不考虑
*   当前元素已经在之前用过的，不考虑（使用set数据结构，保存本层当前元素）

比如\[4,7,6,7]：4回溯，7回溯，6跳过，7跳过

### 题46：全排列

push1 push2 push3 ==> 1,2,3

push1 push2 pop2 push3 ==> 1,3,2

pop掉的元素需要有地方暂存，直接存回数组？—— 可以用1个bool数组标记是否被使用过。

因为全排列不分前后顺序，所以在每层回溯时都要遍历全部nums数组元素。

**变式题47：全排列2**

给定的nums数组现在具有重复元素了，求全排列。

重复元素的做法：排序，for循环遇到重复元素，跳过

这里仅上述判断还不够，因为类似\[1,2,2]的答案，2和2重复，但不能跳。

所以还得再加1个原则：nums\[i - 1]被用过，即used\[i - 1] == false。

## 贪心

### 题455：分发饼干

约束：每个人最多获得1块饼干。

贪心策略：人按胃口排序，饼干按大小排序，遍历每个人，找能满足当前这个人胃口的最小饼干。

### 题376：摆动序列

删除若干元素后形成摆动序列，求序列最长的长度：可以转化为统计峰谷的数量。

> 这里有个思维误区是：求要删除多少个元素，然后用序列总长 - 要删除的元素，该思路本身没问题，但对前后相等的过渡元素场景不好处理。

### 题53：最大子数组和

贪心策略：如果当前数组的sum为负数，则直接丢弃当前数组的sum（使sum置为0），从下一个新的元素开始计数。

### 题122：买卖股票的最佳时机2

贪心策略：求多个不相交的区间，区间的利润不存在负收益，然后对这些区间求和，得到最大利润。

因为多个不相交无负收益区间之和等价于多个间隔为1的无负收益区间之和，所以问题可以简化相邻元素相减。

### 题55：跳跃游戏

贪心策略：每一个元素都遍历一遍，检查该元素能够跳到的位置，不断统计取最远，如果遍历时发现的当前i指针超过了最远的位置，那就说明后面的元素不可达，return false。

### 题45：跳跃游戏2

这道题保证了一定能跳到终点，求最小步数。

仍然是每个元素都遍历一遍，检查该元素能够跳到的位置，不断统计取最远。不同的是，在这里面，需要维护一个每次跳的区间。

比如首次区间就是：start = 0, end = 1

后面的区间则为：start = end, end = rightmost + 1

### 题1005：K次取反后的数组最大和

贪心策略：

*   值越小取反收益越大，比如：1 2选1，-2 -1选-2
*   都是非负数时，对当前最小值进行反复取反

简单点可以考虑用最小堆。

### 题134：加油站

能否到终点：对所有剩余油量求和，如果小于0，则无法到终点。

选起点：该起点应能到达下一站；该起点应剩余油量要足够大（反例：\[1,1,3], \[1,2,1]，0号加油站不能作为起点，2号可以 ==>\[0,-1,2]）

如果有1段特别大cost的呢？比如剩余油量：\[1,2,-3]，选1号加油站过不去，只能选0号加油站。

所以问题变为，剩余油量序列，累积子序列不能为负。

如果出现负数，怎么办？

思路1：从后往前寻找能填平该负数的index，如果有，那么该index就是起点。

```cpp
class Solution {
public:
    int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
        int endIndex = 0;
        int sumRest = 0;
        int minGasRest = INT_MAX;
        for (int i = 0; i < gas.size(); i++) {
            int rest = gas[i] - cost[i];
            sumRest += rest;
            if (sumRest < minGasRest) {
                minGasRest = sumRest;
                endIndex = i;
            }
        }
        if (sumRest < 0) {  // 不能到达终点
            return -1;
        }
        if (minGasRest >= 0) {  // 0作为起点就能到达终点
            return 0;
        }

        // 0作为起点不能达到终点，缺油，从后面补
        for (int i = gas.size() - 1; i >= endIndex + 1; i--) {  // 注意有些答案将endIndex + 1处理为0，原因是即便从后往前重复计算前半段，前半段已经明确缺油，前半段计算进去也找不到补
            int rest = gas[i] - cost[i];
            minGasRest += rest;
            if (minGasRest >= 0) {
                return i;
            }
        }
        return -1;
    }
};
```

思路2：借鉴最大子序列和的思路，出现负数时，就需要舍弃这部分，从下一个元素作为新起点。

```cpp
class Solution {
public:
    int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
        int curSum = 0;
        int totalSum = 0;
        int start = 0;
        for (int i = 0; i < gas.size(); i++) {
            int rest = gas[i] - cost[i];
            curSum += rest;
            totalSum += rest;
            if (curSum < 0) {
                start = i + 1;
                curSum = 0;
            }
        }
        if (totalSum < 0) return -1;  // 无解
        return start;  // 如果有解，一定从子序列之和为非负数开始
    }
};
```

### 题135：分发糖果

rating为1,2,3的，糖果为1,(1+1),(2+1)，后一个数=前一个数+1

rating为3,2,1的，糖果为(2+1),(1+1),1，前一个数=后一个数+1

推测：正方向遍历

如果遇到正向递减的，如1,2,3,2,1，是否要跟着减一？不需要，如果减一会丢失的重复元素的位置信息。故正向递减按规则得到1,2,3,1,1

按倒序规则得到1,1,3,2,1

两个数组结合取最大值，得到1,2,3,2,1，就是我们最后想要的。

重复元素怎么办？也适用

1,2,3,3,3,2,1

*   正向：1,2,3,1,1,1,1
*   反向：1,1,1,1,3,2,1
*   最大：1,2,3,1,3,2,1 ==> 是最终结果

### 题860：柠檬水找零

贪心思路：拿当前能够满足找零的最大面额去填。

### 题406：根据身高重建队列

这道题有点脑筋急转弯。关键还是要发现队列的规律：

*   先按身高排（从高到低），如果身高相等的，按前方高于自身的数量排（从小到大），此时的序列接近直觉
*   按上述顺序依次插入到新队列，身高高的总是先插入，身高矮的后插入，插入位置基于当前元素的前面高于自身的数量定，无论插入到哪都不会对当前新队列中已有的元素值造成影响，所以得到的新队列结果就是我们想要的。

### 题452：用最小数量的箭引爆气球

气球区间重叠的可用1支箭射爆，那么问题就变为寻找子集的数量。

我们对所有气球区间按左边界排序，然后维护1个子集和所有气球区间比对，有更小的子集就更新，最后返回子集数量即可。

### 题435：无重叠区间

两个步骤：

*   排序，按右边界大小排（优先参加结束时间早的），如果相等的，选区间长度短的
*   遍历，发现有交集就去掉当前区间，计数加1

另一个思路：

*   排序，按左边界大小排，如果相等的，选区间长度短的
*   遍历，发现有交集就去掉大的区间，留小区间，计数加1

### 题763：划分字母空间

这道题本质是求重叠子区间的题，但因为字母多，形成的子区间多，求子区间交集较为复杂，为此有一个巧妙的思路：通过遍历字符串，比较当前索引是否等于当前字母的右区间，是则为分割点。

### 题56：合并区间

也是求重叠子区间问题，常规思路，排序，依次遍历，发现有重合的就合并，否则加入结果集返回。

### 题738：单调递增的数字

找规律：从后往前，依次比较，如果出现不符合预期单调的，基于该规则改：比如32改为29，如此反复：4332，4329，4299，3299，2999。

### 题968：监控二叉树

贪心思路：让摄像头数量最少 等价于 让叶子节点尽量不要有摄像头，因为叶子节点放摄像头是非叶子节点的指数倍。

为实现这种思路，需要对节点的状态分级：

*   0：节点无覆盖
*   1：节点有摄像头
*   2：节点有覆盖

通过后序遍历，基于每个节点的情况递推该状态值，并按情况放摄像头和计数。

## 动态规划

### 题509：斐波那契数

f(2) = f(1) + f(0); f(3) = f(2) + f(1)；f(0)和f(1)已知，所以直接基于此for循环滚动相加到指定n值即可。

动态规划在这里面用到的位置为：每次for循环记录n-1 n-2处的值。

### 题70：爬楼梯

n阶楼梯到顶，就有n-1阶楼梯的dp\[n-1]种方法 + n-2阶楼梯的dp\[n - 2]种方法。

本质是斐波那契数。

所以解法和斐波那契数一致。

### 题746：使用最小花费爬楼梯

动态规划数组定义：dp\[i] = min(dp\[i-1] + cost\[i-1], dp\[i-2] + cost\[i-2])

解释：dp\[i]指当前第i阶所花费的最小费用。

### 题62：不同路径

机器人到达mxn矩阵右下角有多少不同路径，定义dp\[i]\[j]，表示到达i,j点有多少不同路径，其等于(i-1,j),(i,j-1)点的路径数之和。

**变式题63：不同路径2**

这道题在mxn矩阵中加了障碍物，整体思路是不变的，核心点在于遇到障碍物后，障碍物位置的dp\[i]\[j]要清零。

### 题343：整数拆分

一个整数i，可以拆分为：

*   两个整数，j \* (i - j)
*   一个整数和再拆分的整数相乘：j \* 再拆分(i - j)

定义dp\[i]为整数i的最大乘积

那么上述再拆分的乘积即为：j \* dp\[i - j]

那么整数i的最大乘积即可从上述两种情况中获取：

```bash
max(j * (i - j), j * dp[i - j])
```

j的取值可以从1到i-1遍历，计算每种拆分j的值，取最大，就得到当前dp\[i]最大：

```bash
curMax = 0;
for j = 1..i-1
	curMax = max(curMax, max(j * (i - j), j * dp[i - j]))
dp[i] = curMax
```

### 题96：不同的二叉搜索树

动态规划思路：

遍历1..n，计算每个数i作为根节点时的二叉树个数，然后求和。

定义dp\[i]，表示以i为根节点时的二叉树个数

针对每个根节点的取值j，都有dp\[j] = dp\[j-1] \* dp\[i-j]，得解。

### 题kama46：01背包

定义物品的weight和价值value，求一个给定bagweight容量的背包，可以放入的最大价值。

这种就是01背包问题，对于每个物品，要么选1个，要么不选

如果是每个物品可以选多次，那就是完全背包问题。

背包问题的dp数组定义：dp\[j]表示达到重量为j的背包最大可放入的价值。

递推公式：dp\[j] = max(dp\[j], dp\[j - weight\[i]] + value\[i])

解释：不选，dp\[j]，选，dp\[j - weight\[i]] + value\[i]

遍历方式：

```cpp
for (int i = 0; i < weight.size(); i++) {  // 遍历物品
	for (int j = bagweight; j >= weight[i]; j--) {  // 遍历重量
		dp[j] = max(dp[j], dp[j - weight[i]] + value[i]);
	}
}
```

### 题416：分割等和子集

将一个数组分成两部分，要求和相等

思路：求和，除以2，得到背包的大小target，数组本身既是weight也是value。满足条件：dp\[target] == target。

### 题1049：最后一块石头的重量2

题目核心问题在于任意石头，不是相邻石头，故需要动态规划。

思路：两快石头相消，得到石头A-石头B的重量，推导到整个数组就是两个子数组相消得到的最小重量。

如何得到最小重量？尽量让两个数组之和相等。这就回到上面的题416解法。

### 题494：目标和

回溯：选+，选-，直到pos遍历完， 检查是否等于target

动态规划：求sum，再添加-，得到target，故公式（设x等于+号和）：x + (sum - x) = target，x = (sum + target) / 2，此时问题转化为：填满背包大小为x的组合有多少种。

dp\[i]定义：填满i大小的背包，有dp\[i]种组合。

递推公式：dp\[j] += dp\[j - nums\[i]]，表示填满0大小背包组合数+填满1大小背包组合数+...

### 题kama52：完全背包

完全背包和01背包的区别是，完全背包可以重复地选择1件物品，而01不行。

在实现上，为支持1件物品被重复选择，只需要将内层背包循环按从小到大遍历即可。

```cpp
for (int i = 0; i < weight.size(); i++) {  // 遍历物品
	for (int j = weight[i]; j <= bagsize; j++) {  // 遍历重量，从小到大遍历
		dp[j] = max(dp[j], dp[j - weight[i]] + value[i]);
	}
}
```

### 题474：一和零

字符串数组，每个字符串0和1的数量，本质就是当前字符串元素的重量，0和1的数量分别不能超过m和n，那么m和n即为背包容量。所以这道题本质是背包问题。

dp\[i]\[j]定义：容量为i的背包A和容量为j的背包B所能承载的最多物品数。

### 题518：零钱兑换2

硬币可重复pick，完全背包问题。

**求组合数，而不是求最大价值。组合数**意味着没有顺序之分。

dp\[j]定义：背包容量为i时的组合数

递推公式：dp\[j] += dp\[j - coins\[i]]，表示：背包容量为j的组合数 = 不加coins\[i]当前背包容量为j的组合数 + 新增coins\[i]从j-coins\[i]继承的组合数

该递推公式要求dp\[0] = 1，不能为0，否则后续的推导全部为0。

| 背包dp槽位            | **0** | **1** | **2**                       | **3** | **4**                                | **5** |
| :---------------- | :---- | :---- | :-------------------------- | :---- | :----------------------------------- | :---- |
| coins\[0] = 1填充背包 | 1     | 1     | 1                           | 1     | 1                                    | 1     |
| coins\[1] = 2填充背包 |       |       | 2=dp\[0]+dp\[2]，更新到当前dp\[2] | 2     | 3=dp\[4]+dp\[2]，dp\[2]前面刚更新为2，所以这里为3 | 3     |
| coins\[2] = 5填充背包 |       |       |                             |       |                                      | 4     |

### 题377：组合总和4

**本题求排列数，有顺序之分。**

排列数和组合数在for循环有差别：

*   **如果求组合数就是外层for循环遍历物品，内层for遍历背包**。
*   **如果求排列数就是外层for遍历背包，内层for循环遍历物品**。

原因：如果把遍历nums（物品）放在外循环，遍历target的作为内循环的话，举一个例子：计算dp\[4]的时候，结果集只有 {1,3} 这样的集合，不会有{3,1}这样的集合，因为nums遍历放在外层，3只能出现在1后面！

### 题kama57：爬楼梯

爬楼梯变式，可以选择小于等于m阶步长，爬n阶大小楼梯。

背包：n，物品：\[1\~m]

递推公式：dp\[i] += dp\[i - j]，其中i是背包，j是物品

因为有序，所以背包在外，物品在内。

### 题322：零钱兑换

这道题求的是最小硬币数，前面求的是组合数。

二者区别在于递推公式的变化。最小硬币数：dp\[j] = min(dp\[j], dp\[j - nums\[i]] + 1)

其他注意点：因为是min，所以dp数组初始化为INT\_MAX

在遍历过程中，需规避INT\_MAX + 1情况，否则溢出。

递推公式的源头是dp\[0]，应初始化为0。

### 题279：完全平方数

思路和上题一致，求最小数量。只是元素上做了一点区别，不是直接给的，而是要自己做下平方。

### 题139：单词拆分

思路：单词数组中的单词可重复抽取 ==> 完全背包

背包：目标字符串。

dp\[j]定义：长度为j的字符串背包是否能被单词数组组合得到。

有序：遍历背包长度，遍历单词长度

递推公式：如果单词长度的子串“xxxxx\[子串]”在单词数组里，且dp\[xxxxx.len] == true, 则dp\[j] = true。

初始值：dp\[0] = true，其他为false

### 题kama56：多重背包

01背包限定了物品数量为1，完全背包不限定物品数量，多重背包介于二者之间，限定了物品数量，但不一定为1。

多重背包符合现实中的逻辑，物品资源是有限个的。

多重背包可以退化为01背包，只需要将多个物品A展开为每一个物品A。

所以整体代码框架是不变的，和01背包类似，唯一需要处理的是新增一个内部的for循环：

```cpp
#include <iostream>
#include <vector>
using namespace std;


int main() {
    int bagweight, n;
    cin >> bagweight >> n;
    
    vector<int> weights(n, 0);
    vector<int> values(n, 0);
    vector<int> nums(n, 0);
    for (int i = 0; i < n; i++) cin >> weights[i];
    for (int i = 0; i < n; i++) cin >> values[i];
    for (int i = 0; i < n; i++) cin >> nums[i];
    
    vector<int> dp(bagweight + 1, 0);
    for (int i = 0; i < n; i++) {
        for (int j = bagweight; j >= weights[i]; j--) {
            // 这里要根据物品数量做处理
            for (int k = 1, k < nums[i]; k++) {
                if (j - k * weights[i] < 0) continue;
                dp[j] = max(dp[j], dp[j - k * weights[i]] + k * values[i]);
            }
        }
    }
    return dp[bagweight];
}
```

### 题198：打家劫舍

错误思路：维护一个前面有没有偷的标记，这种思路会导致起始点的偷or不偷不好处理。

正确思路：偷or不偷，状态依赖。

dp\[i]定义：遍历到第i个物品时的最大金额。

递推公式：dp\[j] = max(dp\[i - 2] + nums\[i], dp\[i - 1])，这里就表示了前1个偷or不偷的关系

初始化：dp\[0] = nums\[0]，dp\[1] = max(nums\[0], nums\[1])

循环：只要1层循环即可，遍历物品。
