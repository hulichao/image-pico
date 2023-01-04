# 算法 - 链表操作思想 && case

[算法 - 链表操作题目套路](<算法 - 链表操作题目套路_usPpXnCaj6u8M5pDG3CyxW.md> "算法 - 链表操作题目套路") 前面这一篇文章主要讲链表操作时候的实操解决方式，本文从本质讲解链表操作的元信息，学完后，再也不怕链表操作题目了。

# 1.链表的基本操作

链表的基本操作无外乎插入，删除，遍历

插入的化，要考虑到前驱节点和后继节点，记住下面的伪代码

```sql
nex = 当前节点.next
当前节点.next = 插入的指针
插入指针.next = tmp 
```

对于删除，是否会觉得需要备份一下next的指针，答案是不用，执行语句是先取右边的值存起来然后赋值给左边的，所以直接下面一句话即可

```sql
cur.next = cur.next.next
```

对于遍历，实际上是又迭代和递归的，另外又有前序和后序

```sql
cur =  head
while cur != null {
   print(cur)
   cur = cur.next
}

//前序
dfs(cur) {
    if cur == null return
    print(cur.val)
    return dfs(cur.next)
} 
```

# 2.链表的考点

链表的考点就只有对指针的修改和对指针的拼接，其中都需要对指针的前驱节点和后继节点进行保留，如果头节点不能首次确定，或者可能会改变，那么又需要一个虚拟头节点，链表的考点无外乎就这些。

下面我们来看两个题

删除链表中的重复元素

[https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list-ii/](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list-ii/ "https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list-ii/")

```sql
public ListNode deleteDuplicates(ListNode head) {
    //递归版
        if (head == null || head.next == null) 
            return head;
        
        if (head.val == head.next.val) {
            while (head.next != null && head.next.val == head.val) 
                head = head.next;
            head = deleteDuplicates(head.next);
        } else {
            head.next = deleteDuplicates(head.next);
        }

        return head;
} 
```

判断是否为回文链表

[https://leetcode-cn.com/problems/palindrome-linked-list/](https://leetcode-cn.com/problems/palindrome-linked-list/ "https://leetcode-cn.com/problems/palindrome-linked-list/")

```sql
public boolean isPalindrome(ListNode head) {
        if(head == null || head.next == null)return true;
        temp = head;
        return dfs(head);
}

public boolean dfs(ListNode head){
    if(head == null)return true;
    boolean res = dfs(head.next) && temp.val == head.val;
    temp = temp.next;
    return res;
}
```

总结，链表的题目，掌握号指针的操作，就会简单点了，对于递归写法也会简化很多代码
