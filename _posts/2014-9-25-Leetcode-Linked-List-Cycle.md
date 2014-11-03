---
layout: post
title: LeetCode Linked List Cycle & Linked List Cycle II题解
keywords: 算法
categories: [Algorithm]
tag: [Algorithm,List]
icon: code
---

#Linked List Cycle I#

> Given a linked list, determine if it has a cycle in it.Follow up:Can you solve it without using extra space?

题目的意思就是在利用O（1）的空间判断一个链表是否存在环。我一开始的想法就是，每访问 一个节点，就遍历这个节点前面的所有节点，然后判断是否相同，如果相同就说明有环。如果所有的节点都访问完了还是没有发现有节点相同，则说明该链表没有环。但是这个想法很快被毙掉，因为这个算法的复杂度将会达到O(N^2)，实在太慢！后来又想到了利用两个指针，一快一慢，如果有环，则这两个指针一定会相遇。这个方法不需要额外的空间，同时复杂度是O（N）的。非常符合这道题目的要求。具体来说，就是设立两个指针，一个fast，一个slow。fast在每一个step中移动两步，slow每次移动一步。如果两者相遇，必然是存在环，同时fast将slow套圈了才会出现这种情况。这道题目也就迎刃而解了，具体的实现代码如下。

{% highlight C++ %}
class Solution {  
public:  
    bool hasCycle(ListNode *head) {  
    if(!head || !head->next)  
        return false;  

    ListNode *fast,*slow;  
    fast = slow = head;  
    bool flag = false;  
    while(fast)  
    {  
        fast = fast->next;  
        if(flag)  
        {  
            slow = slow->next;  
            flag = false;  
        }  
        else  
        {  
            flag = true;  
        }  
        if(fast == slow)  
        return true;  
    }  
    return false;  
}  
}; 
{% endhighlight %}

#Linked List Cycle II#

> Given a linked list, return the node where the cycle begins. If there is no cycle, return null. Follow up:Can you solve it without using extra space?

Linked List Cycle II 不仅要求判断是否存在环，同时还需要在存在环的情况下找出环的起始节点。这就比I要难一些。最开始我想到的方法还是跟上题类似，一个fast ，每次移动两步，一个slow，每次移动一步。两个指针不仅要向前移动，同时还需要记录各自走的步数（fastCount和slowCount）。当相遇的时候，fastCount减去slowCount就是环的长度（假设这个长度的len）。这个时候让fast和slow重新指向head节点。然后先让fast指针向前移动len步。之后fast和slow再同时移动，两个每次均移动一步。当两者相遇的时候就是环的起始节点。具体代码如下：

{% highlight C++ %}
class Solution {  
public:  
ListNode *detectCycle(ListNode *head) {  
    if(!head || !head->next)  
        return NULL;  
    ListNode *fast,*slow;  
    int fastCount = 1,slowCount = 1;  
    bool hasCycle = false, flag = false;  
    fast = slow = head;  
    while(fast)  
    {  
        fast = fast->next;  
        ++fastCount;  
        if(flag)  
        {  
            slow = slow->next;  
            ++slowCount;  
            flag = false;  
        }  
        else  
        {  
            flag = true;  
        }  
        if(fast == slow)  
        {  
            hasCycle = true;  
            break;  
        }  

    }  

    if(!hasCycle)  
        return NULL;  
    int cycleLen = fastCount - slowCount;  

    fast = slow = head;  
    for(int i = 0; i < cycleLen; ++i)  
        fast = fast->next;  
    while(true)  
    {  
        if(fast == slow)  
            return slow;  
        fast = fast->next;  
        slow = slow->next;  
    }  
    }  
}; 
{% endhighlight %}

但是在参考了他人的算法之后，发现这个算法可以更加精简。先来看下面这个图:
![](/image/2014-9-25-Leetcode-Linked-List-Cycle/1.jpg)

p1为环开始的节点，p2为fast指针和slow指针相遇的节点。A 、B、 C分别为三段的距离。当相遇的时候slow指针经过的路程为（A + B）而fast指针经过的路程为(A + B + C + B)。同时fast经过的路程又是slow的两倍，所以又（A + B + C + B）= 2（A + B）所以最后得到A = C！那么当两者相遇的时候，只需要将slow指针放回到起始节点，而fast指针继续在原有位置上向前走。两个指针一相同的速度访问节点。当两者再次相遇的时候，相遇的节点就是环路起始节点p1。实现代码如下：

{% highlight C++ %}
class Solution {  
    public:  
    ListNode *detectCycle(ListNode *head) {  
        if(!head || !head->next)  
            return NULL;  
    ListNode *fast,*slow;  
    fast = slow = head;  
    while(fast)  
    {  
        fast = fast->next;  
        if(!fast) return NULL;  
        fast = fast->next;  
        slow = slow->next;  

        if(fast == slow)//find the cycle  
        {  
            slow = head;  
            while(slow != fast)  
            {  
                fast = fast->next;  
                slow = slow->next;  
            }  
        return fast;  
        }  
    }  
}  
};  
{% endhighlight %}