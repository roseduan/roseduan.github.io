---
title: "Go 实战项目 rosedb 源码剖析 7—启动流程"
date: 2025-02-20 08:42:48
categories:
    - 编程故事
tags:
    - 公众号迁移
wechat_url: "https://mp.weixin.qq.com/s/y8YA_p83cgqS44BWHSfskw"
---

理解了前面的 rosedb 数据写入、读取、merge 流程之后，再来看看数据库启动的流程。

Bitcask 模型需要在启动的时候去执行全量数据扫描，并且加载索引，在 rosedb 当中，主要涉及到加载 merge 文件，并且如果有 HINT 文件的话，可以直接从 HINT 文件中加载索引，否则从数据文件中加载索引。

rosedb 在启动的时候通过 Open 方法，启动的时候可以传递参数配置，目前有如下配置：

```go
// Options specifies the options for opening a database.  
type Options struct {  
    // DirPath specifies the directory path where the WAL segment files will be stored.  
    DirPath string  

    // SegmentSize specifies the maximum size of each segment file in bytes.  
    SegmentSize int64  

    // Sync is whether to synchronize writes through os buffer cache and down onto the actual disk.  
    // Setting sync is required for durability of a single write operation, but also results in slower writes.  
    //  
    // If false, and the machine crashes, then some recent writes may be lost.  
    // Note that if it is just the process that crashes (machine does not) then no writes will be lost.  
    //  
    // In other words, Sync being false has the same semantics as a write  
    // system call. Sync being true means write followed by fsync.  
    Sync bool  

    // BytesPerSync specifies the number of bytes to write before calling fsync.  
    BytesPerSync uint32  

    // WatchQueueSize the cache length of the watch queue.  
    // if the size greater than 0, which means enable the watch.  
    WatchQueueSize uint64  

    // AutoMergeEnable enable the auto merge.  
    // auto merge will be triggered when cron expr is satisfied.  
    // cron expression follows the standard cron expression.  
    // e.g. "0 0 * * *" means merge at 00:00:00 every day.  
    // it also supports seconds optionally.  
    // when enable the second field, the cron expression will be like this: "0/10 * * * * *" (every 10 seconds).  
    // when auto merge is enabled, the db will be closed and reopened after merge done.  
    // do not set this shecule too frequently, it will affect the performance.  
    // refer to https://en.wikipedia.org/wiki/Cron  
    AutoMergeCronExpr string  
}
```

* DirPath：表示文件的存放目录，不存在的话会自动创建
* SegmentSize：表示一个 wal 数据文件的最大容量
* Sync：是否每次写入都将操作系统缓存的数据刷到磁盘中
* BytesPerSync：累计写入多少字节后调用 Sync
* WatchQueueSize：这个参数主要是支持 watch 功能，即 key 对应的值有变化之后通知调用方
* AutoMergeCronExpr：此参数启用自动 merge 功能，填写一个 cron 表达式，后台线程自动进行 merge

在 Open 方法的具体代码中，首先会检查参数配置是否有效，如果传递的目录不存在的话，则会创建这个目录。

```go
    // check options  
    if err := checkOptions(options); err != nil {  
        return nil, err  
    }  

    // create data directory if not exist  
    if _, err := os.Stat(options.DirPath); err != nil {  
        if err := os.MkdirAll(options.DirPath, os.ModePerm); err != nil {  
            return nil, err  
        }  
    }
```

然后会使用文件锁机制，确保一个目录中只能有一个进程，即一个目录之上只能运行一个 rosedb 实例，确保数据安全。

```go
    // create file lock, prevent multiple processes from using the same database directory  
    fileLock := flock.New(filepath.Join(options.DirPath, fileLockName))  
    hold, err := fileLock.TryLock()  
    if err != nil {  
        return nil, err  
    }  
    if !hold {  
        return nil, ErrDatabaseIsUsing  
    }
```

然后是加载 merge 文件。

```go
    // load merge files if exists  
    if err = loadMergeFiles(options.DirPath); err != nil {  
        return nil, err  
    }
```

在前面讲解 merge 的流程当中，提到过 merge 完成之后，如果没有立即清理文件的话，merge 之后的文件和旧的数据文件都存在，这时候需要进行替换，将旧的数据文件删除，将 merge 的文件作为新的数据文件。

在 `loadMergeFiles` 方法中，会加载上一次 merge 终止的 SegmentID。因为我们为了检查 merge 操作是否已经完成，会在 merge 结束之后写一个文件，这个文件是以 `.MERGEFIN` 后缀结尾的。

![img](/img/wechat/47-rosedb-source-7-startup/01.png)

这个文件中记录了上一次 merge 操作完成的 SegmentID，例如上一次 merge 操作的文件是 0-5，那么这个文件中记录的就是 5。

```go
    // get the merge finished segment id  
    mergeFinSegmentId, err := getMergeFinSegmentId(mergeDirPath)  
    if err != nil {  
        return err  
    }
```

如果这个文件并不存在，或者读取的时候失败了，那么说明 merge 操作并没有成功完成，只需要删除掉 merge 目录即可。

然后读取到了这个 `mergeFinSegmentId`，只需要将这些文件移动到原数据目录当中，这样原来的文件就被删除，merge 目录中的文件就成了新的数据文件。

```go
    // now we get the merge finished segment id, so all the segment id less than the merge finished segment id  
    // should be moved to the original data directory, and the original data files should be deleted.  
    for fileId := uint32(1); fileId <= mergeFinSegmentId; fileId++ {  
        destFile := wal.SegmentFileName(dirPath, dataFileNameSuffix, fileId)  

        // remove the original data file  
        if _, err = os.Stat(destFile); err == nil {  
            if err = os.Remove(destFile); err != nil {  
                return err  
            }  
        }  
        // move the merge data file to the original data directory  
        copyFile(dataFileNameSuffix, fileId, false)  
    }
```

Merge 文件加载完成之后，数据文件的状态已经全部准备好了，这时候通过 wal 打开这些文件。

```go
func (db *DB) openWalFiles() (*wal.WAL, error) {  
    // open data files from WAL  
    walFiles, err := wal.Open(wal.Options{  
        DirPath:        db.options.DirPath,  
        SegmentSize:    db.options.SegmentSize,  
        SegmentFileExt: dataFileNameSuffix,  
        Sync:           db.options.Sync,  
        BytesPerSync:   db.options.BytesPerSync,  
    })  
    if err != nil {  
        return nil, err  
    }  
    return walFiles, nil  
}
```

打开数据文件之后，就能够加载索引了，主要的逻辑在 `loadIndex` 方法中，加载索引主要是分为了两部分，第一是如果存在 HINT 文件的话，则直接从 HINT 文件中加载，然后再从数据文件中加载。

```go
func (db *DB) loadIndex() error {  
    // load index frm hint file  
    if err := db.loadIndexFromHintFile(); err != nil {  
        return err  
    }  
    // load index from data files  
    if err := db.loadIndexFromWAL(); err != nil {  
        return err  
    }  
    return nil  
}
```

`HINT` 文件是在 merge 的过程中生成的，存储的是 `key+position` 信息，position 就是 value 在 wal 文件中的位置。需要注意 `HINT` 文件中的位置索引信息肯定是最新的，因为 merge 的时候是直接写入的最新的 value，所以得到的索引信息肯定也是最新的状态，这时候直接插入到索引当中即可。

```go
    for {  
        chunk, _, err := reader.Next()  
        if err != nil {  
            if err == io.EOF {  
                break  
            }  
            return err  
        }  

        key, position := decodeHintRecord(chunk)  
        // All the hint records are valid because it is generated by the merge operation.  
        // So just put them into the index without checking.  
        db.index.Put(key, position)  
    }
```

`HINT` 文件其实也是通过 wal 组件打开的，所以可以直接调用 wal 的遍历方法，加载其中的数据，这时候加载的并不是之前数据文件中提到的 LogRecord，而是只包含 key 和索引信息的 hint 文件的 record 信息，这里并没有使用一个专门的结构体来表示，而是直接对这两个信息进行编码，读出来之后按照同样的方式进行解码。

```go
func encodeHintRecord(key []byte, pos *wal.ChunkPosition) []byte {  
    // SegmentId BlockNumber ChunkOffset ChunkSize  
    //    5          5           10          5      =    25  
    // see binary.MaxVarintLen64 and binary.MaxVarintLen32  
    buf := make([]byte, 25)  
    var idx = 0  

    // SegmentId  
    idx += binary.PutUvarint(buf[idx:], uint64(pos.SegmentId))  
    // BlockNumber  
    idx += binary.PutUvarint(buf[idx:], uint64(pos.BlockNumber))  
    // ChunkOffset  
    idx += binary.PutUvarint(buf[idx:], uint64(pos.ChunkOffset))  
    // ChunkSize  
    idx += binary.PutUvarint(buf[idx:], uint64(pos.ChunkSize))  

    // key  
    result := make([]byte, idx+len(key))  
    copy(result, buf[:idx])  
    copy(result[idx:], key)  
    return result  
}
```

从 HINT 文件中加载索引之后，还需要从数据文件中加载索引。

试想这样一种情况，文件 0-5 是参与了 merge 过程的文件，但是在 merge 的过程当中，有可能有新的数据写入，所以可能生成了数据文件 6，甚至 7，这时候这些新的没有参与 merge 的文件是没有对应的 HINT 索引文件信息的，只能从数据文件本身去加载索引，这部分的逻辑在 `loadIndexFromWAL` 中。

在遍历数据文件加载索引的过程中，需要判断如果已经是从 HINT 文件中加载过索引了，则直接跳过。

```go
// if the current segment id is less than the mergeFinSegmentId,  
// we can skip this segment because it has been merged,  
// and we can load index from the hint file directly.  
if reader.CurrentSegmentId() <= mergeFinSegmentId {  
    reader.SkipCurrentSegment()  
    continue  
}
```

然后是从数据文件中加载 LogRecord 信息，需要判断如果类型是 `LogRecordDeleted`，则表示从索引中删除对应的数据。

并且如果一个 Batch 中的数据并没有对应的特殊的标识 Batch 结束的数据的话，则说明这个 Batch 是无效的，直接丢弃这些数据。

```go
// if we get the end of a batch,  
// all records in this batch are ready to be indexed.  
if record.Type == LogRecordBatchFinished {  
    batchId, err := snowflake.ParseBytes(record.Key)  
    if err != nil {  
        return err  
    }  
    for _, idxRecord := range indexRecords[uint64(batchId)] {  
        if idxRecord.recordType == LogRecordNormal {  
            db.index.Put(idxRecord.key, idxRecord.position)  
        }  
        if idxRecord.recordType == LogRecordDeleted {  
            db.index.Delete(idxRecord.key)  
        }  
    }  
    // delete indexRecords according to batchId after indexing  
    delete(indexRecords, uint64(batchId))  
} elseif record.Type == LogRecordNormal && record.BatchId == mergeFinishedBatchID {  
    // if the record is a normal record and the batch id is 0,  
    // it means that the record is involved in the merge operation.  
    // so put the record into index directly.  
    db.index.Put(record.Key, position)  
} else {  
    // expired records should not be indexed  
    if record.IsExpired(now) {  
        db.index.Delete(record.Key)  
        continue  
    }  
    // put the record into the temporary indexRecords  
    indexRecords[record.BatchId] = append(indexRecords[record.BatchId],  
        &IndexRecord{  
            key:        record.Key,  
            recordType: record.Type,  
            position:   position,  
        })  
}
```

索引加载完成之后，DB 启动的流程就基本完成了。

随后有一些额外的处理，第一个是如果开启了 watch 功能的话，会启动一个后台 goroutine 去处理 key 变化通知的事件信息。

```go
    // enable watch  
    if options.WatchQueueSize > 0 {  
        db.watchCh = make(chan *Event, 100)  
        db.watcher = NewWatcher(options.WatchQueueSize)  
        // run a goroutine to synchronize event information  
        go db.watcher.sendEvent(db.watchCh)  
    }
```

第二个是如果传递的参数 `AutoMergeCronExpr` 是有效的话，则也启动一个后台 goroutine 去定时周期性的去调用 merge 进行数据文件的清理。

```go
    // enable auto merge task  
    iflen(options.AutoMergeCronExpr) > 0 {  
        db.cronScheduler = cron.New(  
            cron.WithParser(  
                cron.NewParser(cron.SecondOptional | cron.Minute | cron.Hour |  
                    cron.Dom | cron.Month | cron.Dow | cron.Descriptor),  
            ),  
        )  
        _, err = db.cronScheduler.AddFunc(options.AutoMergeCronExpr, func() {  
            // maybe we should deal with different errors with different logic, but a background task can't omit its error.  
            // after auto merge, we should close and reopen the db.  
            _ = db.Merge(true)  
        })  
        if err != nil {  
            return nil, err  
        }  
        db.cronScheduler.Start()  
    }
```

---

广告区：

《从零实现 KV 存储》，使用 Rust 和 Go 两种语言实现，手把手教学，只需要基础的语法知识，即可学会一个硬核实战项目！

《从零实现 SQL 数据库》，使用 Rust 手写一个数据库系统，超级硬核，Rust 实战项目首选！

课程详情：<https://roseduan.cn/course/>
