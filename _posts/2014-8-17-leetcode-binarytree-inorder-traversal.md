---
layout: post
keywords: c++ algorithm
title: LeetCode题目详解——Binary Tree Inorder Traversal
categories: [c++]
tags: [c++,algorithm]
group: archive
icon: code
---

> 题目：<br />
Given a binary tree, return the inorder traversal of its nodes' values.
For example:
Given binary tree {1,#,2,3},<br />
1<br />
&nbsp;\ <br />
&nbsp; &nbsp;2<br />
&nbsp;/<br />
3<br />
return [1,2,3].<br />
Note: Recursive solution is trivial, could you do it iteratively?


中序遍历和先序遍历有很多相似的地方，只是在访问的顺序上有一点不太一样。非递归的遍历同样分为使用栈和不使用栈的方式。

#使用栈#

{% highlight c++ %}
class Solution {
public:
vector<int> inorderTraversal(TreeNode *root) {
    stack<TreeNode*> rest;
    vector<int> result;
    rest.push(root);
    TreeNode *cur;

    if(root == NULL)
        return result;
    cur=root;
    while(!rest.empty() || cur)
    {
        while (cur) {
            rest.push(cur);
            cur = cur->left;
        }
        if(!rest.empty())
        {
            cur = rest.top();
            result.push_back(cur->val);
            rest.pop();
            cur=cur->right;
        }
    }
    return result;
    }
};

{% endhighlight %}

#不使用栈#

不使用栈的方法即morris方法，利用改变树的结构，来记录接下来需要访问的节点。morris方法的先序遍历和中序遍历很相似，只有一句代码的位置不一样。对于morris方法不太了解的同学可以参考的前一篇文章：[LeetCode题目详解——Binary Tree Preorder Traversal](http://blog.csdn.net/asialiyazhou/article/details/38636819) 代码如下：

{% highlight c++ %}
class Solution {
public:
vector<int> inorderTraversal(TreeNode *root) {
    vector<int> result;
    TreeNode *p=root;
    if(!root)
        return result;
    while(p)
    {
        if(p->left==NULL)
        {
            result.push_back(p->val);
            p=p->right;
        }
        else
        {
            TreeNode *tmp=p->left;
            while(tmp->right && tmp->right!=p)
            {
                tmp=tmp->right;
            }
            if(tmp->right==NULL)
            {
                tmp->right=p;
                p=p->left;
            }
            else
            {
                result.push_back(p->val);//只有这句代码的位置和先序遍历的时候不太一样
                tmp->right=NULL;
                p=p->right;
            }
        }
    }
return result;
}

};
{% endhighlight %}