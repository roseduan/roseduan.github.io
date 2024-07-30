---
title: 近期对 wal 组件的性能提升
date: 2024-06-15T16:51:56+08:00
categories:
    - 开源
tags:
    - 开源项目
---

## wal 的由来

wal 是我去年写的一个小组件，主要用于 LSM Tree 或者 Bitcask 的预写日志文件，以及任意的 append-only 文件读写都可以使用，第一次发布是 2023.6.13，刚好开源一年了：

![img](https://pic3.zhimg.com/80/v2-67afdeac25522133bfb2f3056d3848c6_1440w.webp)


rosedb 和 lotusdb 将其作为重要的底层日志文件存储组件使用，这个通用的组件简化了 rosedb 和 lotusdb 的一部分代码，使项目整体更加简洁。
一年过去了，wal 同时也被很多其他的开源/闭源项目所使用（生产环境），对这个小组件我还是比较满意的，整体代码的质量还不错，代码理解起来也比较简单。

之前写过一篇文章，简单介绍了如何使用 wal 构建一个极简的 KV 存储模型，以及我还在 Go 夜读社区分享过关于 wal 的设计和实现，结合这些资料看懂源代码应该没有什么困难。
[使用 wal 构建你自己的 KV 存储](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/BAsJsdWYyIpPZjwsc1u8JQ)
[Go 夜读：高性能预写日志（Write Ahead Log）的设计与实现](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1V44y1A7Mx)

总体来说 wal 就是一个 append-only 的日志文件，提供了基础的读写方法，基本上参考了 leveldb 的预写日志文件的格式。

![img](https://pic3.zhimg.com/80/v2-1e2bfa053b3acd365a322681b78fe32a_1440w.webp)


写是一直 append 的，写完之后会得到一个表示数据位置的结构体，通过这个结构可以读取到对应的数据。

## 这次对 wal 的优化

之前对整个 wal 文件进行遍历的时候，如果 value 比较小，那么会多次重复读取 value 所属的 block，这样的话效率比较低，而且是完全没必要的。
之前的策略是加上了一个 block cache 来缓解这个问题。

但是细想，一个 block 读上来之后，如果 value 仍然在当前 block 中，那么可以重复利用这个 block，不用再去读取文件了。
在这个思路之下，对 wal 的读取进行了优化，主要是去掉了 block cache，并且如果 value 比较小的话，会直接重复利用当前 block，避免重复读取。

优化之后的效果还是比较明显的，在我的机器上，遍历 1.8G 的数据，花了 5 s 左右，之前是 20s，遍历读取的性能提升在 4-5 倍左右。

![img](https://pic2.zhimg.com/80/v2-46aec859c87c590a15da49cc3cda6bb1_1440w.webp)


带来的一个好处便是，rosedb 的启动速度会得到提升，因为 rosedb 在启动的时候，会加载所有的 wal 文件进行索引的构建。
