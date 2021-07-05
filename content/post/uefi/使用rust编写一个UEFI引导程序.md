+++
title = "使用rust编写一个UEFI引导程序"
date = 2021-07-04T22:43:31+08:00
draft = false
archives = 2021
categories = "programming"
tags = ["uefi","rust"]
+++

在本篇文章，咱将一起使用rust实现一个UEFI引导程序！

将加载一个简单的内核。

<!--more-->

首先，咱们需要创建一个新项目:
```shell
 $ cargo new hello_uefi
```
咱们还需要使用x86_64-unknown-uefi这个target。不过一般不需要安装。

接着，咱们设置咱们的`cargo.toml`。
```toml
[package]
name = "hello_uefi"
version = "0.1.0"
edition = "2018"

[profile.dev]
# 禁用栈展开
panic = "abort"

[profile.release]
# 同上
panic = "abort"

[dependencies]
# 引入uefi
uefi = "0.11.0"
```
然后编写`main.rs`
```rust
// 禁用自带的东西
// 同时启用一些feature
#![no_std]
#![no_main]
#![feature(abi_efiapi)]
#![feature(panic_info_message)]
#![feature(alloc_error_handler)]

use core::panic::PanicInfo;

extern crate alloc;

// 载入uefi定义
use core::fmt::Write;
use uefi::prelude::*;
use uefi::CStr16;

// 全局系统表
pub static mut IMAGE_HANDLE: Option<Handle> = None;
pub static mut IMAGE_SYSTEM_TABLE: Option<SystemTable<Boot>> = None;

/// 入口函数
#[no_mangle]
pub extern "C" fn efi_main(handle: Handle, system_table: SystemTable<Boot>) -> Status {
    // 初始化全局变量
    unsafe {
        IMAGE_HANDLE = Some(handle);
        IMAGE_SYSTEM_TABLE = Some(system_table);
    }

    // 什么都不干
    loop {}
}

/// panic擦屁股函数
#[panic_handler]
fn panic(panic_info: &PanicInfo) -> ! {
    // 什么都不干
    loop {}
}
```
我们再编写一个脚本以构建此项目：
```shell
cargo build --target=x86_64-unknown-uefi -Z build-std=core,compiler_builtins,alloc -Z build-std-features=compiler-builtins-mem
```
接下来，我们应该就可以编译我们的第一个uefi程序了！

不出所料，我们应该得到了一个错误:
```
error: no global memory allocator found but one is required; link to std or add `#[global_allocator]` to a static item that implements the GlobalAlloc trait.
```
看起来，我们并没有给这个程序添加内存分配器。解决方案也都不难:
 - 添加一个内存分配器，鉴于UEFI已经提供了非常简陋的内存"管理"，我们将使用这个方案。

 - 删除源码中的`extern crate alloc;`。

下面是内存分配器的源码:
```rust
use core::alloc::{GlobalAlloc, Layout};
use uefi::table::boot::MemoryType;

/// UEfi内存分配器
pub struct UefiAllocator;

/// 实现内存分配器
unsafe impl GlobalAlloc for UefiAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let boot = IMAGE_SYSTEM_TABLE.as_ref().unwrap().boot_services();

        // 内存类型为LOADER_DATA(即数据)
        boot.allocate_pool(MemoryType::LOADER_DATA, layout.size())
            .unwrap()
            .unwrap()
    }

    unsafe fn dealloc(&self, ptr: *mut u8, _layout: Layout) {
        let boot = IMAGE_SYSTEM_TABLE.as_ref().unwrap().boot_services();

        boot.free_pool(ptr).unwrap().unwrap();
    }
}

/// 设置内存分配器
#[global_allocator]
static GLOBAL: UefiAllocator = UefiAllocator;

/// 设置分配error处理函数
#[alloc_error_handler]
fn alloc_error_catch(layout: core::alloc::Layout) -> ! {
    panic!(
        "Couldn't alloc memory! Size:{0} Align:{1}",
        layout.size(),
        layout.align()
    );
}
```
UEFI中的allocate_pool()内存以8字节对齐，我们不需要再关心此事（8字节是rust的最大对齐）。


我们现在应该可以顺利通过编译。

下面，咱们就可以开始进行测试啦！

有一个名为`uefi-run`的rust的工具可以帮到我们，install at once:
```shell
 $ cargo install uefi-run
```
用法也很简单:
```shell
 $ uefi-run -b 你的OVMF.fd所在的地址  -q 你的qemu可执行文件的地址，通常是qemu-system-x86_64  你的ELF文件所在的地址 -- 你的QEMU参数列表...
```
OVMF.fd可以从一些Linux发行版的包管理器中找到。
也可以从[这里](https://github.com/tianocore/tianocore.github.io/wiki/OVMF)找到一些有用的信息。

不出意外，我们的程序应该没什么动作。

下面，我们应该可以直接加载内核了。

## TODO
