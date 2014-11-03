---
layout: post
title: LeetCode算法题——Word Break I &II
keywords: 算法
categories: [Algorithm]
tag: [Algorithm,Dynamic_Planning,Back_Tracing]
icon: code
---

这两天做了Leetcode上的Word Break I 和Word Break II两道题，里面涉及到了动态规划和回溯等等方面的知识，是一道很好的题目，所以把自己的做法写出来和大家一起分享。


#Word Break I#
>Given a string s and a dictionary of words dict, determine if s can be segmented into a space-separated sequence of one or more dictionary words.</br >
For example, given</br >
s = "leetcode",</br >
dict = ["leet", "code"].</br >
Return true because "leetcode" can be segmented as "leet code". 


第一题比较简单，自然而然都会想到是用动态规划的方法来解决。具体来说可以用一个bool类型的数组`bool flag[n]`其中n是字符串s的长度。flag[i]表示字符串的前i-1位是否可以被切分。而flag[i]==true的条件是：

- s.substr(0,i - 1) 属于 dict
- 存在一个j，使得flag[j] == true && s.substr(j,i - j - 1) 属于dict

这个公式得到之后，代码就简单了。

{% highlight c++ %}

{% endhighlight %}

Word Break I是总体来说是一个简单的动态规划。唯一容易出错的地方就是控制字符串的脚标时，一定要很小心。对于这一点我的经验就是，针对每一个脚标变量（比如上代码中的i，j）时，一定要明确这个变量是偏移量比如字符串的长度还是位置变量。在通过这些脚标量计算另外的脚标量的时候注意是否需要做减1或者加1的操作。另外就是要注意边缘条件，即处理字符串的头尾的时候，有无特殊情况。


#Word Break II#
>Given a string s and a dictionary of words dict, add spaces in s to construct a sentence where each word is a valid dictionary word.Return all such possible sentences.</br >
For example, given</br >
s = "catsanddog",</br >
dict = ["cat", "cats", "and", "sand", "dog"].</br >
A solution is ["cats and dog", "cat sand dog"].</br >

第二题不仅要求需要得出字符串s是否可以被切分，同时还需要得出所有的切分组合。前者毫无疑问还是需要使用动态规划来解决，而对于
后者，列出所有组合这种情况基本都是使用回溯的方法。看了网上很多人的做法，基本都是利用一个二维数组match[n][n]，match[i][j]记录字符串的i到j位知否可以被切分。具体做法如下：

- 利用动态规划算法得到match矩阵
- 对match进行深度优先搜索列出所有情况

这种做法在网上随便一搜就可以搜到，这里就不讲了。来讲讲我自己的做法吧：不用二维数组match，而使用两个一维数组flag[n],pos[n]。前者和上题中的含义相同。后者则是回溯的关键，pos数组的功用有点像Dijkstra中的pos数组：

>如果pos[n - 1] = j,说明字符串的j到n-1位是一个完整的单词，将其记录下来。同时下一次的起始位置为pos[j - 1];如此迭代之后到达字符串s的首位，此时就得到了一种切分。

而算法的核心就是通过回溯得到不同的pos数组，从而列举出所有的情况。


