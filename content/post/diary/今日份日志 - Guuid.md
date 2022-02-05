+++
title = "今日份日记 - UUID/GUID的替代品:GUUID"
date = 2021-07-24T12:21:35+08:00
draft = false
archives = 2021
categories = "日记"
tags = ["日记"]
+++

哈，咱最近在写代码的时候，碰到了关于GUID的问题：
```rust
Uuid::parse_str(str).unwrap();
```
这段代码会出现大小端序的问题。解决方案是使用自行定义的struct进行处理后再操作。非常麻烦。

<!--more-->
```rust
#[repr(C)]
pub struct Guid {
    /// The low field of the timestamp.
    a: u32,
    /// The middle field of the timestamp.
    b: u16,
    /// The high field of the timestamp multiplexed with the version number.
    c: u16,
    /// Contains, in this order:
    /// - The high field of the clock sequence multiplexed with the variant.
    /// - The low field of the clock sequence.
    /// - The spatially unique node identifier.
    d: [u8; 8],
}

if Uuid::from_fields_le(
            guid.a.swap_bytes(),
            guid.b.swap_bytes(),
            guid.c.swap_bytes(),
            &guid.d,
        )
        .unwrap()
            == disk_guid{
                // ...
            }
```

所以作者咱萌生了一个自制一个简单的ID格式想法，就叫做GUUID吧^_^。
名字呢，取自
 - `Universally` `Unique` Identifier
 - `Generalized` Unique Identifier

中标记的单词。即Generalized Universally Unique Identifier。

标准也比较简单
```
  -----------------------------------
  |    长度:         128位           |
  |    64 位        |    64 位       |
  |  00002YPM7N3N9  | A5XP0NRR46QB4 |
  |  Unix毫秒级时间戳 | 安全随机数       |
  |  64位无符号整数   | 64位无符号整数  |
  |  大端序储存       | 大端序储存      |
  |  字符串使用Crockford Base32储存   |
  -----------------------------------
```
项目地址: [github](https://github.com/GOSCPS/guuid) [crates](https://crates.io/crates/guuid)
        