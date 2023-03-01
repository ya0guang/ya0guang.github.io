---
layout: single
title: Constant Time
date: 2023-03-01 10:10:07
categories: "Tech"
tags:
- Research
- CS
- Crypto
- PL
header:
  teaser: 
---

# Intro

Constant Time作为密码学实现上一个重要的property，在抵御side-channel上面有很重要的作用。尤其是在这个Post-spectre的年代，microarchitectural level的side-channel attack已经成为了一个很大的威胁。近年来这个领域的researcher们飙了很多关于constant time的文章，很多都和microarchitectural side-channel相关。其中A哥（Almeida）和B哥(Barthe)尤甚。我还发现这个领域的大佬们似乎有很多欧洲的，可能也和欧洲佬数学好，喜欢搞各种verification有关。

## Abstract Language Model

这个level的工作主要是从理论上解决一些问题。作者首先定义一个abstract language model，然后在此model上面定义可能的side-channel leak, e.g. small-step semantics[PLDI'20](#PLDI'20) 这个model需要能够帮助real-world assembly/IR来

## At IR Level

大多数工作都是在IR level上做verification

## Reference

##### <a name="PLDI'20"></a>Sunjay Cauligi, Craig Disselkoen, Klaus v. Gleissenthall, Dean Tullsen, Deian Stefan, Tamara Rezk, and Gilles Barthe. 2020. Constant-time foundations for the new spectre era. In Proceedings of the 41st ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI 2020). Association for Computing Machinery, New York, NY, USA, 913–926. https://doi.org/10.1145/3385412.3385970
