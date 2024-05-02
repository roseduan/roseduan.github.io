---
title: Rust 练手项目 1 - mini-bitcask
date: 2024-04-29T10:51:56+08:00
categories:
    - Rust
tags:
    - Rust 项目实战
---

之前写过一个 Go 语言的 mini-bitcask，实现了一个基于 bitcask 存储模型的极简 KV 存储引擎。 可以结合之前的文章食用：[https://mp.weixin.qq.com/s/s8s6VtqwdyjthR6EtuhnUA](https://mp.weixin.qq.com/s/s8s6VtqwdyjthR6EtuhnUA)

这次重新用 Rust 实现了一个版本，代码量和之前的差不多，包含了常用的方法，例如 Set、Get、Delete、Scan、PrefixScan、Merge。

![img](https://pic4.zhimg.com/v2-55bf2a18f09297d03ec4ef32958bd2e7_b.jpg)

项目地址：[https://github.com/rosedblabs/rust-practice/tree/main/mini-bitcask-rs](https://github.com/rosedblabs/rust-practice/tree/main/mini-bitcask-rs)

**Set**

```rust
pub fn set(&mut self, key: &[u8], value: Vec<u8>) -> Result<()> {
    let (offset, len) = self.log.write_entry(key, Some(&value))?;
    let value_len = value.len() as u32;
    self.keydir.insert(
        key.to_vec(),
        (offset + len as u64 - value_len as u64, value_len),
    );
    Ok(())
}
```

Set 逻辑比较直观简洁，写入磁盘日志，并且更新内存索引结构。

**Get**

Get 则是先从内存中获取索引，再从磁盘中获取 Value。

```rust
pub fn get(&mut self, key: &[u8]) -> Result<Option<Vec<u8>>> {
    if let Some((value_pos, value_len)) = self.keydir.get(key) {
        let val = self.log.read_value(*value_pos, *value_len)?;
        Ok(Some(val))
    } else {
        Ok(None)
    }
}
```

**Delete**

delete 的逻辑和 Set 类似，只是写入了一个空的值，并且从内存中对应的 key 移除。

```rust
pub fn delete(&mut self, key: &[u8]) -> Result<()> {
    self.log.write_entry(key, None)?;
    self.keydir.remove(key);
    Ok(())
}
```

**Scan**

scan 功能主要借助了 Rust 自带的内存数据结构 BTreeMap 的迭代器进行实现，非常简洁和方便。

```rust
impl<'a> Iterator for ScanIterator<'a> {
    type Item = Result<(Vec<u8>, Vec<u8>)>;

    fn next(&mut self) -> Option<Self::Item> {
        self.inner.next().map(|item| self.map(item))
    }
}

impl<'a> DoubleEndedIterator for ScanIterator<'a> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.inner.next_back().map(|item| self.map(item))
    }
}
```

**Merge**

merge 的逻辑其实也比较简单，将内存中的数据全部重写，并且替换旧的文件即可。

```rust
pub fn merge(&mut self) -> Result<()> {
    // 创建一个新的临时用于用于写入
    let mut merge_path = self.log.path.clone();
    merge_path.set_extension(MERGE_FILE_EXT);

    let mut new_log = Log::new(merge_path)?;
    let mut new_keydir = KeyDir::new();

    // 重写数据
    for (key, (value_pos, value_len)) in self.keydir.iter() {
        let value = self.log.read_value(*value_pos, *value_len)?;
        let (offset, len) = new_log.write_entry(key, Some(&value))?;
        new_keydir.insert(
            key.clone(),
            (offset + len as u64 - *value_len as u64, *value_len),
        );
    }

    // 重写完成，重命名文件
    std::fs::rename(new_log.path, self.log.path.clone())?;

    new_log.path = self.log.path.clone();
    // 替换现在的
    self.log = new_log;
    self.keydir = new_keydir;

    Ok(())
}
```

通过这个简单的项目，可以学习到 Rust 的大多数基础语法，例如：

- 数据类型，数组、整型等
- match 表达式
- 函数
- 结构体
- 错误处理
- 迭代器 Iterator 和 DoubleEndedIterator
- 文件读写操作
- BufWriter 和 BufReader
- 单元测试撰写

项目地址：[https://github.com/rosedblabs/rust-practice/tree/main/mini-bitcask-rs](https://github.com/rosedblabs/rust-practice/tree/main/mini-bitcask-rs)

觉得有帮助的话请不用吝啬你的 Star ⭐️ 哦！
