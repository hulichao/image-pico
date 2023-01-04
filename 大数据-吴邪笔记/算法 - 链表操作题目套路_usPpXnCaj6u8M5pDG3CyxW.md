# 算法 - 链表操作题目套路

## 0. 前言

简单的题目，但是没有练习过或者背过，可能反而也写不出来，在面试中往往是在短时间内就写完，你没有时间画图，没有时间推演，这些都只能在脑子里快速完成，有时候拼了很久，觉得还是没有感觉，即使写出来了，在过后的一周到一个月照样会忘记，bug free地写出来还是很费力，作为对此深有体会的，或许跟我一样的有99%的人，像本文写的链表反转，如果可以在图上画出来，那你就一定可以写的出来，因为边界很简单，类似有快速排序荷兰国旗问题这些题目是国内面试级别上才会考的，比起像flag公司，还差那么一点，尽管自己在算法方面不是很开窍，包括在校招时候也练过不少题，但是我知道依然很少，而且对题目没有记忆感，总觉得自己是个傻子，这么简单的题目，竟然写不出来，不过依然觉得考算法和数据结构的其实才是公司考核程序员最公平最能考核思维的方式，说了这么多，包括本人在内，一直在身边各种算法大神碾压中，也期待在走向全栈工程师的道路上，单纯地从技术上，自己的算法和数据结构能越来越好把。

## 1.经典题目，链表反转

```java
//递归方式
public ListNode reverseList(ListNode head) {
    if (head == null || head.next == null)
      return head;
    ListNode next = head.next;
    ListNode new_head = reverseList(next);
    next.next = head;
    head.next = null;
    return new_head;
}
//遍历
public ListNode reverseList(ListNode head) {

    ListNode pre = null, cur = head,next = null;
    while( cur != null) {
        next = cur.next;
        cur.next = pre;
        pre = cur;
        cur = next;
    }

    return pre;
} 

```

## 2.相邻反转，部分反转

```java
//反转相邻节点链表1234 2143,反转5个的也类似
//迭代 四个指针
public ListNode swapPairs(ListNode head) {
    if (head == null || head.next == null)
        return head;
    
    //外面3指针pre,head,newHead
    ListNode newHead = new ListNode(-1);
    ListNode cur = head, pre = newHead;
    newHead.next = head;
    while(cur != null && cur.next != null) {
        //内部3指针
        ListNode first = cur;
        ListNode second = cur.next;
        ListNode third = cur.next.next;

        //换位
        pre.next = second;
        second.next = first;
        first.next = third;

        //移动两次
        pre = first;
        cur = third;

    }

    return newHead.next;
}

//递归
public ListNode swapPairs(ListNode head) {
    if (head == null || head.next == null)
        return head;
    
    ListNode newTmp = swapPairs(head.next.next);
    ListNode cur = head;
    ListNode next = cur.next;
    cur.next = newTmp;
    next.next = cur;
    return next;
}

//部分反转，12345 m=2 n=4 14325
public ListNode reverseBetween(ListNode head, int m, int n) {
        if (m < 1 || m >= n)
            return head;
        
        ListNode dump = new ListNode(-1);
        dump.next = head;
        ListNode pre = dump;
        for (int i = 1; i < m; i++) {
            pre = pre.next;
        }

        ListNode cur = pre.next;
        for (int i = m; i < n; i++) {
            ListNode nex = cur.next;
            cur.next = nex.next;
            nex.next = pre.next; //为什么不能指向cur? 因为没有指针移动，只有指向 1 2 3 4 5 -> 1 3 2 4 5
            pre.next = nex;
        }

        return dump.next;
        
    }

```

## 3.链表求和

```java
//两个链表求和
//输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
//输出：7 -> 0 -> 8
//原因：342 + 465 = 807
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
       ListNode resultListNode = new ListNode(0);
       ListNode current = resultListNode;
       int carry = 0;
       while(l1 !=null || l2 !=null){
           int sum = carry;
           if(l1!=null){
               sum += l1.val;
               l1 = l1.next;
           }
           if(l2!=null){
               sum += l2.val;
               l2 = l2.next;
           }
           int val = sum < 10?sum:sum - 10;
           carry = sum < 10 ?0:1;
           current.next = new ListNode(val);
           current =  current.next;
       }
        if(carry  == 1){
            current.next = new ListNode(1);
        }
        return resultListNode.next;
    } 
```
