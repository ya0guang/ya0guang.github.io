---
layout: single
title: Rust 学习笔记 (To Be Continued)
date: 2021-01-14 14:10:07
categories: "Notes"
tags:
- Rust
- CS
header:
  teaser: https://rustacean.net/assets/rustacean-flat-gesture.svg
---

最近突然对于Rust产生了兴趣~~因为它的吉祥物小螃蟹很可爱~~，遂开始阅读[The Rust PL](https://doc.rust-lang.org/book/)一书想要一探究竟。我发现Rust真的是一门相当阳间的语言，一些functional programming的特性和build system可以让使用者用起来十分舒爽。

虽然目前只看了两章，但我已经从代码中嗅到了了浓郁的Rust的香~~锈~~味。

创建时间：2021-01-09 21:10:07

# Toolchain, Builder and Cargo

Rust的安装还是十分方便舒爽的，[官网](https://www.rust-lang.org/tools/install)提供了非常方便的小脚本来装上它的一整套工具链，包括compiler`rustc`和`cargo`等等。类似`cargo`，`python-pip`这种依赖管理的方式真可谓是十分的阳间，不知道这种天才的idea是谁第一个发明的。Rust作为一个能够做System Programming的语言有这一套Cargo简直可以碾压其他这一领域的爷爷们了。而且还提供了黑魔法一般的Web Based Document，只需要运行`cargo doc --open`即可打开浏览器查看该项目所有的dependencies（在`toml`中指定的版本）的document，真可谓是十足的方便。

对于编译中摸不到头脑的奇奇怪怪的错误，它会告诉你一个错误代码，比如`E0061`。然后跑一下`rustc --explain E0061`，就可以更进一步的了解这货到底是啥玩意，Rust的开发者真是怕你不会用就差开个任意门到你脸上教你写Rust了。这种无微不至对于user的人文关怀真的让人倍感舒适。

# Style

随便放一段第二章中完成的代码感受一下！

```rs
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

# Rust的设定

## Copy

为了防止double free之类的错误，rust对于所有编译时大小未知的变量（以及**没有**implement`Copy` trait的变量）在形如这样的assignment时都是只进行shallow copy的（书中的说法是move）。这样做的后果是在之后所有的`s`都无法被正常使用。为了避免使用runtime gc的性能损耗，Rust在离开一个变量的scope（简单地说就是花括号包住的地方里）的时候就会彻底地释放掉它（在heap上清除）。因此如若`s`和`s2`都能够正常使用但是他们却指向了同一个heap中的地址，那么就会在离开这个scope的时候被double free从而产生安全隐患。因此可以使用`clone`方法来创建deep copy。

```rs
let s = SomeType;
let s2 = s;
```

## Ownership

Rust这个奇葩在传递参数的时候也会传递Ownership。Ownership简单地说就是这个变量属于哪个函数。这种传递是否为deep copy和上面一小节中assignment的规律是一样的。这样就会导致一些奇奇怪怪的问题：
```rs
fn main() {
    let s = String::from("hello");
    takes_ownership(s);

    let x = 5;
    makes_copy(x);
    //若想编译通过，下面这个print不能要
    //否则会有error: value borrowed here after move
    // println!("{}", s);

    println!("{}", x);

    let s = String::from("hello2");
    //
    let s = takes_and_gives_back(s);
    println!("{}", s);
}

fn takes_ownership(some_str: String) {
    println!("{}", some_str);
}

fn makes_copy(some_int: i32) {
    println!("{}", some_int);
}

fn takes_and_gives_back(some_str: String) -> String {
    println!("{}", some_str);
    //return
    some_str
}
```
因为`s`在传递给`takes_ownership`时也会给出ownership，故而在函数return的时候ownership就么的了，然后这个String的memory便会被free掉。而与之相对的，`x`能够在调用完以之为参数的函数后继续使用。

然而，以下代码可以获得正确的输出：
```rs
fn main() {
    let s = "hello";

    takes_ownership(s);

    println!("{}", s);
}

fn takes_ownership(some_str: &str) {
    println!("{}", some_str);
}
```
因为在这里`s`并非是`String`，而是`str`，其是string literal，被hardcoded into text。

### Call by reference

```rs
fn main() {
    let s1 = String::from("hello");

    let len = calculate_len(&s1);

    println!("{}, {}", s1, len);
}

fn calculate_len (s: &String) -> usize {
    s.len()
}
```

Rust似乎并不像C一样在每一次dereference的时候都需要使用到`*`来dereference。[Stackoverflow上的这个问题](https://stackoverflow.com/questions/36335342/meaning-of-the-ampersand-and-star-symbols-in-rust)也部分的解释了我的疑惑：Rust在调用method的时候会自动地加上`&`，`&mut`或`*`来match方法的signature。

### Multiple borrows

如何做到在参数传递的时候不交出ownership呢？可以传递一个reference给函数。如上面的代码写的，`s`在函数`calculate_len`存储了一个指向`s1`的指针。在Rust中，这种参数传递方式称为*borrowing*。

不过以这种方式传递的参数有一个缺点：数据是immutable的。可以使用另一种ref：
```rs
fn main() {
    let mut s1 = String::from("hello");

    let len = calculate_len(&mut s1);

    println!("{}, {}", s1, len);
}

fn calculate_len (s: &mut String) -> usize {
    s.push_str(" world");
    s.len()
}
```

然而，如果两个mutable borrow发生在不同的scope里则是允许的
```rs
let mut s = String::from("hello");

{
    let r1 = &mut s;
}
let r2 = &mut s;
```

Rust为了防止data race，禁止创建两个同样的mutable borrow。immutable borrow可以有无限多个，但是此时如果出现了mutable references的所有use在它的scope里都在一个mutable reference之前的话则可以compile）。感觉这些设定基本都是为了避免多并发时可能出现的一些bug。

## Slice

和python类似，Rust里面的String也可以被slice。这种slice是为了避免变量被drop之后index还存在的尴尬事。

```rs

let s = String::from("hello");
let s1 = &s[0..2];
// let s1 = &s[..2];
```

将之前的功能重新用slice来实现。slice将作为`&str`返回。

```rs
fn main() {
    let s1 = String::from("hel lo");

    let len = first_space(& s1);

    //s1.clear();
    //error because clear() will need a mutable reference to truncate the String

    println!("{}, {}", s1, len);
}

fn first_space(s: &String) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }    
    
    &s
}
```

`fn first_space(s: &str) -> &str`这样的函数原型的参数也可以是`String`。

Slice这种机制也可以运用在其他类型的array中。

# Struct

```rs
struct User {
    username: String,
    email: String,
    active:bool,
    sign_in_count: u64,
}

fn main() {
    let user1 = User {
        email: String::from("eg@eg.com"),
        username: String::from("user1"),
        active: false,
        sign_in_count: 1
    };
}
```
- 结构体可以像这样被定义和实例化。Rust也提供了额外的语法糖来简化一些参数：
    ```rs
    fn init_user(email: String, username: String) -> User {
        User {
            //email: email,
            email,
            //username: username,
            username,
            active: false,
            sign_in_count: 1,
        }
    }
    ```
- Rust还允许从一个结构体创建另一个：
    ```rs
        let user2 = User {
            username: String::from("user2"),
            ..user1
        };
    ```
- **String会被deep copy吗？** 

- Tuple也可以被作为Struct来使用:

    ```rs
    struct Color(i32, i32, i32);
    let blk = Color(0, 0, 0);
    ```
- Struct中data fields的ownership归实例化的变量。
- Struct中可以有对于其他data的reference，但是这要求specify lifetime。
- 可以有不含任何data的struct(unit-like Struct)

## Debug output

```rs
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rec1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!("rec1 is {:#?}", rec1);
}
```
Rust对于Struct的格式化输出可以借助`#[derive(Debug)]`来实现。这种用法有些类似Python中的装饰器。用官方的说法是这里derive了一个`Debug` trait。具体是什么东西还得往后看看才明白。

## Method

```rs
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn can_hold(&self, sub_rec: &Rectangle) -> bool {
        self.width > sub_rec.width && self.height > sub_rec.height
    }

    fn new_square(edge: u32) -> Rectangle {
        Rectangle {
            width: edge,
            height: edge,
        }
    }
}

fn main() {
    let rec1 = Rectangle {
        width: 30,
        height: 50,
    };
    let rec2 = Rectangle {
        width: 20,
        height: 45,
    };

    let square = Rectangle::new_square(30);

    println!("rec1 is {:#?}, area is {}", rec1, rec1.area());
    println!("rec1 can hold re2? {}", rec1.can_hold(&rec2));
}
```

可以implement一个area方法。和Python中的方法类似，这里Struct的方法的第一个参数也是`&self`。因为`area`被实现在`impl Rectangle {}`内部，所以`&self`的类型不需要被指定。Ownership同样也可以在这里被方法拿走。  
和C/C++不同的是，Rust并没有用`->`来调用对象(当其为指针时)的方法或者取对象的成员。Rust会自动添加`&`, `&mut`, `*`来match函数的signature。个人感觉这是一个很方便的东西，而且看上去比C要清爽很多。这个feature被称为*automatic referencing and dereferencing*。  
~~不过这样的话，感觉需要多多在函数定义的时候注意到底参数是什么类型的。~~可能只会给Object(a.k.a self)去自动加？其他的参数似乎并不会。

## Associated Functions

不把`self`作为参数之一的方法。如果我没有记错的话，这种在Python中被称作类方法？Rust中被称为*associated function*。通常被用来return一个新的对象。语法参考上面的代码。

此外Rust还支持对于同一对象的多个`impl`代码块，这似乎是为了trait和泛型设计的。

# Enum

Rust的enum可以按照这种方式来定义：
```rs
enum Message {
    // unit
    Quit,
    // anonymous nested struct
    Move {x: i32, y: i32},
    // String included
    Write(String),
    // 3 * i32 included
    ChangeColor(i32, i32, i32),
}

impl Message {
    fn call(&self) {
        //
    }
}
```


# Other References

- [如何看待 Rust 的应用前景？](https://www.zhihu.com/question/30407715/answer/48032883)里面说的关于Rust的各种features还有待我去体会到
- [用Rust重写Linux内核模块体验](https://zhuanlan.zhihu.com/p/137077998)留用参考