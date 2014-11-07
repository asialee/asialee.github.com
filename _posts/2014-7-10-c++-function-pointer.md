---
layout: post
keywords: c++
title: (翻译)c++下类函数机制详解
categories: [c++]
tags: [c++,类函数]
group: archive
icon: code
---
[原文出处](http://www.codeguru.com/cpp/cpp/article.php/c17401/C-Tutorial-PointertoMember-Function.htm#6)
在C++中类成员函数指针是一种比较特别的指针，尽管直接使用类成员函数的情况不太多，但是还是有必要详解一下这类指针。

#具体语法#
首先说明一下类成员函数指针的声明方式：

{% highlight c++ %}
Return_Type (Class_Name::* pointer_name) (Argument_List);
Return_Type:   member function return type.
Class_Name:    name of the class in which the member function is declared.
Argument_List: member function argument list.
pointer_name:  a name we'd like to call the pointer variable.
{% endhighlight %}


通过函数指针调用类成员函数需要使用操作符.*或者->*,具体使用参考下面这段代码：

{% highlight c++ %}
#include <iostream>
#include <string>
using std::string;

class Foo{
public:
int f(string str){
std::cout<<"Foo::f()"<<std::endl;
return 1;
}
};

int main(int argc, char* argv[]){
int (Foo::*fptr) (string) = &Foo::f;
Foo obj;
(obj.*fptr)("str");//call: Foo::f() through an object
Foo* p=&obj;
(p->*fptr)("str");//call: Foo::f() through a pointer
}
{% endhighlight %}

#类成员函数指针#
通常的函数指针指向的地址都是一个准确的代码入口地址，但是不难想象类成员函数指针指向的应该是一个相对的位置（相对于整个类的偏移地址）。这一点很好证明。
类中的静态函数实际和全局函数一样，有着确定的入口地址，类成员静态函数没有this指针，但是却可以共享整个类的命名范围，也可以访问到类中的其他成员。现在来看下面这段代码：

{% highlight c++ %}
#include <iostream>
#include <string>
using std::string;

class Foo{
public:
    static int f(string str){
        std::cout<<"Foo::f()"<<std::endl;
        return 1;
    }
};

int main(int argc, char* argv[]){
    int (Foo::*fptr) (string) = &Foo::f;//error
{% endhighlight %}

{% highlight c++ %}
int (Foo::*fptr) (string) = &Foo::f;//error
{% endhighlight %}

会引起如下错误：
cannot convert 'int (*)(std::string)' to 'int (Foo::*)(std::string）
这说明了类成员函数指针不是一般的函数指针

#类成员函数指针的转换#

{% highlight c++ %}
#include <iostream>
class Foo{
    public:
    int f(char* c=0){
        std::cout<<"Foo::f()"<<std::endl;
        return 1;
    }
};

class Bar{
    public:
    void b(int i=0){
        std::cout<<"Bar::b()"<<std::endl;
    }
};

class FooDerived:public Foo{
    public:
    int f(char* c=0){
        std::cout<<"FooDerived::f()"<<std::endl;
        return 1;
    }
};

int main(int argc, char* argv[]){
    typedef  int (Foo::*FPTR) (char*);
    typedef  void (Bar::*BPTR) (int);
    typedef  int (FooDerived::*FDPTR) (char*);

    FPTR fptr = &Foo::f;
    BPTR bptr = &Bar::b;
    FDPTR fdptr = &FooDerived::f;

    //bptr = static_cast<void (Bar::*) (int)> (fptr); //error
    fdptr = static_cast<int (Foo::*) (char*)> (fptr); //OK: contravariance

    Bar obj;
    ( obj.*(BPTR) fptr )(1);//call: Foo::f()
    }

Output:
Foo::f()
{% endhighlight %}

语句：
`bptr = static_cast<void (Bar::*) (int)> (fptr);`无法正常执行，因为非静态、非虚的类成员函数指针都是属于强类型，所以无法进行转换。
但是语句
`fdptr = static_cast<int (Foo::*) (char*)> (fptr);`可以执行，这是因为两者所属的类存在继承关系。

#虚成员函数指针#

{% highlight c++ %}
#include <iostream>
class Foo{
    public:
    virtual int f(char* c=0){
        std::cout<<"Foo::f()"<<std::endl;
        return 1;
    }
};

class Bar{
    public:
    virtual void b(int i=0){
        std::cout<<"Bar::b()"<<std::endl;
    }
};

class FooDerived:public Foo{
    public:
    int f(char* c=0){
        std::cout<<"FooDerived::f()"<<std::endl;
        return 1;
    }
};

int main(int argc, char* argv[]){
    typedef  int (Foo::*FPTR) (char*);
    typedef  void (Bar::*BPTR) (int);
    FPTR fptr=&Foo::f;
    BPTR bptr=&Bar::b;

    FooDerived objDer;
    (objDer.*fptr)(0);//call: FooDerived::f(), not Foo::f()

    Bar obj;
    ( obj.*(BPTR) fptr )(1);//call: Bar::b() , not Foo::f()
}

Output:
FooDerived::f()
Bar::b()
{% endhighlight %}