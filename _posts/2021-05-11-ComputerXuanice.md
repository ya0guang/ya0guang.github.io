---
layout: single
title: 计算机玄学——主要是踩的C的坑
date: 2021-05-11 21:30:07
categories: "Hack"
tags:
- 二进制
- CS
- 逆向工程
header:
  teaser: https://upload.wikimedia.org/wikipedia/commons/thumb/6/69/64_qu%E1%BA%BB.png/330px-64_qu%E1%BA%BB.png
---

> 计算机没有玄学，因为它没有血肉。
> 但凡认为计算机玄学存在者皆是麻瓜。
>                           ——我

# `cdqe`

## 现象

和儿子debug的时候遇到了*计算机玄学*问题：

```c
    syscallnr = rec->nr;
    printf("syscallnr=%d\n", syscallnr);
    //WL: debug
    printf("In renderxxx\nsyscall table address: %p\n", syscalls);

    // cause seg fault
    entry = get_syscall_entry(syscallnr, false);

    // executed normally
    // entry = syscalls[syscallnr].entry;

    printf("entry: %p\n", entry);

    printf("syscall name: %s\n", entry->name);
    printf("arg1name: %s\n", entry->arg1name);
```
在执行上面这段代码的时候，`get_syscall_entry`会引起segmentation fault，然而把那一行注释掉执行的话则不会报错。然而这个函数的实现和被注释掉的那句话相比没有半毛钱的区别！实现如下：

```c
struct syscallentry * get_syscall_entry(unsigned int callno, bool do32)
{

    //WL: do32 is useless

    printf("In getxxx\nsyscall table address: %p\n", syscalls);
    printf("In getxxx\nentry: %p\n", syscalls[callno].entry);

    return syscalls[callno].entry;
}
```

## 朔源

运行的结果是：
![img](/assets/images/Xuanice/output.jpg)

两个`entry`的地址中，箭头标记是return之后的`entry`的值。可以看到return本身让值发生了变化。  
这个问题十分玄学，按理说`return syscalls[callno].entry`这句话不应该引起任何值本身的改变才对，毕竟如果不通过调用函数而是直接修改值是不会有任何问题的。于是我们来debug。

![img](assets/images/Xuanice/gef_before_ret.jpg)

首先，在return的时候，函数`get_syscall_entry`并没有做任何更改返回值（存放在`rax`）中的事情。此时的`rax`为：
![img](assets/images/Xuanice/rax_at_ret.jpg)

但是，在返回后出鬼了！注意这里的`rax`已经变成了奇奇怪怪的形状。然而元凶竟然是一个奇奇怪怪的`cbqe`指令！
![img](assets/images/Xuanice/after_ret.jpg)

查证[资料](https://80x86.dev/instruction/cdqe)后发现其作用是对于`EAX`的*signed extension*。这也就意味着`RAX`的高位会被抹去，和我们观测到的现象一样。

## 刨根

既然知道了这是由`cbqe`引起的，接下来就需要看看**为什么我没写的指令会被编译出来？**  

在查阅了大量资料后，我发现了Intel的[一篇博文](https://software.intel.com/content/www/us/en/develop/articles/a-collection-of-examples-of-64-bit-errors-in-real-programs.html?utm_source=feedburner)。其讲述了自己在Linux 64开发中踩得各种大坑。感觉其中的**Example 7**和我们遇到的问题十分相似。

> In C, you can use functions without preliminary declaration. Let's look at an interesting example of a 64-bit error related to this feature.

就是说C里面函数**不**声明也能用呗。但是由于C很**蠢**不知道你是怎么用的，于是就给你做了`signed extension`。解决方案很简单，就是在header里面重新定义这个函数即可。

如何**彻底**避免这种问题再次出现呢？**别用C了**。