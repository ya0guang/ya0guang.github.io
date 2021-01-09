---
layout: single
title: Rust 学习笔记 (To Be Continued)
date: 2020-01-09 21:10:07
categories: "Notes"
tags:
- Rust
- CS
header:
  teaser: https://rustacean.net/assets/rustacean-flat-gesture.svg
---

最近突然对于Rust产生了兴趣~~因为它的吉祥物小螃蟹很可爱~~，遂开始阅读[The Rust PL](https://doc.rust-lang.org/book/)一书想要一探究竟。我发现Rust真的是一门相当阳间的语言，一些functional programming的特性和build system可以让使用者用起来十分舒爽。

虽然目前只看了两章，但我已经从代码中嗅到了了浓郁的Rust的香~~锈~~味。

# Toolchain, Builder and Cargo

Rust的安装还是十分方便舒爽的，[官网](https://www.rust-lang.org/tools/install)提供了非常方便的小脚本来装上它的一整套工具链，包括compiler`rustc`和`cargo`等等。类似`cargo`，`python-pip`这种依赖管理的方式真可谓是十分的阳间，不知道这种天才的idea是谁第一个发明的。Rust作为一个能够做System Programming的语言有这一套Cargo简直可以碾压其他这一领域的爷爷们了。而且还提供了黑魔法一般的Web Based Document，只需要运行`cargo doc --open`即可打开浏览器查看该项目所有的dependencies（在`toml`中指定的版本）的document，真可谓是十足的方便。

对于编译中摸不到头脑的奇奇怪怪的错误，它会告诉你一个错误代码，比如`E0061`。然后跑一下`rustc --explain E0061`，就可以更进一步的了解这货到底是啥玩意，Rust的开发者真是怕你不会用就差开个任意门到你脸上教你写Rust了。这种无微不至对于user的人文关怀真的让人倍感舒适。

# Style

随便放一段第二章中完成的代码感受一下！

```Rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    println!("Guess the number!");

    // 这里用到了rand库中更新版本的API,原书中`gen_range`是take两个参数的。
    // rand = "0.8.1"
    let secret_number = rand::thread_rng().gen_range(1..101);
    println!("The secret number is: {}", secret_number);

    loop {
        println!("Please input your guess");
        let mut guess = String::new();
        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };
        // expect("Please type a number");
        println!("You guessed: {}", guess);
        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}
```

简单说一下
- `mut` 来区分mutable和immutable的变量
- 一个变量名可以被shadow：这个名字所对应的新的变量可以拥有一个不同的类型和值
- `match`这种functional programming中流行的玩意似乎真的是可以玩出花来，并且字里行间都有函数式编程的形状
- 函数调用返回`Ok`或者`Err`，感觉是比较麻烦的事情，但确实更加安全了
- `::`还是相当多的
- Rust是statically typed lang，但是可以看出来它的type inferencer似乎相当强大

# Other References

- [如何看待 Rust 的应用前景？](https://www.zhihu.com/question/30407715/answer/48032883)里面说的关于Rust的各种features还有待我去体会到
- [用Rust重写Linux内核模块体验](https://zhuanlan.zhihu.com/p/137077998)留用参考