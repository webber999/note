[toc]

# Leetcode 题解与总结 -- 链表

## 1. 找出两个链表的交点（160）

```python
class ListNode(object):
    def __init__(self, x):
        self.val = x
        self.next = None


class Solution(object):
    def getIntersectionNode(self, headA, headB):
        """
        :type head1, head1: ListNode
        :rtype: ListNode
        """
        l1, l2 = headA, headB
        while l1 != l2:
            l1 = l1.next if l1 else headB
            l2 = l2.next if l2 else headA
        return l1
```

## 2. 链表反转 （206）

```python
# 原地迭代
class Solution(object):
    def reverseList(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        cur = head
        pre = None
        while cur:
            tmp = cur.next
            cur.next = pre
            pre = cur
            cur = tmp
        return pre

# 递归
class Solution(object):
    def reverseList(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        if not head or not head.next:
            return head
        cur = self.reverseList(head.next)
        head.next.next = head
        head.next = None
        return cur
```

## 3. 归并两个有序的链表 (21)

```python
class Solution(object):
    def mergeTwoLists(self, l1, l2):
        """
        :type l1: ListNode
        :type l2: ListNode
        :rtype: ListNode
        """
        root = cur = ListNode(0)
        while l1 and l2:
            if l1.val < l2.val:
                cur.next = l1
                l1 = l1.next
            else:
                cur.next = l2
                l2 = l2.next
            cur = cur.next
        if l1: cur.next = l1
        if l2: cur.next = l2
        return root.next
```

## 4. 从有序链表中删除重复节点 (83)

```python
class Solution(object):
    def deleteDuplicates(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        if not head:
            return head
        cur = head
        while cur.next:
            if cur.val == cur.next.val:
                cur.next = cur.next.next
            else:
                cur = cur.next
        return head
```

## 5. 删除链表的倒数第 n 个节点 (19)

```python
class Solution(object):
    def deleteDuplicates(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        if not head:
            return head
        cur = head
        while cur.next:
            if cur.val == cur.next.val:
                cur.next = cur.next.next
            else:
                cur = cur.next
        return head
```

## 6. 交换链表中的相邻结点 (24)

```python
class Solution(object):
    def deleteDuplicates(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        if not head:
            return head
        cur = head
        while cur.next:
            if cur.val == cur.next.val:
                cur.next = cur.next.next
            else:
                cur = cur.next
        return head
```

## 7. 链表求和 （445）

```python
class Solution(object):
    def addTwoNumbers(self, l1, l2):
        """
        :type l1: ListNode
        :type l2: ListNode
        :rtype: ListNode
        """
        stack1, stack2, stack_res = [], [], []
        head = ListNode(0)
        cur = head
        while l1:
            stack1.append(l1.val)
            l1 = l1.next
        while l2:
            stack2.append(l2.val)
            l2 = l2.next
        flag = 0
        while stack1 or stack2:
            if stack1 and stack2:
                tmp = stack1.pop() + stack2.pop() + flag
            elif stack1:
                tmp = stack1.pop() + flag
            else:
                tmp = stack2.pop() + flag
            flag = tmp // 10
            stack_res.append(tmp % 10)
        if flag:
            stack_res.append(flag)
        while stack_res:
            node = ListNode(stack_res.pop())
            cur.next = node
            cur = cur.next
        return head.next
```

## 8. 回文链表 (234)

要求：时间复杂度O(N), 空间复杂度O(1)

```python
class Solution(object):
    def isPalindrome(self, head):
        """
        :type head: ListNode
        :rtype: bool
        """
        length = 0
        cur = head
        while cur:
            length += 1
            cur = cur.next
        fast = head
        mid = length // 2 if length % 2 == 0 else length // 2 + 1
        while mid:
            mid -= 1
            fast = fast.next
        back = self.reverse(fast)
        front = head
        while back:
            if back.val != front.val:
                return False
            back = back.next
            front = front.next
        return True

    def reverse(self, node):
        cur = node
        pre = None
        while cur:
            tmp = cur.next
            cur.next = pre
            pre = cur
            cur = tmp
        return pre
```

## 9. 分隔链表 (725)

```python
class Solution(object):
    def splitListToParts(self, root, k):
        """
        :type root: ListNode
        :type k: int
        :rtype: List[ListNode]
        """
        length = 0
        res = []
        cur = root
        while cur:
            length += 1
            cur = cur.next
        cur = root
        width, remainder = length // k, length % k
        for i in range(k):
            head = cur
            for _ in range(width + (i<remainder) - 1):
                if cur:
                    cur = cur.next
            if cur:
                cur.next, cur = None, cur.next
            res.append(head)
        return res
```
## 10. 链表元素按奇偶聚集 (328)

```python
class Solution(object):
    def oddEvenList(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        if not head or not head.next:
            return head
        odd = head
        evenHead = head.next
        even = evenHead
        while even and even.next:
            odd.next = even.next
            odd = odd.next
            even.next = odd.next
            even = even.next
        odd.next = evenHead
        return head
```