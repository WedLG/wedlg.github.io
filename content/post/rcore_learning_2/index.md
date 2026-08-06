---
title: rCore Lab学习 第二部分
draft: false
date: 2026-08-06T21:32:08+08:00
tags:
  - rust
  - 操作系统
  - rCore
categories:
  - Rust学习
  - 操作系统
  - rCore
image: "https://picsum.photos/800/600?random=1786023128"
---

## 前言

作为本系列的第二部分，好久没写了。
本人目前在通过rCore Lab复习相关的操作系统知识。
如果有写的不好的地方烦请指出，谢谢！。

>[!TIP] 有关rCore Lab: [rCore教程 - v3](https://rcore-os.cn/rCore-Tutorial-Book-v3/index.html)

## 有关Rust宏编程的记录

在熟悉Rust的时候在宏编程上栽了坑。在此记录一下。有关Rust宏的具体定义看[这里](https://rustwiki.org/zh-CN/reference/macros-by-example.html)

### 有关某题

记录本题。(题目来源是这里：[Rust Quiz](https://dtolnay.github.io/rust-quiz/1))

题目如下，分析下列程序并判断输出。

```rust{.line-numbers}
macro_rules! m {
    ($($s:stmt)*) => {
        $(
            { stringify!($s); 1 }
        )<<*
    };
}

fn main() {
    print!(
        "{}{}{}",
        m! { return || true },
        m! { (return) || true },
        m! { {return} || true },
    );
}
```

首先来分析m宏在做什么。由上述代码可知，m宏接受若干**表达式（Statement，简写为stmt）** 作为参数，即匹配部分

```rust
$($s:stmt)*
```

而后在内部（宏展开部分）有一个宏重复语句

```rust
 $({ stringify!($s); 1 })<<*
```

宏展开表达式``$()<<*``的作用是，对于第一个匹配到的表达式，会放入分隔符``<<``的前方括号内，而后所有的表达式，表达式和表达式之间都使用分隔符``<<``连接。
该语句的含义是，在前边匹配到的第一个语句会在``<<``前面的块表达式中展开，
并作为宏``stringify!()``的参数，该宏会将捕获的语句转化成字符串。
该块表达式之后返回1。若有多个表达式被捕获，则会连接到``<<``的后方。例如
现在有表达式stmt1、stmt2、stmt3作为m宏的参数，则最终替换的结果会变为

```rust
{ stringify!(stmt1); 1 }<<stmt2<<stmt3
```

接着来逐个分析main函数中有关宏m的语句。对于原代码片段，
第12行为一个表达式``return || true``，
该表达式返回一个不带参数的lambda表达式``|| true``，
此lambda表达式返回true。
故而宏展开表达式变为

```rust
{stringify!(return || true);1}
```

第13行语句中``(return) || true``的return部分被括号包裹，因而
在实际执行的过程中会先执行括号内的语句，因而是一个括号表达式，
此时的``||``是*逻辑或操作符*，故而这一句是一个逻辑表达式。

第14行语句中``{return} || true``的return部分被块表达式标识符包裹，
因而是一个*块表达式*，这导致后半部分的``|| true``也变为了一个块表达式，
因而这是两个表达式，此时宏展开后变为

```rust
{stringify!({return});1}<<{|| true}
```

上述展开式中，``<<``的前半返回1，后半返回true（最后被视为1），因而运算结果
为`1<<1=2`。

### 有关print宏的实现

在rCore中，有关print宏的实现如下

```rust{.line-numbers}
// os/src/console.rs
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}
```

其中，第一行的属性`#[macro_export]`用于表明该宏可以导出到包外使用。重点分析print的宏参数（第4行和第5行）。

第一个宏参数`fmt`为literal类型，即*字符串字面量*。
而对于第二部分`$(, $($arg: tt)+)?`,其中`arg`为tt（Token Tree）类型，可以用于匹配任何Token树。有关Token Tree参考[这里](https://doc.rust-lang.net.cn/proc_macro/enum.TokenTree.html)。
这部分参数表示后续若有*一个逗号*，则必须要跟上*至少一个Token Tree*。
对于第5行的输出部分同理。

## 有关QEMU

这一部分记录做实验过程中有关QEMU的一些坑。

QEMU虚拟机在运行的时候会将终端的所有权进行抢夺。在运行本实验的内核时，
若想要关闭内核，需要通过`Ctrl+A`的快捷键进入QEMU虚拟机，而后再按下`X`
进行关闭。
