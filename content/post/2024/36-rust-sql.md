---
title: Rust 手写数据库的成就感！
date: 2024-12-05T16:51:56+08:00
categories:
    - Rust
    - Go
tags:
    - SQL 数据库
---

前几天刚好完成了 rust 手写数据库课程的命令行工具，有一个交互式的客户端，瞬间成就感拉满了！

不枉我写这个项目接近一年的时间，觉得之前的努力都没有白费哈哈。

在这个客户端当中，可以创建表：

![img](https://picx.zhimg.com/v2-8914f71c13b92f6a9abb23863518a40d_1440w.jpg)

可以增删改查数据：

![img](https://pic4.zhimg.com/v2-5a9d47cf3434e97910dc1f78ef641481_1440w.jpg)



![img](https://pic3.zhimg.com/v2-1542ec04ea4aa071a377b1d5854e366c_1440w.jpg)

可以进行常见的 sql 查询，比如

- Order By
- Limit 和 Offset
- Projection 投影
- Join 语句
- Agg [聚集函数](https://zhida.zhihu.com/search?content_id=251216851&content_type=Article&match_order=1&q=聚集函数&zhida_source=entity)
- Group By 分组

![img](https://pic4.zhimg.com/v2-e9a17a9e177cac1ea41199522210ac71_1440w.jpg)



![img](https://pic2.zhimg.com/v2-39664db4a1d182b1d8ac245f0754a72b_1440w.jpg)

并且还支持 ACID 的事务操作：

![img](https://pica.zhimg.com/v2-fcaf19b7e6715ee293375e36331d6978_1440w.jpg)

也可以查看当前表的信息：

![img](https://pic3.zhimg.com/v2-f88cc3ff3f6573c1e3a270f9d110aae4_1440w.jpg)

《从零实现 SQL 数据库》是我今年开始搞的，最初也只是想着试试看能不能做，但是后来帮助了一些同学，他们都从中学习到了很多，这也让我一路坚持到了现在。

课程虽然实现的只是一个非常简单的数据库，但是麻雀虽小五脏俱全，数据库内核的各个模块基本都实现了。

相信通过这样一个项目，对编程基础、[rust](https://zhida.zhihu.com/search?content_id=251216851&content_type=Article&match_order=2&q=rust&zhida_source=entity) 上手、[项目实战](https://zhida.zhihu.com/search?content_id=251216851&content_type=Article&match_order=1&q=项目实战&zhida_source=entity)、系统设计等等能力都会上一个台阶！

现在课程项目接近尾声了，代码量（加注释）接近 6000 行，是非常不错的适合上手和实践 rust 的项目。

![img](https://pic1.zhimg.com/v2-2189148cc0a9a5c22c7769857b49e970_1440w.jpg)

课程的详细目录如下：

![img](https://pic1.zhimg.com/v2-67a4387d0a16cc4ad2880126b0ebb19a_1440w.jpg)

![img](https://pic2.zhimg.com/v2-d3eb15903564359b63aebbefd6545ca9_1440w.jpg)

感兴趣的同学可以进这个课程详情链接查看：

https://icnyamgobd0u.feishu.cn/docx/AbXZdEbY0obdcTxF0FKcx4Pqnyf