---
layout: post
keywords: c++ algorithm
title: LeetCode题目详解——Binary Tree Preorder Traversal
categories: [c++]
tags: [c++,algorithm]
group: archive
icon: code
---

> 题目：<br />
Given a binary tree, return the preorder traversal of its nodes' values.
For example:
Given binary tree {1,#,2,3},<br />
1<br />
&nbsp;\ <br />
&nbsp; &nbsp;2<br />
&nbsp;/<br />
3<br />
return [1,2,3].<br />
Note: Recursive solution is trivial, could you do it iteratively?


二叉树的遍历是一个基础问题，具体涉及到先序、中序和后序问题，基本的方法有递归和非递归，其中非递归的思路又分为使用栈以及不使用栈。递归的方法代码很简单，但是开销很大，所以在此不表。仅针对leetcode上对应的题目给出非递归的两种方法。

#一、使用栈#
二叉树非递归遍历很自然地会联想到使用栈来记录还没有访问的节点。每次从栈中pop出一个节点，并访问该节点，同时将其右左子树（顺序很重要，是右左）压入栈中，直到栈为空的时候，遍历结束。代码如下：
{% highlight c++ %}
class Solution {
public:
    vector<int> preorderTraversal(TreeNode *root) {
        stack<TreeNode*> nodeStack;
        vector<int> valVec;
        if(!root)
            return valVec;
        nodeStack.push(root);
        while(!nodeStack.empty())
        {
            TreeNode* top = nodeStack.top();
            valVec.push_back(top->val);
            nodeStack.pop();
            if(top->right)
                nodeStack.push(top->right);
            if(top->left)
                nodeStack.push(top->left);
        }
    }
};
{% endhighlight %}

#不使用栈#

使用栈的作用就是利用栈来记录接下来需要访问的节点，如果不使用栈的话，那么就必须想出其他的方法来记录接下来需要访问的节点。Morris设计出利用改变树本身的结构来记录接下来需要访问的节点，并在访问完之后再将树的节点改变回来。具体的思路就是：对于当前要访问的节点curNode，找到左子树的最右节点tmp（如果没有左子树就不必了），然后将curNode节点赋值给tmp的右子树。这样就可以利用树本身的结构来记录需要访问的信息。当访问完curNode的左子树之后，再将tmp的右子树赋值为NULL，以恢复树原有的结构。代码如下：

{% highlight c++ %}
class Solution {
public:
vector<int> preorderTraversal(TreeNode *root) {
    vector<int> valVec;
    TreeNode *p=root;
    if(!root)
        return valVec;
    while(p)
    {
        if(p->left==NULL)
        {
            valVec.push_back(p->val);
            p=p->right;
        }
        else
        {
            TreeNode *tmp=p->left;
            while(tmp->right && tmp->right !=p)
            {
                tmp=tmp->right;
            }
            if(tmp->right==NULL)
            {
                valVec.push_back(p->val);
                tmp->right=p;
                p=p->left;
            }
            else
            {
                tmp->right=NULL;
                p=p->right;
            }
        }
    }
}
};
{% endhighlight %}

如果觉得还是不好理解的话，那么可以看看接下来的几幅图以帮助理解。
![](/image/2014-8-17-Leetcode-binarytree-preorder-traversal/1.jpg)
