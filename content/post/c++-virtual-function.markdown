---
author: admin
comments: true
date: 2013-09-16 14:16:48+00:00
layout: post
slug: c%e7%9a%84%e8%99%9a%e5%87%bd%e6%95%b0%e6%8c%87%e9%92%88%e6%98%af%e5%90%91%e4%b8%8b%e4%bc%a0%e9%80%92%e7%9a%84
title: c++的虚函数指针是向下传递的
wordpress_id: 282
categories:
- c
tags:
- 函数指针
- 虚函数
---

```c++
typedef void (Base::*bfnPrint)();
bfnPrint func = ::Print;
```

就是说虽然函数指针func取值对象用的是父类.
但是如果Print函数是虚函数,
利用子类实例调用func函数的时候,实际上会调用到子类的Print函数.

```c++
// virtual_functions.cpp
// compile with: /EHsc
#include <iostream>
using namespace std;

class Base
{
public:
    virtual void Print();
};

void Base :: Print()
{
    cout << "Print function for class Base\n";
}

class Derived : public Base
{
public:
    virtual void Print();
};

void Derived :: Print()
{
    cout << "Print function for class Derived\n";
}

int main()
{
    Base    bObject;
    Derived dObject;

    typedef void (Base::*bfnPrint)();
    bfnPrint func = &Base::Print;


    (&bObject->*func)();
    (&dObject->*func)();
}
```

print out

```shell
Print function for class Base
Print function for class Derived
```
