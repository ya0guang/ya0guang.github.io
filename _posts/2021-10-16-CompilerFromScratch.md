---
layout: single
title: 从零开始手搓编译器
date: 2021-10-16 22:00:07
categories: "Tech"
tags:
- CS
- PL
- Notes
header:
  teaser: 
---

今年手痒痒选了一个Implementation of PL的课，需要徒手搓编译器。这里来小记一下这个过程。

其实老师给了Racket和Python两种语言的选择，奈何我实在是不习惯那长到姥姥家的Racket括号，于是选择了Python。幸好在最新的Python 3.10中引入了`match case`的新特性，这直接使得代码量减少了非常多！它太好用了！！！

源码目前放在[这个repo](https://github.com/ya0guang/ToyPythonCompiler)里。里面将阶段性的进展都打上了tag。如若有人有兴趣的话可以去看看代码！我觉得我的代码质量还算凑合？

这个坑先开着，慢慢填！
