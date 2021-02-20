---
layout: single
title: Rust 学习笔记 (To Be Continued)
date: 2021-02-20 11:10:07
categories: "Notes"
tags:
- Rust
- CS
header:
  teaser: https://rustacean.net/assets/rustacean-flat-gesture.svg
---

最近突然对于Rust产生了兴趣~~因为它的吉祥物小螃蟹很可爱~~，遂开始阅读[The Rust PL](https://doc.rust-lang.org/book/)一书想要一探究竟。我发现Rust真的是一门相当阳间的语言，一些functional programming的特性和build system可以让使用者用起来十分舒爽。

虽然目前只看了~~两~~十章，但我已经从代码中嗅到了了浓郁的Rust的香~~锈~~味。

创建时间：2021-01-09 21:10:07

# Prologue

## 为什么我要学Rust

经过了一段时间的使用，我发现Rust娘的脾气如下：

- 哪里不对点哪里
  - Rust不像一个喜欢无理取闹的女朋友一样，觉得你做错了还憋着不说，硬是让你问：“我这里是不是做的不对？我那里是不是做错了？”；
  - C这种女朋友即使你觉得自己哪哪都没错，她也不说你做错了，但是偏偏就是怎么也不work；
  - 但是Rust不一样的地方在于，会明确的指出你哪里惹她不开心了~~虽然她会经常经常不开心~~；
  - 即使Rust娘对你不爽了，她也不会弃你不顾，而是会贴心的给出各种help，陪你解决问题~~实际上更多时候可能是谷哥解决的~~。
- 牛逼的各种属性
  - Closure，虽然似乎对于生成closure的参数的生命周期有一定的限制
  - `match`: 帮你不漏过每一个小角落？
- 语法相对自由，检查绝壁严格

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
可以看出其支持在enum里面的不同variants有着不同的nested data types。这倒是一个十分方便的feature。

## Option

为了避免C/C++中`NULL`引起的一些问题，例如null pointer dereference之类的，Rust引入了Option枚举类。
```rs
enum Option<T> {
    Some(T),
    None,
}
```
这有些类似于functional programming中monad的思想。这样便允许了可以为空的fields。

那么如何把Option里面的东西取出来呢？那就要用到牛逼的match了
```rs
fn main() {
    let x = Some(3);
    let x = plus_one(x);
    println!("x is {:?}", x);

    let y = None;
    let y = plus_one(y);
    println!("y is {:?}", y);
}

fn plus_one(num: Option<i32>) -> Option<i32> {
    match num {
        Some(v) => Some(v+1),
        None => None,
    }
}
```

PS：在match里面，`_`可以去match任意值。也可以用`if let`来match一种case，这个时候Rust不会check是否有其他的case没有被match到。
```rs
if let Some(3) = num {
    println!("Lucky!");
}
```

# Rust包 

- 通过`cargo new --lib [libname]`来创建一个Rust的lib
- 在`src/lib.rs`下面可以通过`mod`关键字创建多个module
- module之间可以嵌套，可以包含enum, struct, function，这些都可以是public的
- 所有的`fn`, `mod`, `enum`, `struct`都是默认private的，父类无法直接访问到子类但是反之则可
- 可以使用相对或者绝对“路径”来访问module里面的内容
- `impl`里面的东西可以access被impl的对象的private field
- `use`基本等同于Python中的`import`，也可以使用`use A as a`
- `pub use`可以被用来提升命名空间？[ref](https://tonydeng.github.io/2019/10/28/rust-mod/)
- 也有`use [namespace]::*`的用法，和Python类似
- nested path
  Rust提供了一个语法糖：
  ```rs
  // original
  use std::cmp::Ordering;
  use std::io;
  use std::io::Write;

  // nested
  use std::{cmp::Ordering, io};
  use std::io::{self, Write};
  ```

## 分割到不同文件


Rust通过把module里面的sub module放在root下的其他文件/文件夹中来完成module的嵌套。Rust的文件名即是其所在module的名字。

以下为在单个文件里面的module结构：
```rs
//#[cfg(test)]
mod back_of_house {

    // #[derive(Debug)]
    pub struct Breakfast {
        pub toast: String,
        seasonal_fruit: String,
    }

    impl Breakfast {
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast {
                toast: String::from(toast),
                seasonal_fruit: String::from("peaches"),
            }
        }
    }
}
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}

        fn seat_at_table() {}

    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}

use crate::front_of_house::hosting;
// or use relative path
// use self::front_of_house::hosting;

pub fn eat_at_restaurant() {

    // abosulute path
    crate::front_of_house::hosting::add_to_waitlist();
    // relative path
    front_of_house::hosting::add_to_waitlist();
    // effective after use
    hosting::add_to_waitlist();

    let mut meal = back_of_house::Breakfast::summer("Rye");
    meal.toast = String::from("Wheat");
    println!("{} toast pls!", meal.toast);
}
```

其中，`front_of_house`可以通过这种重构实现：

- `src/lib.rs` root
```rs
mod front_of_house;
```

- `src/front_of_house.rs`
```rs
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```

或者可以对`hosting`进行更进一步的拆分：
- `src/front_of_house.rs`
```rs
pub mod hosting;
```

- `src/front_of_house/hosting.rs`
```rs
pub fn add_to_waitlist() {}
```

# Rust Collections

Rust提供了一堆方便的数据结构。

## Vector

这里的Vector其实基本上类似于可变长度的数组，定义与Pie language里面的vector比较类似。  
可以通过`vec!`宏或者`Vec::new()`初始化。  
因为vector内部的实现问题，mutable vector在长度变化的时候有可能会被挪动到别的内存（在vector变长的时候原本malloc的内存可能装不下），因此vector也遵循同一scope内不能有mutable和immutable reference的规则，即使immutable reference指向vector初始的几个元素。

还可以用引用和`get()`方法来拿到vector中的元素。其处理的方式会有些许不同：
```rs
fn main() {
    let mut v = Vec::new();

    v.push(5);
    v.push(6);

    // panic if out of bound
    let third = &v[2];

    // will get an Option<T>
    match v.get(1) {
        Some(val) => println!("get {}", val),
        None => println!("not even get!"),
    }
}
```

### Iterate a vector

和Python的语法类似，和C的语义类似，可以用指针来迭代vector：
```rs
    for i in &mut v {
        *i += 10;
    }
```
但是如若像这样使用迭代器则会报错：
```rs
    for mut i in v {
        i += 10;
    }

    for j in v {
        println!("{}", j);
    }
```
报错：
```shell
error[E0382]: use of moved value: `v`
   --> src/main.rs:12:14
    |
2   |     let mut v = Vec::new();
    |         ----- move occurs because `v` has type `Vec<i32>`, which does not implement the `Copy` trait
...
8   |     for mut i in v {
    |                  -
    |                  |
    |                  `v` moved due to this implicit call to `.into_iter()`
    |                  help: consider borrowing to avoid moving into the for loop: `&v`
...
12  |     for j in v {
    |              ^ value used here after move
    |
note: this function consumes the receiver `self` by taking ownership of it, which moves `v`
```
这是因为在把给了迭代器的时候传递了ownership，导致v在后续的scope内失效了。看来如无必要迭代器还是得borrow。

如若想要把不同类型的元素存储在同一个vector里面，可以用它来存储一个enum类型，这个enum类型里面包括不同类型。  

## String

和之前的代码中出现的api类似，可以通过`::new()`, `::from()`, `.to_string()`等方法创建一个`String`。  
`String`的api还包括`push_str()`, `push()`, `+`运算符的重载。要说明的是这些运算并不会take ownership。
还有`format!()`宏可以使用：
```rs
let s1 = String::from("tic");

let s = format!("{}-tok", s1);
```
`s`也是一个`String`。

Rust中String其实是`Vec<u8`。但是其支持utf-8编码，因此Rust并不能像C一样直接index一个String。因此可以使用slice的方式去拿sub elements，但如若卡在了char boundary上面的话会报错。

可以用`.chars()`或者`.bytes()`方法来生成一个可迭代的对象，前者会划分为一个个unicode，而后者则是`u8`。

## Hashmap

看如下代码，`HashMap`不像vector一样有用来初始化的宏，但是它可以从两个vector来初始化。
```rs
use std::collections::HashMap;

fn main() {
    let mut scoreboard = HashMap::new();

    scoreboard.insert(String::from("A"), 20);
    scoreboard.insert(String::from("B"), 30);

    println!("{:?}", scoreboard);

    let teams = vec![String::from("A"), String::from("B")];
    let scores = vec![20, 30];

    // must specify the type here
    let mut scoreboard: HashMap<_, _> = teams.into_iter().zip(scores.into_iter()).collect();
    println!("{:?}", scoreboard);

    // get an Option<T> here
    let team = String::from("A");
    let s = scoreboard.get(&team);
    // euivalent to
    let s = scoreboard.get("A");
    println!("{:?}", s);

    for (k, v) in &scoreboard {
        println!("{}: {}", k, v);
    }

    // insert if the Key has no value
    scoreboard.entry(String::from("A")).or_insert(50);
    scoreboard.entry(String::from("C")).or_insert(50);

    // update the data
    let score = scoreboard.entry(String::from("A")).or_insert(0);
    *score += 100;
    println!("{:?}", scoreboard);
}
```

需要注意的是，`HashMap`的key值不能重复，如果重复的话则会update现有的key对应的value。那么便可以用`entry().or_insert`来检查是否存在，如若不存在则插入。  
这里`entry()`是一个蛮神奇的存在，其会返回一个`Entry`。如果key存在则会返回一个mutable reference，如果不存在则会创建一个mutable reference并返回。  
也可以通过`entry()`来获得对于`HashMap`里面一个slot的mutable reference，从而进一步更新其中的数据。

# Error

和其他语言不同的是，Rust并没有Exception handling这种机制。其将错误分为recoverable和unrecoverable。前者通过`Result<T, E>`来处理，而后者则会在调用`panic!()`宏的时候出现。  
这个宏可以主动调用来使程序panic，还可以在运行时设置`RUST_BACKTRACE=1`来在程序panic的时候backtrace。

Result的定义如下：
```rs
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

用`match`来对result处理是Rust的常规操作。骚操作呢？也不是没有：`unwrap_or_else()`，后面会讲到，先卖个关子。如果懒得用match的话可以直接`unwrap()`。这样如果没有错误便会直接返回`Ok`里面夹着的东西，反之则panic。也可以用`expect()`来在其参数里面填上其他的描述来实现更友好的错误输出，`except()`除了接收一个参数外和`unwrap()`一毛一样。

```rs
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("./src/hello.txt");
    let f = match f {
        Ok(file) => file,
        Err(err) => match err.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(f) => f,
                Err(err) => panic!("create file failed {:?}", err),
            },
            other => {
                panic!("unexpected error: {:?}", other)
            }
        },
    };
    println!("open file successfully! {:?}", f);
}
```

## 错误传递

```rs
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");

    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut s = String::new();

    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}
```
这段简单的代码可以返回一个`Result`。而Rust提供了一个更简单的操作符, `?`~~？？？~~。用这个操作符，函数可以写成：
```rs
    let mut f = File::open("hello.txt")?;

    let mut s = String::new();

    f.read_to_string(&mut s)?;

    Ok(s)
```
`?`所做的事情几乎和之前写的match一毛一样，区别在于它会调用standard library里面的`From` trait来把错误的类型转换为函数定义的return type中的错误类型。（*难道`matcch`在return的时候没有做类型转换吗？*）但是这里还有一处改变，`f`被定义为了mutable。这里令我十分无法理解，为什么在前面的code中immutable `f`可以正常运行？？？？`read_to_string`的定义是`fn read_to_string(&mut self, buf: &mut String) -> Result<usize>`，很显然需要一个mutable self。但是这tm居然能运行？？？我十分**不解**这是为什么。

**破案了，因为上一个f被shadow了。。。我没发现f被定义了两次！！！**

不过上面的任务有一个更简单的写法：
```rs
    fs::read_to_string("hello.txt")
```
这里应该是使用了类似于类方法的东西，并不需要我们创建一个File就可以调用类的方法了。但是这个signature是这样的：`pub fn read_to_string<P: AsRef<Path>>(path: P) -> io::Result<String>`，可见返回值并不是像之前看到的`Result<T, E>`一样。因为这里面不知道为啥没法直接看到它的定义，但是猜测是封装了后者。

# Generic Types, Traits, and Lifetimes

泛型以及其相关内容。本章会讲述形如这样的代码到底说了什么：

```rs
use std::fmt::Display;

fn longest_with<'a, T>(x: &'a str, y: &'a str, ann: T) -> &'a str
where
    T: Display,
{
    println!("announcement {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

## Generic Types

### Struct
```rs
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

impl Point<f32> {
    fn distance(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

struct PowerfulPoint<T, U> {
    x: T,
    y: U,
}

impl<T, U> PowerfulPoint<T, U> {
    fn mixup<V, W>(self, other: PowerfulPoint<V, W>) -> PowerfulPoint<T, W> {
        PowerfulPoint {
            x: self.x,
            y: other.y,
        }
    }
}
```
这里实现了结构体内的泛型。枚举类的泛型类似。有几点需要说明：
- 如果要实现泛型结构体的函数，`impl`后面要加上泛型参数`<T>`
- 那么也可以实现非泛型结构体的函数：为某一特定类型实现
- 实现的函数也可以具有不同与泛型结构体的泛型参数，如`PowerfulPoint`的实现
- 注意`x()`在这里的返回是一个引用。如果不是引用则会报错：
  - `cannot move out of `self.x` which is behind a shared reference`
  - 其实应该是有办法的，如果`T` implement了`Copy`这个trait的话。但是Rust现在不知道有没有这个trait，所以我们姑且先返回引用


### Function

将这个函数变成泛型的版本：
```rs
fn largest(list: &[i32]) -> &i32 {
    let mut largest = &list[0];

    for n in list {
        if n > largest {
            largest = n;
        }
    }
    largest
}
```
然而简单地加上泛型参数并不能使之编译：
```rs
fn largest<T>(list: &[T]) -> &T {
    let mut largest = &list[0];

    for n in list {
        if n > largest {
            largest = n;
        }
    }
    largest
}
```
这是因为并非所有的type都可以去比大小。这个`T`必须实现了偏序的trait。  
改写成如下形式能够编译通过：
```rs
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &n in list {
        if n > largest {
            largest = n;
        }
    }
    largest
}
```
这里的语法细节将在下面进一步讨论。

### 编译

Rust的泛型是通过在编译时把不同使用到的泛型都编译出来实现的，所以性能开销很小。比如`foo<T>()`如果被传入了`i32`和`u8`，那么其在编译时便会被转变成两个函数：`foo_i32()`和`foo_u8()`，从而减少泛型引起的性能开销。

## Trait

简单地说trait就是让编译器知道一些类型有着一些共有的功能。Rust说这货和接口比较像。

### Define a trait

这里我们定义一个叫做`Summary`的trait。注意函数的Signature以`;`结尾。一个trait里面可以有多个函数，而每个类型若想实现一个trait必须自己拥有每一个trait里面的函数的实现。
```rs
// src/lib.rs
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct News {
    pub headline: String,
}

impl Summary for News {
    fn summarize(&self) -> String {
        format!("{}", self.headline)
    }
}

pub struct Tweet {
    pub username: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}", self.username)
    }
}

// src/main.rs
mod lib;
use lib::Summary;

fn main() {
    println!("aaa");
    let n = lib::News{
        headline: String::from("headline_A"),
    };

    println!("{}", n.summarize());
}
```

Rust的trait遵循coherence property (orphan rule)，即trait的implementation要么在trait里，要么在类里（类或者trait的crate是local的）。反之，假设在一个奇奇怪怪的地方（e.g.自己的lib）定义了比如`Vec<T>`实现了`Display`，但是这俩玩意都是来自stdlib里面的东西，对于自己的lib而言都是extern的，那么这样是不行的。这样做是为了防止trait被重复实现。除此之外也有安全的考虑：如果这些extern的trait和类可以被重新实现，那么即意味着任何在项目中使用的crate都可以**劫持**这些实现。

若要使用刚刚写的类和trait，可以切换到`main.rs`下面去先`mod lib`，然后`use lib::Summary`。注意这里`lib`其实可以是任何模块名（文件名），只要和src目录下的rs源文件名相同即可。而若要调用`summarize()`这个来自`Summary` trait的函数，则必须use它。

如同接口中所能做的一样，也可以将trait实现：
```rs
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("Unimplemented Summmary")
    }
}

pub struct News {
    pub headline: String,
}

impl Summary for News {
}
```

这里在调用`news.summarize()`时则会直接找trait定义里面的实现。但是必须`impl Summary for News`才可能进行这个调用，即使`impl`里面为空。这也有些类似类里面（虚）函数的重载。

```rs
mod lib;
use lib::Summary;

// also works here if fn notify(item: &Summary)
fn notify(item: &impl Summary) {
    println!("Notification: {}", item.summarize());
}

fn main() {
    println!("aaa");
    let n = lib::News{
        headline: String::from("headline_A"),
    };

    notify(&n);
}
```

- 定义在trait里面的函数可以调用这个trait中的其他函数，即使后者没有在trait里面实现
- 可以定义函数其参数必须实现一个trait从而不指定具体的类型
  - 上面代码函数signature中`impl`其实是语法糖
  - 原型应为：`fn notify<T: Summary>(item: &T)`
  - **可以要求参数同时实现多个trait吗？**可以，使用`&(impl Trait1 + Trait2)`
- 也可以要求返回值它实现了trait
- 当函数泛型有比较多的trait的约束时，可以用`where`关键字使其更清晰
```rs
fn some_fn<T, U>(t: &T, u: &U) -> impl Summary
    where T: Summary + Display,
          U: Summary
{
    lib::News{
        headline: String::from("headline_A"),
    }
}
```
目前`some_fn`只能return一种类型。即：它没法return `News`或者`News2`，即使两者都实现了`Summary`。这与编译的实现有关。后面会讲到如何让它可以return不同类型的变量。我猜测这里是因为如果return了多种变量的话，编译器不知道在后续接收这个对象的代码中以何种方式调用对象的方法，因为**似乎**Rust中的trait并非是像虚函数那样实现的。

一开始的`largest`函数也可以不需要`Copy`或者`Clone`这样的trait。这里它可以只返回一个slice：
```rs
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];

    for n in list {
        if *n > *largest {
            largest = &n;
        }
    }
    largest
}
```

### Conditionally Implemented Method

我们也可以完成这个目标：当类中的泛型实现了某（些）trait时，为这个类实现一个方法。语法如下：
```rs
impl<T: Display + PartialOrd> Some_type<T> {
    fn some_fn(&self) {
        // do something
    }
}
```
这里，当且仅当`Some_type<T>`中的`T`类型实现了`Display`和`PartialOrd`两个trait时，才为其实现`some_fn`方法。

## Lifetime

Rust和其他语言最不一样的地方应该属lifetime了。这里的lifetime指的是变量的lifetime。由于默认情况下所有变量在离开自己的scope之后便会被销毁，有些时候可能需要一些变量live longer。

```rs
fn main() {
    let r;
    {
        let x = 5;
        r = &x;
    }
    println!("{}", r);
}
```
在这段代码中，尝试编译会报错：

```
error[E0597]: `x` does not live long enough  
 --> src/main.rs:5:13  
  |  
5 |         r = &x;  
  |             ^^ borrowed value does not live long enough  
6 |     }  
  |     - `x` dropped here while still borrowed  
7 |     println!("{}", r);  
  |                    - borrow later used here  
```

若想要实现和引用相关的函数，有时候也需要显示标注lifetime:
```rs
fn main() {
    let s1 = String::from("str1");
    {
    let s2 = "s2";
    let result = longest(&s1, s2);
    println!("{}", result);
    }

}

// fn longest (s1: &str, s2: &str) -> &str
// rasie error: expected named lifetime parameter
fn longest<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() {
        s1
    }
    else {
        s2
    }
}
```
这里因为`longest`的参数`s1`和`s2`可能拥有不同的lifetime，导致return之后编译器不会玩了（不知道什么时候需要销毁变量）。因此我们这里需要显示的标注他们的lifetime。这里标注的意义是，`s1`和`s2`overlap的部分（或者说较小的那个？）即是return value的lifetime。标注之后，即使是如下的调用也是ok的：
```rs
fn main() {
    let s1 = String::from("str1");
    {
        let s2 = "s2";
        let result = longest(&s1, s2);
        println!("{}", result);
    }
}
```
因此，这样的调用会报错：
```rs
fn main() {
    let s1 = String::from("str1");
    let result;
    {
        let s2 = String::from("s2");
        result = longest(&s1, &s2);
    }

    println!("{}", result);
}
``` 
lifetime类似于泛型参数，一般以`'` + 小写字母表示。类似类型，lifetime是不可以被更改的。

对于结构体和方法也可以标记lifetime：
```rs
struct Strstr<'a> {
    part: &'a str,
}

impl<'a> Strstr<'a> {
    fn level(&self) -> i32 {
        42
    }
}
```

实际上我们在真正写代码的时候并不需要那么多的annotations，这是因为Rust编译器引入了一些规则来帮助自动标记lifetime
1. 所有的参数的lifetime都会被标记成不同的
2. 如果只有一个参数，那么其返回的lifetime会被标记成和参数相同的
3. 如果方法有`self`之类的参数，那么返回的lifetime和`self`的相同

### `'static`

除了上述的标记之外，还有一种特殊的lifetime标记：`'static`。其意味着可以活到程序的终结。所有的string literal都具有这个lifetime。

# Test

Rust提供了完善的测试功能。对于library，在创建的时候便会在`lib.rs`里面生成test module。类似：
```rs
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        assert_eq!(2 + 2, 4);
        assert!(true);
    }

    #[test]
    fn another() {
        panic!("Fail!")
    }
}
```
在需要作为测试函数的函数头上标记`#[test]`，然后运行`cargo test`即可自动去做test。而在module上面标记
`#[cfg(test)]`可以实现对于一个模块的unit test。

除此之外，Rust还可以做panic的捕获，以及输出更加详细的测试信息。

# Functional Rust

终于要快进到魔法Rust了。来这里体验一把Rust强大的函数式特性。

# Other References

- [如何看待 Rust 的应用前景？](https://www.zhihu.com/question/30407715/answer/48032883)里面说的关于Rust的各种features还有待我去体会到
- [用Rust重写Linux内核模块体验](https://zhuanlan.zhihu.com/p/137077998)留用参考