---
title: RustStudyBasic
date: 2022-02-22 07:16:23
tag:
 - Rust
 - basic
categories:
 - Rust
description: Rust基础学习总结（或者可以说是体会）
cover: https://pic1.zhimg.com/v2-31abd59d97446ec29a8aee9a9284761f_1440w.jpg?source=172ae18b

---
## Rust基础学习总结

### 前言

- 之前浏览公众号的时候看见几篇文章，谈及Rust对前端基础建设具有可观的潜力~~（同时在家里有点无聊）~~，自己也对这方面感兴趣，就从 2.9 开始学习Rust
  - 学习方式：阅读[Rust 程序设计语言](https://kaisery.github.io/trpl-zh-cn/title-page.html) + 手动练习语法
- 2.22 阅读完前19章（除了最后的练习项目以及附录），下文则是**简单的学习总结**
  - 由于之前一直在学习和使用**`JavaScript`**（下文写作JS），所以也使用JS中的语法做比较

## hello_world

### 编程习惯

- Rust对变量、文件、函数等命名规定使用`_`分割单词，例如：`hello_world.rs`，JS没有明说，但大部分人使用驼峰命名，例如：`helloWorld.js`

### 运行环境

- JS不用多说，浏览器是一切的开始（具体可以是chrome的V8引擎），Rust需要从官网安装运行环境，同时自带官方包管理器（Cargo），类比Node和npm

### 语言特征

- Rust和原生JS相反，Rust是静态语言，需要先编译再运行，同时也需要指明类型供编译器检查

## 基本概念

- 大多数编程语言公有的类型，Rust也一样，这些可以略过
  - 数据类型
  - 函数
  - 结构（类比class）
  - 枚举
  - 控制流（Rust的控制流可以玩得比较花）
  - 泛型（静态语言一般都有）
  - 迭代器
  - ...

- Rust最为独特的特性无疑是**`ownership`**（所有权系统）

### 所有权

- Rust对比JS更偏向于底层，JS的垃圾回收自动执行，Rust也是自动执行，但是是基于所有权系统执行
- 所有权系统也保证了Rust的内存安全
- 官方给出的所有权规则

> - Rust 中的每一个值都有一个被称为其 **所有者**（*owner*）的变量。
> - 值在任一时刻有且只有一个所有者。
> - 当所有者（变量）离开作用域，这个值将被丢弃。

### 引用

- 学到这里的第一反应就是C++中的引用（keyword都一样`&`）
- Rust的引用实际上也是在所有权系统之上运行
- 官方给出的引用规则

> - 在任意给定时间，**要么** 只能有一个可变引用，**要么** 只能有多个不可变引用。
> - 引用必须总是有效的。

- 理解引用十分重要，后续Rust代码编写过程中，会处处使用到引用。在我学习过程中，基本可以理解引用为指针（官方也这么类比），一般引用可以是C语言中的**`* const ptr`**（值不可变）

### 模块系统

- Rust有官方的包管理器，模块系统也有自己的一套，不过模块系统实际要解决的问题和其他语言一样，所以容易理解，只是关键字需要替换下

## trait和life_cycle

### trait

- 学习到这里让我想到了红宝书（JavaScript）中讲到`class`提及的组合模式以及设计模式中经典的鸭子类型
- 简而言之，可以为`struct`实现`trait`，那么就可以调用`trait`中的方法（不会关心是什么类型的`struct`）
- 学习完基础的`trait`，可以发现Rust到处都是用到了`trait`，最常用到的打印宏`println!`，里面需要格式化输出时，可以输出的数据类型，都在Rust内部实现了**`std::fmt::Display`**，例如：String、i8-128、u8-128、Vec等

### life_cycle

- 还是由于所有权系统引出的引用所需要的语法，Rust确保不会出现空指针（NULL）（unsafe情况下可能出现），所以函数传入引用参数，返回引用时Rust编译器检查会出问题，可能会出现悬垂引用（类比空指针），Rust为了不让这种情况出现，需要我们指明引用关系（声明周期）
- 读完文档之后，很好理解，除了语法很怪（`fn test<'a>(str1:&'a str)->&'a str`）

## 智能指针

- 和C语言的指针基本一致，学习下来智能最大体现在不需要手动释放（例如：`free(p);`）
- 同时也涉及到了解引用（和C语言一样的keyword：`*`），Rust内部则是使用**`trait`**实现，实际上Rust可以使用**`trait`**实现运算符重载
  - 效果和C语言一致，解引用可以读写指针指向的内存地址上存放的值

- 后续涉及到了引用计数`RC<>`（很像JS的过去的垃圾回收机制，但是Rust是为了实现多个所有权的特殊情况），内部不可变性`RefCell<>`

## 函数式编程

- JS很熟悉的一个概念，我的简单理解就是：**函数可以随便传**（例如：函数当做参数传递给函数）
- Rust也一样，可以利用泛型和`trait`配合，把函数当做参数传递给函数，实现许多高阶功能
- 同时Rust还有闭包（JS中的闭包更偏向于变量机制），Rust主要是可以实现内联匿名函数，写出许多灵活的代码
