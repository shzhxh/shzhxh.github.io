---
title: 6.1内核里的Rust
date: 2023-02-01 13:57:49
categories: Linux
tags:
---

这是一篇翻译文档，源链接：[lwn](https://lwn.net/Articles/910762/)

#### 概述

- Linux6.1内核引入了对Rust语言的支持。
- 6.1内核不运行Rust代码，只是给了内核开发者体验Rust开发的机会。
- 开发者所能做的事是构建一个可以加载到内核里的模块。
- 开发者很可能会觉得内核里的Rust还不够做一些有趣的事。

#### Rust for Linux的相关工作

- 支持代码
- 驱动程序
- Apple图形驱动程序

#### 构建Rust支持

- 参考[quick-start](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/rust/quick-start.rst)以获取构建Rust支持的条件。
- Rust编译器要求是1.62.0。
- bindgen要求是0.56.0。

#### 示例模块

- 条件满足后，内核配置系统将出现CONFIG_RUST选项，可以构建示例模块了。
- 示例模块是samples/rust/rust_minimal.rs
  ```rust
  use kernel::prelude::*;  // 引入一些类型、函数和宏  
  // 用一个宏取代C内核模块里的宏 
  module! {
      type: RustMinimal,  // 实际模块代码的入口
      name: "rust_minimal",
      author: "Rust for Linux Contributors",
      description: "Rust minimal sample",
      license: "GPL", 
  }  
  // 开发人员需要提供RustMinimal的定义 
  struct RustMinimal {
      numbers: Vec<i32>, 
  }  
  // 为RustMinimal实现kernel::Module的trait 
  impl kernel::Module for RustMinimal {
      // 模块初始化，打印2行信息，然后保存一些数字
      fn init() -> Result<Self> {
          ...    } 
  }  
  // 析构函数的实现 
  impl Drop for RustMinimal {
      // 打印一些数字后，模块被移除，内存被释放
      fn drop(&mut self) {
          ...
      } 
  }

#### 展望

- 下一步是添加一些基础设施以与其它内核子系统交互
- 幸运的话，6.2内核里的Rust功能将显著增强
