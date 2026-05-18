---
title: "Go 实战项目 rosedb 源码剖析 5—内存索引"
date: 2025-02-18 08:41:59
categories:
    - 编程故事
tags:
    - 公众号迁移
wechat_url: "https://mp.weixin.qq.com/s/qDrqTycS5NGtVGqLYcbMmQ"
---

在 bitcask 论文当中，索引使用了一个哈希表来存储，这样做的好处是插入、查找、删除操作的时间复杂度都是接近于 O(1) 的，点查的速度很快。

![](/img/wechat/45-rosedb-source-5-memory-index/01.png)

但是具体实现的时候，内存索引的选择可以是多样化的，任意能够存储 key/value 对，并且可以根据 key 取出 value 的内存数据结构都是可以的，例如 BTree、自适应基数树（ART）、红黑树、哈希表等等。所以在实现的过程当中，可以根据不同的需要去进行选择。

例如如果想要性能更好，并且对 key 顺序扫描没有要求，那么可以使用哈希表；如果想要 key 有序进行扫描，可以使用一些天然支持排序的数据结构，比如 BTree；如果想要更加的节省内存空间，那么可以使用自适应基数树。

在 rosedb 当中，最开始的 V1 版本中使用了自适应基数树（ART）作为内存索引，目前在 V2 的版本中，使用 BTree 作为内存索引。

Google 开源的 Go 语言实现的 BTree(<https://github.com/google/btree>)是一个使用广泛、性能优异、接口功能丰富的库，并且能够天然的支持 key 有序排列，对顺序扫描，以及范围扫描都能够很好的支持。

并且 BTree 的插入、删除、查询操作的时间复杂度接近 O(logN)，速度也非常的快。

rosedb 当中的索引接口是一个 interface，这样方便在代码实现层面去接人不同的索引数据结构，索引接口的定义如下：

```go
// Indexer is an interface for indexing key and position.  
// It is used to store the key and the position of the data in the WAL.  
// The index will be rebuilt when the database is opened.  
// You can implement your own indexer by implementing this interface.  
type Indexer interface {  
    // Put key and position into the index.  
    Put(key []byte, position *wal.ChunkPosition) *wal.ChunkPosition  

    // Get the position of the key in the index.  
    Get(key []byte) *wal.ChunkPosition  

    // Delete the index of the key.  
    Delete(key []byte) (*wal.ChunkPosition, bool)  

    // Size represents the number of keys in the index.  
    Size() int  
}
```

当然除了基础的增删改查的方法之外还有一些数据遍历的方法。

开发者可以在这个基础之上去做其他的延伸，比如去实现其他的数据结构，只需要实现这个 interface 即可，其他的上下层代码都不需要进行修改。

```go
// Change the index type as you implement.  
var indexType = BTree  

func NewIndexer() Indexer {  
    switch indexType {  
    case BTree:  
        return newBTree()  
    default:  
        panic("unexpected index type")  
    }  
}
```

目前在 rosedb 当中只实现了 Google 的 BTree，对 btree 做了一个简单的封装，使用了 Mutex 做为读写互斥锁。

```go
// MemoryBTree is a memory based btree implementation of the Index interface  
// It is a wrapper around the google/btree package: github.com/google/btree  
type MemoryBTree struct {  
    tree *btree.BTree  
    lock *sync.RWMutex  
}
```

一些简单的 Put、Get、Delete 的方法都调用了btree 的方法。

```go
func (mt *MemoryBTree) Put(key []byte, position *wal.ChunkPosition) *wal.ChunkPosition {  
    mt.lock.Lock()  
    defer mt.lock.Unlock()  

    oldValue := mt.tree.ReplaceOrInsert(&item{key: key, pos: position})  
    if oldValue != nil {  
        return oldValue.(*item).pos  
    }  
    return nil  
}  

func (mt *MemoryBTree) Get(key []byte) *wal.ChunkPosition {  
    mt.lock.RLock()  
    defer mt.lock.RUnlock()  
    value := mt.tree.Get(&item{key: key})  
    if value != nil {  
        return value.(*item).pos  
    }  
    return nil  
}  

func (mt *MemoryBTree) Delete(key []byte) (*wal.ChunkPosition, bool) {  
    mt.lock.Lock()  
    defer mt.lock.Unlock()  

    value := mt.tree.Delete(&item{key: key})  
    if value != nil {  
        return value.(*item).pos, true  
    }  
    return nil, false  
}
```

rosedb 的索引是可以自定义实现的，开发者如果感兴趣的话，可以去按照各自的需求进行实现，这里推荐一些开源的库，可以作为一些不错的候选，感兴趣的同学可以自己去实现。

* <https://github.com/armon/go-radix>：基数树实现
* <https://github.com/plar/go-adaptive-radix-tree>：自适应基数树实现
* <https://github.com/emirpasic/gods>：包含红黑树、AVL 树等数据结构的实现
* <https://github.com/derekparker/trie>：Go 语言的字典树实现

---

广告区：

《从零实现 KV 存储》，使用 Rust 和 Go 两种语言实现，手把手教学，只需要基础的语法知识，即可学会一个硬核实战项目！

《从零实现 SQL 数据库》，使用 Rust 手写一个数据库系统，超级硬核，Rust 实战项目首选！

课程详情：<https://roseduan.cn/course/>
