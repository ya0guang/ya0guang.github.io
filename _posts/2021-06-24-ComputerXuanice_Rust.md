---
layout: single
title: 计算机玄学——主要是踩的Rust的坑
date: 2021-06-24 12:00:07
categories: "Notes"
tags:
- 二进制
- CS
- 逆向工程
- Rust
header:
  teaser: https://hackaday.com/wp-content/uploads/2015/12/rust.jpg?w=640
---

有言道：  
> 我的一生，  
> 是和Rust Compiler搏斗的一生。  
>                           ——我

又有言道：
> 当你觉得Rust出现问题，  
> 不是因为Rust有问题，  
> 而是，你有问题。  
>                           ——我

# CString Ownership

最近在做Teaclave的开发，使用到了很多FFI的玩意。但是CString却给我留下了深刻的印象。

## Problem

```rs
            NativeSymbol {
                symbol: CString::new("teaclave_open_input").as_ptr() as _,
                func_ptr: wasm_open_input as *const c_void,
                signature: CString::new("($)i").as_ptr() as _,
                attachment: std::ptr::null(),
            },
```

上面这段代码创建一个`NativeSymbol` struct，其中`symbol`和`signature`两个C的字符串都在创建结构体的过程中被生成，并得到其指针。  
然而问题就出现在了这里，根据Rust的ownership rule，在离开这个花括号（scope）时，在这个scope内创建的属于这个scope的data会被drop掉。  
换句话说，就是`CString`么了，但是pointer还在。这是什么？这是妥妥的内存dangling pointer啊！最坑爹的地方在于Rust并不会向你报错！~~我是不知道为什么它不报错，但是感觉这个样子是有问题的。~~在Rust里面创建raw pointer是没有问题的，但是dereference它则unsafe。这里对于raw pointer的deref并没有在(safe) Rust里面，而是在unsafe FFI call里面，所以Rust也没有在编译时报错。

## solution

这个问题的解决办法是什么呢？首先是使用String literals：
```rs
            NativeSymbol {
                symbol: b"teaclave_open_input\0".as_ptr() as _,
                func_ptr: wasm_open_input as *const c_void,
                signature: b"($)i\0".as_ptr() as _,
                attachment: std::ptr::null(),
            },
```
String literals在Rust里的lifetime是static的，因此指向其的指针不会被简单地invalidate。

除此之外似乎还可以使用reference(`&`)。但是这个方法我还没有尝试，暂时还不知道具体的实现是怎样的，等我实现了之后再来更新吧。