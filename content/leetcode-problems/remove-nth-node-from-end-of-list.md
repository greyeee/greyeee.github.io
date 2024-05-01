+++
title = '19. 删除链表的倒数第 N 个结点'
date = 2024-05-01T10:27:41+08:00
draft = false

+++

## [题目](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)

给你一个链表，删除链表的倒数第 `n` 个结点，并且返回链表的头结点。



**示例 1：**

![img](https://assets.leetcode.com/uploads/2020/10/03/remove_ex1.jpg)

```
输入：head = [1,2,3,4,5], n = 2
输出：[1,2,3,5]
```

**示例 2：**

```
输入：head = [1], n = 1
输出：[]
```

**示例 3：**

```
输入：head = [1,2], n = 1
输出：[1]
```



**提示：**

- 链表中结点的数目为 `sz`
- `1 <= sz <= 30`
- `0 <= Node.val <= 100`
- `1 <= n <= sz`

## 解题思路

双指针，`fast`指针先走`n`步，再同时移动`fast`和`slow`，当`fast`到达尾节点时，`slow`就指向倒数第n个结点的前一个节点。 `fast`指针最终停在尾节点，所以循环条件判断用`fast.Next != nil`，而不是`fast != nil`。 有可能删除的是头节点，所以引入虚拟头部。

## 代码

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func removeNthFromEnd(head *ListNode, n int) *ListNode {
    dummy := &ListNode{Next: head}
    slow, fast := dummy, dummy
    for ; n > 0; n-- {
        fast = fast.Next
    }
    for fast.Next != nil {
        fast = fast.Next
        slow = slow.Next
    }
    slow.Next = slow.Next.Next
    return dummy.Next
}
```

