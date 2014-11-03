---
layout: post
description: css下position属性详解
keywords: web HTML CSS
title: css下position属性详解
categories: [web]
tags:   [CSS,position]
group: [Web]
icon: code
---

写过CSS的人都免不了与position属性打交道，但是要真正理解position属性还不是一个很容易的事。前两天博主想在一个html页面上实现一个<div>元素重叠在另一个<div>元素上,并且位于该<div>元素的右下角的效果。在网上搜到他人的解决方法，并且也实现了，这里面最关键的就是利用position属性以及left、right、top、bottom等属性。为了更为透彻地理解背后原理，博主在网上搜了相关的资料，总算是有了一些认识，或许也只是一知半解，不过先写下来供大家分享。

在html中有块元素和行元素之分。块元素诸如div、p等，其中的子元素会按垂直方向排列，这些元素显示为一块内容；与之相反，span、strong等等元素被称为行内元素，他们的内容显示在行中，即横向扩展。

凡是写在HTML中的元素都会被加载到文档流中，简单的说文档流就相当于一个容器，HTML中的所有元素都是按照从上到下，从左到右的顺序被加载到文档流中，然后在渲染网页的时候，再根据文档流中的顺序将元素一个一个显示在网页中。所以在文档流中的元素都是根据相对位置来进行绘制的。不过并不是所有的元素都会被放入到文档流中，比如position为absolute、fixed等等情况时。

那么现在就正式来讲position属性。position属性共有4个值，分别是relative、absolute、fixed和static：

- static：这是position的默认值，static元素会出现在正常的文档流中，并且按照磨人的规则绘制；
- relative：position为relative的元素依然会出现在文档流中，设置为relative的元素同时是希望在正常的显示位置的基础上进行一些微 调。例如“left：20px”的意思就是在正常显示位置的基础上左边缩进20像素；
- absolute：position为absolute的元素会从文档流中删除，absolute元素的绘制不再以正常位置进行显示，而是以其第一个position不为static的父元素为定位范围，再根据left、right等等属性进行定位。
- fixed：position为fixed的元素的定位规则更absolute类似，只是fixed元素的定位范围不再是父元素而是整个窗口。所以设置为fixed的元素在用户滚动浏览网页时，其相对于浏览器窗口的位置也不会改变。

好了，基本的定义解释清楚了，现在就结合博主自己的实践来谈谈具体的使用，博主想要实现的效果如下：
![](/image/2014-10-11-css-position/1.jpg)

即一张图片的右下角有一个按钮。根据上文讲解到的知识，这里应该有两个div元素，其中一个为另一个的父元素：

{% highlight html %}
<div class="background_img" >  
<img src="***" />  
<div class="btn"><input type="button" /></div>  
</div> 
{% endhighlight%}

既然class为btn的<div>要相对于父元素定位，所以其position为absolute，同时再通过bottom和right来设置位置：
{% highlight css %}
div.btn{  
position:absolute;  
right:10px;  
bottom:10px;  
}  
{% endhighlight%}

而对于class为background_img的<div>元素来说，其position元素不能为默认的static，所以设置为relative即可：
{% highlight css %}
div.background_img  
{  
position:relative;  
} 
{% endhighlight %}
就这么几行，这个效果就实现了，各位读者可以自己试一下。





