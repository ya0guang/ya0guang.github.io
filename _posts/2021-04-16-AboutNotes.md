---
layout: single
title: 关于博客的一些变化
date: 2021-04-16 11:30:07
categories: "Meta"
tags:
- Notes
- Research
- Course
header:
  teaser: 
---

最近好像博客都没啥更新，但其实不是的：我用上了TiddlyWiki作为自己的笔记本。

# Overview

它可以当作单一的html文件来使用，也可以在server被部署。

# Why Tiddlywiki?

最近我逐渐发现用markdown写笔记有的劣势：
- 没法交叉在各个笔记之间索引
- 没法更高效地整合信息
- 没法在单页标签内往复跳转

在参阅了一些大佬的Blog后，发现了一个神奇的笔记工具：Tiddlywiki。

它能够方便的以非线性的方式整合知识，感觉很适合我做源码分析和读论文的时候去使用。而且一些宏能够根据标签来统合在同一Tag下面的笔记，这对我而言是一个非常Nice的玩意。

我也曾尝试使用它的自动同步功能，做一个serverless的网页来使用Github的token往这个博客repo里面push它自己，但是我似乎失败了。于是目前的方案是：**Server + Build**，即在每次做出更新之后手动build，然后再commit到我的repo。

# What's there?

今后我想将他主要用于学术相关的内容，以英文为主，包括但不限于：

- 看的paper
- 分析的代码
- 学习的笔记

那这个Blog拿来干啥呢？估计写一些随笔和系统性的技术总结吧，以及自己探索的以西玩意。毕竟丢在Notes里面的知识都并非“一手原创”的，把它们丢在这里总感觉像是在剽窃别人的知识成果。