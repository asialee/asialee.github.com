---
layout: post
keywords: c++ algorithm
title: LeetCode题目详解——Binary Tree Postorder Traversal
categories: [c++]
tags: [c++,algorithm]
group: archive
icon: code
---

> 题目：<br />
Given a binary tree, return the postorder traversal of its nodes' values.
For example:
Given binary tree {1,#,2,3},<br />
1<br />
&nbsp;\ <br />
&nbsp; &nbsp;2<br />
&nbsp;/<br />
3<br />
return [1,2,3].<br />
Note: Recursive solution is trivial, could you do it iteratively?


后序遍历比先序和中序遍历要复杂一些，因为一个节点必须要它的左子树和右子树都被遍历之后才能被访问。所以被压入栈的元素重新被访问到的时候（即此时该元素位于栈顶），必须要判断该节点的左右子树是不是已经被遍历过，如果左右子树都已经被遍历，那么访问该元素，然后出栈。接着再处理新的栈顶；否则则将该节点的左右子树压入栈。具体代码如下：

{% highlight c++ %}
class Solution {//应该是最优化的实现方法
//每次从栈中取出一个节点，通过判断当前节点是否可以结束访问。如果可以则将其对应的值放到vector中，并从栈中取出下一个要访问的节点；如果不可以则将其右左节点分别压入栈中
public:
vector<int> postorderTraversal(TreeNode *root) {
    stack<TreeNode*> rest;
    vector<int> result;
    rest.push(root);
    TreeNode *cur,*pre;
    if(root == NULL)
        return result;
    while(!rest.empty())
    {
        cur = rest.top();
        if((cur->left==NULL && cur->right==NULL) || (pre!=NULL)&&(pre==cur->left)||(pre==cur->right))//当前节点可以结束访问的条件：1、没有子树；2、前一个被访问的节点为当前节点的子树
        {
            result.push_back(cur->val);
            pre = cur;
            rest.pop();
        }
        else
        {
            if(cur->right)
                rest.push(cur->right);
            if(cur->left)
                rest.push(cur->left);
        }
    }
}
};
{% endhighlight %}