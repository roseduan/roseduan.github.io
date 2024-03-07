---
title: Go 语言数据库/存储项目推荐
date: 2024-03-06T21:07:56+08:00
categories:
    - Go 语言
    - 数据库
tags:
    - 项目推荐
---

大家好，今天给大家分享一些使用 Go 语言编写的数据库/存储项目。

因为我的两个存储引擎开源项目 `rosedb` 和 `lotusdb` 都是使用 Go 语言编写的，所以这几年在这方面也有很多的积累，今天就把自己压箱底的干货分享给大家。

首先需要说明的是，像 `TiDB`、`CockroachDB`、`etcd` 这些大型的项目我就不再列举了，因为这些项目都耳熟能详，公开的资料也很多，不需要我再多说什么了。

**今天想给大家分享的是，一些比较小众的，代码量在 1w 行以内，适合大家去学习上手的一些项目。如果你想入门数据库/存储领域，这些项目其实都非常适合你去研究学习，能够让你对数据库/存储领域有一个更深入的了解。如果你是学习了 Go 语言，想找一个小项目来练手，这些项目也是非常适合你。**

这次分享的项目主要分为了两个大的类型，一是一些基础类型的教程，二是一些比较完整的项目。 最后也会分享我自己推荐的学习方式。

## 基础类型的教程

1. 第一个是我自己写的 mini-bitcask 教程 https://github.com/rosedblabs/mini-bitcask，300 多行代码实现了一个极简的 bitcask 存储引擎，可以看做是 rosedb 的 mini 版本，对于你学习存储引擎的原理和实现有很大的帮助。我之前还专门写了一篇文章来介绍这个项目，可以结合起来观看效果更佳。
2. 两百行代码实现基于 paxos 的分布式 KV https://github.com/openacid/paxoskv，也有一个专门讲解的博客文章，非常值得学习。
3. 从零开始写时序数据库 https://github.com/chenjiandongx/mandodb。这个项目是一个时序数据库，作者从零开始写了一个简单的时序数据库，代码量不大，适合新手学习。
4. Go 语言实现的易于学习的 sql 数据库 https://github.com/qw4990/NYADB2，参考了很多 boltdb 的实现。
5. 1k 行代码的极简分布式 kv 数据库 https://github.com/geohot/minikeyvalue，并且用于了生产环境。

## 进阶类型的项目

### 关系型数据库

关系型数据库这里推荐几个我觉得还不错的，但是关系型 DB 难度肯定比 KV 更大，因为关系型 DB 包含了多个组件比如 parser、执行器、事务、存储等模块，感兴趣的同学可以参考。

https://github.com/chaisql/chai，嵌入式 SQL 数据库，兼容 Postgres 的 sql，支持持久化存储。

https://github.com/codenotary/immudb，支持文档、SQL、KV 的多模数据库。

https://github.com/auxten/go-sqldb，简单的 sql  数据库，使用 B+ 树存储数据，实现了 parser 和 executor。

https://github.com/rqlite/rqlite，基于 sqlite 的分布式数据库，可以认为是 raft+sqlite。

https://github.com/hashicorp/go-memdb，hashicorp 的内存型嵌入式数据库，比较轻量级。

### KV 数据库

#### Bitcask

rosedb https://github.com/rosedblabs/rosedb，基于 bitcask 的 KV 存储引擎，轻量级，支持 WriteBatch、TTL、Scan 等功能，目前是被应用到了生产环境中。

nutsdb https://github.com/nutsdb/nutsdb，同样也是基于 bitcask 的 KV 存储引擎，支持类似 Redis 的数据结构，国人开发和维护。

#### B+Tree

boltdb https://github.com/etcd-io/bbolt, Go 语言领域知名的存储引擎，B+ 树实现，支持一写多读的事务，广泛运用于生产环境，etcd 就是使用了 boltdb 作为持久化存储引擎。

#### LSM Tree

goleveldb https://github.com/syndtr/goleveldb，leveldb 的 Go 语言实现，学习 LSM Tree 实现细节的好项目。

badger https://github.com/dgraph-io/badger，wisckey 的实现，LSM Tree KV 分离。

pebble https://github.com/cockroachdb/pebble，CockroachDB 的底层存储引擎，目前 Go 领域最难的 KV 存储引擎了，设计非常精细，代码量也比较大，主要参考了 RocksDB。

#### Hybrid（LSM+BPTree）

lotusdb https://github.com/lotusdblabs/lotusdb，结合 LSM 和 B+Tree 的存储引擎，架构较为新颖。

## 学习建议

最后，针对 KV 数据库的学习，这里给出我的一些小的建议。

从存储模型上来说，主流的 KV 存储模型有两种，分别是 `B+Tree` 和 `LSM Tree`，当然后来也出现了很多基于此的变种和优化，但最基本的还是这两个。

bitcask 可以看做是一个简化版的 LSM Tree，它大致只包含 LSM 中的 wal 和 memtable 组件，LSM 中最复杂的 SSTable 组件被省略了。

所以在学习上，建议先从 bitcask 学起，可以参考我的那个 mini-bitask 教程，结合文章，很容易就能够理解了，代码也只有 300 多行。

然后再看看我的 rosedb 项目，就基本上能够理解 bitcask 存储模型了。


有了这个基础之后，可以再学习 B+树或者 LSM Tree，倒也不用两个都学，可以挑选一个自己感兴趣的去学习。

B+树的话就去看 boltdb，网上也有很多 boltdb 源码解析的文章。LSM Tree 的话推荐 goleveldb，结合 leveldb 一些资料和原理，应该理解起来也不难。

当然，看代码学习也只是迈出了第一步，想要更加深入的话，比如自己去撸一个出来，就需要花费更多的时间和精力了。
