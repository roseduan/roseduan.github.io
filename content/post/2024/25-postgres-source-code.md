---
title: Postgres 源码学习番外篇—FDW 详解
date: 2024-08-07T16:51:56+08:00
categories:
    - 数据库
    - Postgres
tags:
    - Postgres 源码学习
---

## FDW 概述

FDW，即 Foreign Data Wrapper，是 PostgreSQL 中的一项关键特性，通过接入 fdw，用户可以直接通过 SQL 语句访问各种外部[数据源](https://zhida.zhihu.com/search?q=数据源&zhida_source=entity&is_preview=1)。

![img](https://pica.zhimg.com/80/v2-17880528227249d55fecf659b40c9760_1440w.webp)

在 Postgres 中，FDW 有很多应用场景，比如：

**1. 跨数据库查询：**
在 PostgreSQL 数据库中，我们可以通过 FDW 直接请求和查询其他 PostgreSQL 实例，或是其他数据库如 MySQL、Oracle、DB2、SQL Server 等。

**2. 数据整合：**
当我们需要从不同数据源整合数据时，例如 RESTFUL API、文件系统、NoSQ L数据库以及流式系统等，FDW 能够帮助我们轻松实现这种跨来源的数据整合。

**3. 数据迁移：**
利用 FDW，我们可以高效地将数据从旧系统迁移到新的 PostgreSQL 数据库中。

**4. [实时数据](https://zhida.zhihu.com/search?q=实时数据&zhida_source=entity&is_preview=1)访问：**
通过 FDW，我们能够访问外部实时更新的数据源。

## 常见的 FDW

PostgreSQL 支持非常多常见的 FDW，能够[直接访问](https://zhida.zhihu.com/search?q=直接访问&zhida_source=entity&is_preview=1)多种类型的外部数据源。例如，可以连接并查询远程的PostgreSQL，或者其他主流的 SQL 数据库如 Oracle、MySQL、DB2 以及 SQL Server。同时，PostgreSQL FDW 也具备灵活的接口，支持用户自定义外部访问方式。

![img](https://pic4.zhimg.com/80/v2-285c7eb5c7238784d8cf1f78a5ea51e9_1440w.webp)


此外，对于 NoSQL 数据库，以及实时数据库如 InfluxDB、[消息队列](https://zhida.zhihu.com/search?q=消息队列&zhida_source=entity&is_preview=1)如 Kafka、文档型数据库如 MongoDB 等等都能通过FDW实现数据访问。

![img](https://pic3.zhimg.com/80/v2-b5d4e5f648de69ccab89fb16d2dfa3a6_1440w.webp)


常见的文本格式数据，如 CSV、JSON、Parquet 和 XML，也可以通过 FDW 轻松访问。大数据组件如 Elasticsearch、BigQuery，以及 Hadoop 生态系统中的 HDFS 和 Hive 等等都可以通过 FDW 实现无缝集成。

![img](https://pic3.zhimg.com/80/v2-9c84681f52a6a24553bdc580314136d6_1440w.webp)


## FDW 基本使用

FDW机制由四个核心组件构成：

**1. Foreign Data Wrapper：**
特定于各数据源的库，定义了如何建立与外部数据源的连接、执行查询及处理其他操作。

**2. Foreign Server：** 在本地PostgreSQL中定义一个外部服务器对象，对应实际的远程或非本地数据存储实例。

**3. User Mapping：** 为每个外部服务器设置用户映射，明确哪些本地用户有权访问，并提供相应的认证信息，如用户名和密码。

**4. Foreign Table：** 在[本地数据库](https://zhida.zhihu.com/search?q=本地数据库&zhida_source=entity&is_preview=1)创建表结构，作为外部数据源中表的映射。对这些外部表发起的 SQL 查询将被转换并传递给相应的 FDW，在外部数据源上执行。

以 postgres_fdw 为例，下面是一个 fdw 的基础使用方法：

**创建插件和 Foreign Server**

```text
创建插件

test=# create extension postgres_fdw;
CREATE EXTENSION

创建 Foreign Server

CREATE SERVER foreign_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host '127.0.0.1', port '8001', dbname 'postgres');
```

**创建 User Mapping 和外部表**

```text
创建 User Mapping

CREATE USER MAPPING FOR gpadmin SERVER foreign_server;

创建外部表

CREATE FOREIGN TABLE foreign_table (
    val int
)
SERVER foreign_server
OPTIONS (schema_name 'public', table_name 't2');
```

## FDW 实现原理

在 PostgreSQL 的内核代码中，FDW 访问外部数据源的操作接口主要通过 FdwRoutine 这一结构体进行定义。任何接入外部数据源的插件都可以根据自身需要去实现这些接口。

![img](https://pica.zhimg.com/80/v2-b655ef63c413f6abdb9dcb5db6de8f0c_1440w.webp)


接口函数大致分为多个类别，包括但不限于扫描、修改、分析外部表等等。例如，扫描外部表相关接口定义了如何扫描外部表，常见的操作包括开始扫描（ BeginForeignScan，主要进行准备工作）、执行扫描（IterateForeign Scan，从扫描中获取数据）、重新扫描（RescanForeignScan）以及结束扫描（EndForeignScan）等。

![img](https://pic3.zhimg.com/80/v2-289a04fd609f8e90357a0807a4e3eea0_1440w.webp)


此外，还有用于修改数据的**外部表接口**，支持对数据进行 insert、delete、updat e等操作，以及 explain 和 analyze 等外部表接口。

![img](https://pic3.zhimg.com/80/v2-c519aeebd64b55d148b9520245f5a662_1440w.webp)


如下图，这是一个file_fdw 插件实现 FdwRoutine 的示例，这里仅实现了一些基础的扫描操作接口（如BeginForeignScan、IterateForeignScan 等），以及用于表分析的 AnalyzeForeignTable 接口。

![img](https://pic2.zhimg.com/80/v2-28112b30957818aa921b3216f914ee53_1440w.webp)


在 PostgreSQL 的执行过程中，这些接口函数会在 planner 或 executor 阶段被调用。尤其是当 executor 需要依赖外部服务插件访问数据时，它会通过插件提供的数据访问接口来获取数据。这使得 FDW 能够与 PostgreSQL 的Parser、Planner 以及 Rewriter 等组件能够无缝协作。

![img](https://pic3.zhimg.com/80/v2-a2f3236f67cc8b398f14e689df6d4042_1440w.webp)


在需要访问外部数据源时，我们只需定义好相应的数据访问接口，就能直接获取数据，并按照PostgreSQL的标准流程进行后续处理。在执行过程中，执行器会分解为几个阶段进行：

1.首先进入 **初始化阶段**，核心任务是执行外部表扫描 ExecInitForeignScan。在这个阶段，主要是定义了一些外部扫描的接口，并调用 FdwRoutine 中用户自定义的接口，从而进行扫描前的准备。

![img](https://pic3.zhimg.com/80/v2-d9d88d2a0d657c122dce4fa1eddb7218_1440w.webp)


2.紧接着是**执行查询阶段** ，此时会调用 ExecuteForeignScan 方法。在这个方法中，我们主要需指定 ForeignNext 来获取下一条数据，并定义 ForeignRecheck 来检验数据元组的可见性。

![img](https://pic2.zhimg.com/80/v2-8f9ef9a5e2ec09c02f92b7adde654805_1440w.webp)


3.最后进入**结束查询阶段** ，即执行 EndForeignScan，该阶段主要负责资源清理工作。若系统检测到存在 FDWRoutine，就会利用用户自定义的 EndForeignScan 函数来释放资源。

![img](https://picx.zhimg.com/80/v2-3a420250a9830f9a7153f33c1e535981_1440w.webp)


以上就是FDW整体的实现流程。接下来，为了更深入地了解FDW的工作机制，我们将深入探讨FDW的源码。

## FDW源码解析

FDW 支持的数据类型众多，但在此我们以常见的 Postgres_fdw 为例，剖析其源码实现，同样可帮助理解其他 FDW的源码逻辑。

首先，我们需要定义 FdwRoutine。前文提到了 FdwRoutine 主要负责定义外部数据扫描的接口，接口需要自定义实现外部扫描的方法。

![img](https://pica.zhimg.com/80/v2-fdf90c010c657f02b0f5f0cc159aa914_1440w.webp)

**访问外部数据源**

定义好 FdwRoutine 之后，开始访问并扫描外部数据源。在 Postgres_fdw 中，流程也就是进入 BeginForeignScan 阶段。这一阶段主要是获取我们先前定义的外部表实例和用户信息，然后初始化并获取一个连接到远端数据源。

![img](https://pic1.zhimg.com/80/v2-9a364180c86a79206c7d01b5b6c85efc_1440w.webp)


**执行查询阶段**

获取连接后，执行查询，即进行 IterateForeignScan 阶段。这个过程的逻辑是创建一个游标[迭代器](https://zhida.zhihu.com/search?q=迭代器&zhida_source=entity&is_preview=1)(cursor)，并从cursor中持续获取数据。

![img](https://pica.zhimg.com/80/v2-10a141aa6a137d4d72a5538ae7dbef4e_1440w.webp)


当全部数据迭代或扫描完成后，我们会释放连接并关闭 cursor 等资源，通过自定义的 EndForeignScan 阶段完成。

![img](https://pic3.zhimg.com/80/v2-bae4224a2d22798e6b702c74df3bf822_1440w.webp)

**insert操作**

对于 insert 操作，例如，在本地 PostgreSQL 数据库中修改Web数据源，增加一条数据，需要访问插入Web数据的接口。此操作先进入BeginForeignInsert 阶段，任务是构造SQL语句，通过预处理语句进行初始化，做好插入准备。

![img](https://pic4.zhimg.com/80/v2-bc52a148d29e2096df847275f21506df_1440w.webp)


之后，进入 ExecuteForeignInsert 阶段，执行数据插入，主要通过预处理语句传递参数，然后发送SQL到远端执行。

![img](https://pic2.zhimg.com/80/v2-ba21268abd33da52a31248e9a05f8557_1440w.webp)


最后，EndForeignInsert 阶段负责收尾和资源清理。

![img](https://pic3.zhimg.com/80/v2-7b0d18fe630b56b030b8d1854f1a8974_1440w.webp)


**更新/删除操作**

更新和删除操作的逻辑与插入类似。首先进入BeginDirectModify阶段，进行数据修改前的准备，如构建查询语句、获取连接等。随后执行修改操作，主要通过发送参数和查询到远端来执行。

![img](https://pic4.zhimg.com/80/v2-d373b0b7b91e2a4504d013c4ae815a05_1440w.webp)
