# 算法-双指针问题解决思路

算法中的双指针使用，有时候会觉得很巧妙，解决了很多的问题，有必要归纳总结一下，首先双指针也是个很宽泛的概念，它类似于遍历中的 `i 和 j` 但是其区别是，两个指针是同时移动的，即没有贡献复杂度从`O(N)` 到 `O(N*N)` ，所以被很多算法大佬所推崇，所以基于此归纳总结出双指针的常见解法和套路。

# 1.题型归纳

这里将题型归纳为三种，分别如下：

-   快慢指针（前后按不同步调的两个指针）
-   前后双端指针（一个在首，一个在尾部，向中间靠拢）
-   固定间隔的指针（以i, i + k的间距的两个指针）

前面讲到，这三种指针都是遍历一次数组即可完成，其时间复杂度低于或者等于`O(N)` ，空间复杂度是`O(1)` 因为就两个指针存储。

## 2.常见题型

### 2.1快慢指针

相当经典的一道题：

-   判断链表是否有环-

通过步调不一致的两个指针，一前一后一起移动，直到指针重合了

[https://leetcode.com/problems/linked-list-cycle/description](https://leetcode.com/problems/linked-list-cycle/description/ "https://leetcode.com/problems/linked-list-cycle/description")，代码片段如下：

```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head;
          ListNode fast = head;
    while (slow != null && fast != null) {
                 ListNode n = fast.next;
     fast = n == null ? null : n.next;
     if (slow == fast) {
         return true;
     }
                 slow = slow.next;
    }
    return false;
    }
```

-   寻找重复复数，从数组中找出一个重复的整数：[https://leetcode-cn.com/problems/find-the-duplicate-number/](https://leetcode-cn.com/problems/find-the-duplicate-number/ "https://leetcode-cn.com/problems/find-the-duplicate-number/")

代码解决如下：

```java
public int findDuplicate(int[] nums) {
        // 将其看成是一个循环的链表，快慢指针循环
        int index1 = 0;
        int index2 = 0;
        do
        {
            index1 = nums[index1];
            index2 = nums[index2];
            index2 = nums[index2];
            
        }while (nums[index1] != nums[index2]);
        index1 = 0;
// 找出在哪个位置为起始点，可证必定在圆圈起点相遇
        while(index1 != index2){
            index1 = nums[index1];
            index2 = nums[index2];
        }
        return index1;
    }
```

### 2.2 前后首尾端点指针

-   二分查找

二分查找是典型的前后指针的题型，代码如下：

```java
public static int binarySearch(int[] array, int targetElement) {
    int leftIndex = 0, rightIndex = array.length - 1, middleIndex = (leftIndex + rightIndex) / 2;
    
    while(leftIndex <= rightIndex) {
      int middleElement = array[middleIndex];
      if(targetElement < middleElement) {
        rightIndex = middleIndex - 1;
      }else if(targetElement > middleElement) {
        leftIndex = middleIndex + 1;
      }else {
        return middleIndex;
      }
      
      middleIndex = (leftIndex + rightIndex) / 2;
    }
    
    return -1;
  }

```

## 2.3 固定间隔的指针

-   求链表中点[https://leetcode-cn.com/problems/middle-of-the-linked-list/](https://leetcode-cn.com/problems/middle-of-the-linked-list/ "https://leetcode-cn.com/problems/middle-of-the-linked-list/")

```java
// 快指针q每次走2步，慢指针p每次走1步，当q走到末尾时p正好走到中间。

class Solution {
    public ListNode middleNode(ListNode head) {
        ListNode p = head, q = head;
        while (q != null && q.next != null) {
            q = q.next.next;
            p = p.next;
        }
        return p;
    }
}
```

-   求链表倒数第k个元素 [https://leetcode-cn.com/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/](https://leetcode-cn.com/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/ "https://leetcode-cn.com/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/")

```java
    // 快慢指针，先让快指针走k步，然后两个指针同步走，当快指针走到头时，慢指针就是链表倒数第k个节点。

    public ListNode getKthFromEnd(ListNode head, int k) {
        
        ListNode frontNode = head, behindNode = head;
        while (frontNode != null && k > 0) {

            frontNode = frontNode.next;
            k--;
        }

        while (frontNode != null) {

            frontNode = frontNode.next;
            behindNode = behindNode.next;
        }

        return behindNode;
    }
```

## 3. 模板总结

看完三个代码，是不是觉得很简单，下面总结一下三种双指针的代码模板

```java
// 1.快慢指针
l = 0
r = 0
while 没有遍历完
  if 一定条件
    l += 1
  r += 1
return 合适的值

//2. 左右端点指针
l = 0
r = n - 1
while l < r
  if 找到了
    return 找到的值
  if 一定条件1
    l += 1
  else if  一定条件2
    r -= 1
return 没找到

//3. 固定间距指针
l = 0
r = k
while 没有遍历完
  自定义逻辑
  l += 1
  r += 1
return 合适的值
```
