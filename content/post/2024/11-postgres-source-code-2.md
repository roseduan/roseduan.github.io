---
title: Postgres 源码学习 2—Postgres 的 VFD 机制
date: 2024-04-18T10:51:56+08:00
categories:
    - 数据库
    - Postgres
tags:
    - Postgres 源码学习
---

## 操作系统中的文件

数据库的本质其实就是用来存储数据的，所以免不了和文件系统、存储进行交互，万丈高楼平地起，存储一般是一个数据库的最底层，Postgres 在存储的文件管理方面也有很多的设计与抽象。

在操作系统层面，提供了一些文件操作相关的系统调用（fopen、fclose、fsync 等），我们作为上层使用者，可以直接通过 C 语言库进行调用即可（Postgres 使用 C 语言编写）。

具体和文件系统的交互我们并不关心，操作系统打开文件之后，会在进程的控制块中维护一些打开文件的相关信息，并返回一个文件描述符，后续我们与文件的交互都通过文件描述符进行。

操作系统能够打开多少文件，是有限制的，一个是系统级限制，指的是在内核中可以打开多少文件，可以通过命令 `sysctl fs.file-max` 查看。另一个是用户级限制，为了不让某个进程打开太多的文件，进而消耗所有的资源，对单个进程能打开文件也有限制，可以通过 `ulimit -n` 命令查看。

## Postgres 的 VFD 作用

Postgres 数据库在运行的过程当中，可能会打开非常多的文件，比如数据表对应的文件，元数据表文件，以及一些在 SQL 运行时打开的临时文件，例如排序、哈希表所需的文件。

所以有非常大的概率超过单个进程打开文件数量的限制，为了解决这个问题，Postgres 设计了 VFD（虚拟文件描述符）机制，主要是将实际的操作系统文件描述符维护到一个 LRU 缓存中，通过切换打开的方式，规避了进程打开文件数量的限制。

如果一个进程打开的文件数目达到了限制，则暂时关闭最久未使用的文件，保存其状态，待下次重新打开。

## VFD 的基本工作方式

Postgres 主要通过一个进程私有的数组来维护 VFD，名为 `VfdCache`。

```SQL
/*
 * Virtual File Descriptor array pointer and size.  This grows as
 * needed.  'File' values are indexes into this array.
 * Note that VfdCache[0] is not a usable VFD, just a list header.
 */
static Vfd *VfdCache;
```

VfdCache 数组的第一个元素不存储任何数据，仅作为头部使用，下面是 vfdCache 的初始化逻辑，会在 backend 进程启动的时候调用，大致的逻辑就是为 VfdCache 数组分配内存。

```C
/*
 * InitFileAccess --- initialize this module during backend startup
 *
 * This is called during either normal or standalone backend start.
 * It is *not* called in the postmaster.
 *
 * Note that this does not initialize temporary file access, that is
 * separately initialized via InitTemporaryFileAccess().
 */
void
InitFileAccess(void)
{
    Assert(SizeVfdCache == 0);  /* call me only once */

    /* initialize cache header entry */
    VfdCache = (Vfd *) malloc(sizeof(Vfd));
    if (VfdCache == NULL)
        ereport(FATAL,
                (errcode(ERRCODE_OUT_OF_MEMORY),
                 errmsg("out of memory")));

    MemSet((char *) &(VfdCache[0]), 0, sizeof(Vfd));
    VfdCache->fd = VFD_CLOSED;

    SizeVfdCache = 1;
}
```

如果需要打开一个文件，那么会首先在 VfdCache 数组中查找空闲的虚拟文件描述符，主要是通过 `nextFree` 指针进行查找，如果当前没有空闲的 vfd 了，那么会启动扩容机制，初始情况下，VfdCache size 是 32，每次扩容为原来的 2 倍。

Vfd 扩容和分配的逻辑都在方法 `AllocateVfd` 中。

```C
static File
AllocateVfd(void)
{
    Index       i;
    File        file;

    DO_DB(elog(LOG, "AllocateVfd. Size %zu", SizeVfdCache));

    Assert(SizeVfdCache > 0);   /* InitFileAccess not called? */

    if (VfdCache[0].nextFree == 0)
    {
        /*
         * The free list is empty so it is time to increase the size of the
         * array.  We choose to double it each time this happens. However,
         * there's not much point in starting *real* small.
         */
        Size        newCacheSize = SizeVfdCache * 2;
        Vfd        *newVfdCache;

        if (newCacheSize < 32)
            newCacheSize = 32;

        /*
         * Be careful not to clobber VfdCache ptr if realloc fails.
         */
        newVfdCache = (Vfd *) realloc(VfdCache, sizeof(Vfd) * newCacheSize);
        if (newVfdCache == NULL)
            ereport(ERROR,
                    (errcode(ERRCODE_OUT_OF_MEMORY),
                     errmsg("out of memory")));
        VfdCache = newVfdCache;

        /*
         * Initialize the new entries and link them into the free list.
         */
        for (i = SizeVfdCache; i < newCacheSize; i++)
        {
            MemSet((char *) &(VfdCache[i]), 0, sizeof(Vfd));
            VfdCache[i].nextFree = i + 1;
            VfdCache[i].fd = VFD_CLOSED;
        }
        VfdCache[newCacheSize - 1].nextFree = 0;
        VfdCache[0].nextFree = SizeVfdCache;

        /*
         * Record the new size
         */
        SizeVfdCache = newCacheSize;
    }

    file = VfdCache[0].nextFree;

    VfdCache[0].nextFree = VfdCache[file].nextFree;

    return file;
}
```

拿到虚拟文件描述符之后，会调用 C 库函数 open 实际去打开文件，并且将一些文件状态维护到 Vfd 结构体中，这个结构体主要存储的是虚拟文件描述符的一些信息，也就是存储到 VfdCache 数组中的结构。

```C
typedef struct vfd
{
    int         fd;             /* current FD, or VFD_CLOSED if none */
    unsigned short fdstate;     /* bitflags for VFD's state */
    ResourceOwner resowner;     /* owner, for automatic cleanup */
    File        nextFree;       /* link to next free VFD, if in freelist */
    File        lruMoreRecently;    /* doubly linked recency-of-use list */
    File        lruLessRecently;
    off_t       fileSize;       /* current size of file (0 if not temporary) */
    char       *fileName;       /* name of file, or NULL for unused VFD */
    /* NB: fileName is malloc'd, and must be free'd when closing the VFD */
    int         fileFlags;      /* open(2) flags for (re)opening the file */
    mode_t      fileMode;       /* mode to pass to open(2) */
} Vfd;
```

Vfd 结构体中，主要通过 nextFree、lruMoreRecently、lruLessRecently 指针将 vfd 维护到不同的队列里面。

每次新打开一个文件，都会将该 vfd 通过 lruMoreRecently 和 lruLessRecently 指针，维护这个双向链表，每次关闭一个 VfdCache 中的文件，都会将其从链表中删除。

每次查找空闲的 VfdCache 的时候，都会通过 nextFree 链表进行查找。

![img](https://pic3.zhimg.com/80/v2-416dd3fdf45cc6d45f2193fe7bdc65f6_1440w.webp)

以访问文件为例，首先会判断文件是否打开，如果没有打开的话，则打开文件并且将其放到最近使用的链表中。

主要的逻辑在函数 LruInsert 中，在实际打开文件之前，会尝试关闭最久未使用的文件。

然后会通过系统调用打开文件，并且获取到实际的文件描述符（fd），将其保存到 vfdP 结构中。

```C
static int
LruInsert(File file)
{
    Vfd        *vfdP;

    Assert(file != 0);

    DO_DB(elog(LOG, "LruInsert %d (%s)",
               file, VfdCache[file].fileName));

    vfdP = &VfdCache[file];

    if (FileIsNotOpen(file))
    {
        /* Close excess kernel FDs. */
        ReleaseLruFiles();

        /*
         * The open could still fail for lack of file descriptors, eg due to
         * overall system file table being full.  So, be prepared to release
         * another FD if necessary...
         */
        vfdP->fd = BasicOpenFilePerm(vfdP->fileName, vfdP->fileFlags,
                                     vfdP->fileMode);
        if (vfdP->fd < 0)
        {
            DO_DB(elog(LOG, "re-open failed: %m"));
            return -1;
        }
        else
        {
            ++nfile;
        }
    }

    /*
     * put it at the head of the Lru ring
     */

    Insert(file);

    return 0;
}
```

如果文件已经是打开状态，那么会先从链表中删除，然后将其插入到最近使用的链表中。将 Vfd 加入到链表中，代码如下，可以看到主要是通过维护 lruMoreRecently 和 lruLessRecently 这两个指针，将当前 vfd 加入到链表的头部。

```C
static void
Insert(File file)
{
    Vfd        *vfdP;

    Assert(file != 0);

    DO_DB(elog(LOG, "Insert %d (%s)",
               file, VfdCache[file].fileName));
    DO_DB(_dump_lru());

    vfdP = &VfdCache[file];

    vfdP->lruMoreRecently = 0;
    vfdP->lruLessRecently = VfdCache[0].lruLessRecently;
    VfdCache[0].lruLessRecently = file;
    VfdCache[vfdP->lruLessRecently].lruMoreRecently = file;

    DO_DB(_dump_lru());
}
```

而 Delete 方法则描述的是将一个 vfd 从链表中删除。

```C
static void
Delete(File file)
{
    Vfd        *vfdP;

    Assert(file != 0);

    DO_DB(elog(LOG, "Delete %d (%s)",
               file, VfdCache[file].fileName));
    DO_DB(_dump_lru());

    vfdP = &VfdCache[file];

    VfdCache[vfdP->lruLessRecently].lruMoreRecently = vfdP->lruMoreRecently;
    VfdCache[vfdP->lruMoreRecently].lruLessRecently = vfdP->lruLessRecently;

    DO_DB(_dump_lru());
}
```

## 小结

Postgres 中的 VFD，即虚拟文件描述符，主要是为了能够规避操作系统中最大打开文件数的限制，采用切换打开的方式，维护了一个链表，将最近打开的文件维护到链表头部，最久未使用的文件放置到链表尾部。

访问文件的时候，会从 VfdCache 数组中查找空闲的虚拟文件描述符，如果找到的话，则直接使用，否则分配新的 VfdCache 空间。

在打开文件的时候，会尝试关闭最久未使用的文件，将位置留给最新打开的文件。

通过这种方式，Postgres 可以打开远超过系统和进程限制的文件数量，是一个非常精妙的设计。