---
title: 未使用c++11的时候，空指针使用NULL或0可能带来的问题之一
date: 2017-11-29 02:22:26
categories: 编程语言
tags: c++11
---

### 问题描述

`void foo(type1* t1)`

`void foo(int i)`

c++中用NULL和0来表示空指针，NULL就是0，所以在调用foo的2个重载函数的时候，如果参数传递的是type1类型的空指针， 会调用到参数是int的重载foo，这样就带来了一个隐性问题。

使用c++11中的nullptr的话就没有这个问题。关键字nullptr是std::nullptr_t类型的值，用来指代空指针。nullptr和任何指针类型以及类成员指针类型的空值之间可以发生隐式类型转换，同样也可以隐式转换为bool型（取值为false）。但是不存在到整形的隐式类型转换。

