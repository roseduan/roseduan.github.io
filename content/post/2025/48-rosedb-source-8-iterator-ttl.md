---
title: "Go 实战项目 rosedb 源码剖析 8—迭代器、过期时间"
date: 2025-02-22 09:23:58
categories:
    - 编程故事
tags:
    - 公众号迁移
wechat_url: "https://mp.weixin.qq.com/s/QXE22Z90_sCVEiJpQb3Xdw"
---

## 1、迭代器

由于全内存索引的缘故，rosedb 的迭代器实现起来相对简单，因为在内存中已经维护了全量的 key 和索引信息，并且前面提到过 rosedb 中使用了一个 btree 数据结构，btree 本身是有序的，所以在迭代数据的时候只需要根据 key 的信息取出索引，然后从数据文件中读到对应的 Value 即可。

在迭代数据的时候，rosedb 提供了两种方式，分别是调用 Ascend、Descend 这种方法，可以直接传入一个自定义的方法处理 Key 和 Value，定义大致如下：

```go
// Ascend calls handleFn for each key/value pair in the db in ascending order.  
func (db *DB) Ascend(handleFn func(k []byte, v []byte) (bool, error)) {  
    db.mu.RLock()  
    defer db.mu.RUnlock()  

    db.index.Ascend(func(key []byte, pos *wal.ChunkPosition) (bool, error) {  
        chunk, err := db.dataFiles.Read(pos)  
        if err != nil {  
            return false, err  
        }  
        if value := db.checkValue(chunk); value != nil {  
            return handleFn(key, value)  
        }  
        return true, nil  
    })  
}
```

还有一种方式是使用更常见的 Iterator 迭代器，这种迭代器可以更加灵活的控制，比如可以进行 Seek、可以手动调用 Next 跳转到下一个 Value，大致使用方法如下：

```go
iter := db.NewIterator(iterOpts)  
defer iter.Close()  

for iter.Rewind(); iter.Valid(); iter.Next() {  
    item := iter.Item()  
    if item != nil {  
        fmt.Printf("key = %s, value = %s\n", string(item.Key), string(item.Value))  
    }  
}
```

在创建 Iterator 的时候可以传入一个参数配置，目前有如下三个配置项：

```go
// IteratorOptions defines configuration options for creating a new iterator.  
type IteratorOptions struct {  
    // Prefix specifies a key prefix for filtering. If set, the iterator will only  
    // traverse keys that start with this prefix. Default is empty (no filtering).  
    Prefix []byte  

    // Reverse determines the traversal order. If true, the iterator will traverse  
    // in descending order. Default is false (ascending order).  
    Reverse bool  

    // ContinueOnError determines how the iterator handles errors during iteration.  
    // If true, the iterator will log errors and continue to the next entry.  
    // If false, the iterator will stop and become invalid when an error occurs.  
    ContinueOnError bool  
}
```

* Prefix：指定扫描的 key 前缀，只有前缀匹配才会输出对应的数据
* Reverse：控制是否进行迭代反转，false 为正向迭代，true 为反向迭代
* ContinueOnError：在遇到某个 value 读取错误的时候，是否继续读取下一条数据

### 索引迭代器

面向外层调用者的迭代器接口，在 rosedb 内部，实际上是基于索引迭代器进行实现的，所以先来介绍下索引迭代器。

索引迭代器的方法名和外层的方法基本类似，大致的区别是索引返回的 Value 是 `ChunkPosition`，而外层 DB 的迭代器，需要根据 `ChunkPosition` 从 wal 当中读取 Value。

索引层面的迭代器也是分为了两种，一种是 Ascend 和 Descend 方法：

```go
// Ascend iterates over items in ascending order and invokes the handler function for each item.  
// If the handler function returns false, iteration stops.  
Ascend(handleFn func(key []byte, position *wal.ChunkPosition) (bool, error))  

// AscendRange iterates in ascending order within [startKey, endKey], invoking handleFn.  
// Stops if handleFn returns false.  
AscendRange(startKey, endKey []byte, handleFn func(key []byte, position *wal.ChunkPosition) (bool, error))  

// AscendGreaterOrEqual iterates in ascending order, starting from key >= given key,  
// invoking handleFn. Stops if handleFn returns false.  
AscendGreaterOrEqual(key []byte, handleFn func(key []byte, position *wal.ChunkPosition) (bool, error))  

// Descend iterates over items in descending order and invokes the handler function for each item.  
// If the handler function returns false, iteration stops.  
Descend(handleFn func(key []byte, pos *wal.ChunkPosition) (bool, error))  

// DescendRange iterates in descending order within [startKey, endKey], invoking handleFn.  
// Stops if handleFn returns false.  
DescendRange(startKey, endKey []byte, handleFn func(key []byte, position *wal.ChunkPosition) (bool, error))  

// DescendLessOrEqual iterates in descending order, starting from key <= given key,  
// invoking handleFn. Stops if handleFn returns false.  
DescendLessOrEqual(key []byte, handleFn func(key []byte, position *wal.ChunkPosition) (bool, error))
```

这种方法，直接调用了 BTree 提供的方法进行遍历，以 Ascend 为例：

```go
func (mt *MemoryBTree) Ascend(handleFn func(key []byte, position *wal.ChunkPosition) (bool, error)) {  
    mt.lock.RLock()  
    defer mt.lock.RUnlock()  

    mt.tree.Ascend(func(i btree.Item) bool {  
        cont, err := handleFn(i.(*item).key, i.(*item).pos)  
        if err != nil {  
            return false  
        }  
        return cont  
    })  
}
```

另一种是 Iterator 迭代器接口，定义如下：

```go
// IndexIterator represents a generic index iterator interface.  
type IndexIterator interface {  
    // Rewind resets the iterator to its initial position.  
    Rewind()  

    // Seek positions the cursor to the element with the specified key.  
    Seek(key []byte)  

    // Next moves the cursor to the next element.  
    Next()  

    // Valid checks if the iterator is still valid for reading.  
    Valid() bool  

    // Key returns the key of the current element.  
    Key() []byte  

    // Value returns the value (chunk position) of the current element.  
    Value() *wal.ChunkPosition  

    // Close releases the resources associated with the iterator.  
    Close()  
}
```

* Rewind：迭代器指针重新回到初始位置
* Seek：跳转到第一个大于等于（或小于等于）给定值的位置
* Next：跳转到下一个值
* Valid：是否有效，用于表示遍历是否已经结束
* Key：用户实际的 key
* Value：内存中的 Value，返回值是 `ChunkPosition`
* Close：关闭迭代器，清理资源

这种 Iterator 迭代器实际上底层的 BTree 并没有直接提供类似的接口，我们进行了一层封装，利用 BTree 已有的方法去实现了迭代器中的接口，例如 Rewind 方法让迭代器指针重新回到初始位置，调用了 Btree 提供的 Min 或 Max 方法：

```go
func (it *memoryBTreeIterator) Rewind() {  
    if it.tree == nil || it.tree.Len() == 0 {  
        return  
    }  

    if it.reverse {  
        it.current = it.tree.Max().(*item)  
    } else {  
        it.current = it.tree.Min().(*item)  
    }  
    it.valid = true  
}
```

例如 Next 方法，直接找到比当前值更大（或更小）的值即可：

```go
it.valid = false  
if it.reverse {  
    it.tree.DescendLessOrEqual(it.current, func(i btree.Item) bool {  
        if !i.(*item).Less(it.current) {  
            return true  
        }  
        it.current = i.(*item)  
        it.valid = true  
        return false  
    })  
} else {  
    it.tree.AscendGreaterOrEqual(it.current, func(i btree.Item) bool {  
        if !it.current.Less(i.(*item)) {  
            return true  
        }  
        it.current = i.(*item)  
        it.valid = true  
        return false  
    })  
}
```

### DB 迭代器接口

在索引迭代器的基础之上，DB 迭代器的接口就是面向实际用户的了，实现上也比较简单，直接对索引迭代器进行封装。

```go
// Iterator represents a database-level iterator that  
// provides methods to traverse over the key/value pairs in the database.  
// It wraps the index iterator and adds functionality to  
// retrieve the actual values from the database.  
type Iterator struct {  
    indexIter index.IndexIterator // index iterator for traversing keys  
    db        *DB                 // database instance for retrieving values  
    options   IteratorOptions     // user-defined configuration options  
    lastError error               // stores the last error encountered during iteration  
}
```

提供的方法是类似的，唯一的区别是索引迭代器的返回值是内存索引 `ChunkPosition`，这里需要根据 `ChunkPosition` 从 wal 文件中读取实际的 Value。

```go
position := it.indexIter.Value()  
if position == nil {  
    it.indexIter.Next()  
    continue  
}  

// read the record from data file  
chunk, err := it.db.dataFiles.Read(position)  
if err != nil {  
    it.lastError = err  
    if !it.options.ContinueOnError {  
        it.Close()  
        return nil  
    }  
    log.Printf("Error reading data file at key %q: %v", key, err)  
    it.indexIter.Next()  
    continue  
}
```

## 2、过期时间

rosedb 支持过期时间，也就是说一个 key 可以存活一段时间，然后自动失效。

在实现上，主要是对 `LogRecord` 加上一个额外的字段 expire，表示其过期时间，然后在写入的时候将这个信息存储起来，并且在读取的时候，需要结合这个字段进行判断，如果 expire 时间已经超过了当前时间，那么就认为 key 是无效的。

```go
// LogRecord is the log record of the key/value pair.  
// It contains the key, the value, the record type and the batch id  
// It will be encoded to byte slice and written to the wal.  
type LogRecord struct {  
    Key     []byte  
    Value   []byte  
    Type    LogRecordType  
    BatchId uint64  
    Expire  int64  
}
```

### PutWithTTL

PutWithTTL 和 Put 方法类似，但是可以额外指定一个 ttl，表示 key 的存活时间。

在填充 LogRecord 的时候，只需要结合 ttl 和当前时间就能够计算得到 expire 信息。

```go
// PutWithTTL adds a key-value pair with ttl to the batch for writing.  
func (b *Batch) PutWithTTL(key []byte, value []byte, ttl time.Duration) error {  
    // 其他代码省略.....  
    record.Key, record.Value = key, value  
    record.Type, record.Expire = LogRecordNormal, time.Now().Add(ttl).UnixNano()  
    b.mu.Unlock()  

    return nil  
}
```

在读取的时候，从 wal 中拿到的是 chunk 信息，也就是一个字节数组，然后我们需要解码得到 `LogRecord` 的信息。

这时候需要判断 expire 是否已经超过了当前时间，如果 key 已经过期了，那么直接将其从索引中删除，并且返回 Key 不存在的错误。

```go
    // check if the record is deleted or expired  
    record = decodeLogRecord(chunk)  
    if record.Type == LogRecordDeleted {  
        panic("Deleted data cannot exist in the index")  
    }  
    if record.IsExpired(now) {  
        b.db.index.Delete(record.Key)  
        return nil, ErrKeyNotFound  
    }
```

### Expire

Expire 方法主要是对一个已经存在的 key 设置过期时间。

如果针对的是已经存在于 `pendingWrites` 临时数组中的数据，那么可以直接更新其过期时间，并且在 Commit 阶段再写入到 wal 文件中。

```go
    // if the key exists in pendingWrites, update the expiry time directly  
    if record != nil {  
        // return key not found if the record is deleted or expired  
        if record.Type == LogRecordDeleted || record.IsExpired(time.Now().UnixNano()) {  
            return ErrKeyNotFound  
        }  
        record.Expire = time.Now().Add(ttl).UnixNano()  
        return nil  
    }
```

否则的话，则需要从 wal 文件中读取数据，如果 key 不存在的话直接返回错误，否则更新数据的过期时间。

然后写入到 `pendingWrites` 中，待 Commit 时进行写入。

```go
    now := time.Now()  
    record = decodeLogRecord(chunk)  
    // if the record is deleted or expired, we can assume that the key does not exist,  
    // and delete the key from the index  
    if record.Type == LogRecordDeleted || record.IsExpired(now.UnixNano()) {  
        b.db.index.Delete(key)  
        return ErrKeyNotFound  
    }  
    // now we get the value from wal, update the expiry time  
    // and rewrite the record to pendingWrites  
    record.Expire = now.Add(ttl).UnixNano()  
    b.appendPendingWrites(key, record)
```

### TTL

TTL 方法返回一个 key 的存活时间。

具体逻辑也比较简单，首先还是从 `pendingWrites` 中获取，如果能拿到数据的话，则直接根据 expire 信息得到 tll 值。

```go
    var record = b.lookupPendingWrites(key)  
    if record != nil {  
        if record.Expire == 0 {  
            return -1, nil  
        }  
        // return key not found if the record is deleted or expired  
        if record.Type == LogRecordDeleted || record.IsExpired(now.UnixNano()) {  
            return -1, ErrKeyNotFound  
        }  
        // now we get the valid expiry time, we can calculate the ttl  
        return time.Duration(record.Expire - now.UnixNano()), nil  
    }
```

否则需要从 wal 文件中获取数据，并且解码得到 `LogRecord`，并且判断其是否已经过期了，然后计算得到 ttl。

```go
    chunk, err := b.db.dataFiles.Read(position)  
    if err != nil {  
        return-1, err  
    }  

    // return key not found if the record is deleted or expired  
    record = decodeLogRecord(chunk)  
    if record.Type == LogRecordDeleted {  
        return-1, ErrKeyNotFound  
    }  
    if record.IsExpired(now.UnixNano()) {  
        b.db.index.Delete(key)  
        return-1, ErrKeyNotFound  
    }  

    // now we get the valid expiry time, we can calculate the ttl  
    if record.Expire > 0 {  
        return time.Duration(record.Expire - now.UnixNano()), nil  
    }
```

### Persist

`Persist` 方法主要用于移除一个 key 的过期时间，这样 key 就能够永久存活了。

首先还是从 `pendingWrites` 中获取，如果存在的话则直接将 expire 信息设置为 0。

```go
    // if the key exists in pendingWrites, update the expiry time directly  
    var record = b.lookupPendingWrites(key)  
    if record != nil {  
        if record.Type == LogRecordDeleted && record.IsExpired(time.Now().UnixNano()) {  
            return ErrKeyNotFound  
        }  
        record.Expire = 0  
        return nil  
    }
```

否则从 wal 文件中读取数据，解码得到 LogRecord，然后将 expire 设置为 0，并且存到临时数组中，待 Commit 时写入。

```go
    chunk, err := b.db.dataFiles.Read(position)  
    if err != nil {  
        return err  
    }  

    record = decodeLogRecord(chunk)  
    now := time.Now().UnixNano()  
    // check if the record is deleted or expired  
    if record.Type == LogRecordDeleted || record.IsExpired(now) {  
        b.db.index.Delete(record.Key)  
        return ErrKeyNotFound  
    }  
    // if the expiration time is 0, it means that the key has no expiration time,  
    // so we can return directly  
    if record.Expire == 0 {  
        return nil  
    }  

    // set the expiration time to 0, and rewrite the record to wal  
    record.Expire = 0  
    b.appendPendingWrites(key, record)
```

### DB 过期时间接口

前面提到的几个方法都是 Batch 中的，在 DB 实例中也有一个同样的方法，其实和上一节提到的关于 Put、Delete、Get 方法类似，都是对 Batch 接口做了一层简单的封装，类似只有单个操作的 Batch。

```go
// PutWithTTL a key-value pair into the database, with a ttl.  
// Actually, it will open a new batch and commit it.  
// You can think the batch has only one PutWithTTL operation.  
func (db *DB) PutWithTTL(key []byte, value []byte, ttl time.Duration) error {  
    batch := db.batchPool.Get().(*Batch)  
    deferfunc() {  
        batch.reset()  
        db.batchPool.Put(batch)  
    }()  
    // This is a single put operation, we can set Sync to false.  
    // Because the data will be written to the WAL,  
    // and the WAL file will be synced to disk according to the DB options.  
    batch.init(false, false, db)  
    if err := batch.PutWithTTL(key, value, ttl); err != nil {  
        _ = batch.Rollback()  
        return err  
    }  
    return batch.Commit()  
}
```

### 过期 key 清理

如果过期的 key 没有及时清理的话，key 和索引都会残留在内存中，造成不必要的空间浪费。

在 rosedb 当中，有两种方式可以对过期的 key 进行清理。

一种是自动的，比如在调用 `Get` 或者 `Exist` 的时候，如果判断到 key 是存在的，但是已经过期了，那么会直接删除掉索引中对应的 key。

```go
    if record.IsExpired(now) {  
        b.db.index.Delete(record.Key)  
        return nil, ErrKeyNotFound  
    }
```

第二种方式是调用者手动进行清理，rosedb 当中提供了方法 `DeleteExpiredKeys` ，这个方法主要是扫描索引中的数据，并且判断是否已经过期了，如果是的话则从索引中将其删除。

```go
// delete from index if the key is expired.  
for _, pos := range positions {  
    chunk, err := db.dataFiles.Read(pos)  
    if err != nil {  
        innerErr = err  
        done <- struct{}{}  
        return  
    }  
    record := decodeLogRecord(chunk)  
    if record.IsExpired(now) {  
        db.index.Delete(record.Key)  
    }  
    db.expiredCursorKey = record.Key  
}
```

调用者可以在外层封装一个定时任务，实现自动清理过期 key 的功能。
