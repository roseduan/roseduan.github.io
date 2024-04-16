---
title: Postgres 源码学习 1—Postgres 源码编译和 debug
date: 2024-04-16T10:51:56+08:00
categories:
    - 数据库
    - Postgres
tags:
    - Postgres 源码学习
---

## docker 环境

这里我使用了一个纯净的 Ubuntu 环境来进行演示，为了方便，使用了 docker。

如果你有其他的物理机，或者云服务器，都是可以的，Postgres 支持多种平台编译，如果你是非 Ubuntu 环境，可以自行查阅相关的资料进行编译安装，步骤都是大同小异的。

我使用了 Ubuntu 20.04 版本的镜像作为演示：

![img](https://pic4.zhimg.com/v2-9920d521d2801c948be231bb6c21fef7_b.jpg)

使用镜像启动容器：

```Bash
docker run -itd --name <container-name> --privileged <image id>
```

进入环境：

```Bash
docker exec -it <container-name | container id> /bin/bash
```

## 创建用户

最好不要在 root 用户下编译和安装 Postgres，这里我们可以新建一个用户

```Go
useradd <username> -m -s /bin/bash
```

切换到新的用户环境中。

```Go
su <username>
```

## 安装依赖

安装 Postgres 编译所需的依赖（这里是摘取了 `Greenplum` 的安装依赖，可能包含了一些没必要安装的，但肯定是涵盖了 Postgres 需要的依赖，所以全部安装上也没啥问题）。

```Go
sudo apt-get update
sudo apt-get install -y \
        bison \
        ccache \
        cmake \
        curl \
        flex \
        git-core \
        gcc \
        g++ \
        inetutils-ping \
        krb5-kdc \
        krb5-admin-server \
        libapr1-dev \
        libbz2-dev \
        libcurl4-gnutls-dev \
        libevent-dev \
        libkrb5-dev \
        libpam-dev \
        libperl-dev \
        libreadline-dev \
        libssl-dev \
        libxerces-c-dev \
        libxml2-dev \
        libyaml-dev \
        libzstd-dev \
        locales \
        net-tools \
        ninja-build \
        openssh-client \
        openssh-server \
        openssl \
        pkg-config \
        python3-dev \
        python3-pip \
        python3-psycopg2 \
        python3-psutil \
        python3-yaml \
        zlib1g-dev
```

## 执行编译

拉取 Postgres 的源代码，并进入到 postgres 代码目录中。

如果是拉取最新版本的代码，可以从 Github 上获取：

```Rust
git clone https://github.com/postgres/postgres.git
```

如果想要获取对应版本的源代码，则可以从 Postgres 官网中下载：

地址：https://www.postgresql.org/ftp/source/

Postgres 有非常多的编译选项，详情可以参考官方文档：https://www.postgresql.org/docs/current/install-make.html#CONFIGURE-OPTIONS

我们这里只使用最简单的编译方式即可。

```Go
CFLAGS=-O0 ./configure --prefix=/home/roseduan/pg-install --enable-debug
```

我们关闭了编译器的优化，方便后续的调试，并且打开了 debug 模式。

`--prefix` 指定编译后的二进制目录的位置，这里不指定也是可以的，默认是在 `/usr/local` 下面。

Configure 之后，如果没有错误产生的话，则执行编译并安装：

```Go
make -s -j`nproc` install
```

编译安装之后，得到了二进制目录，可以将 bin 目录加入到 PATH 环境变量中，如果嫌麻烦，可以加入到 $HOME 目录中的 .bashrc 或者 .zshrc（取决于你的 sh 是什么），这样下次登录就不用重复设置了。

```Bash
export PATH=/<posgres-install-dir>/bin:$PATH
```

## 初始化 DB

上述步骤完成后，可以使用 init 命令来初始化 postgres 的数据目录。

```Bash
pg_ctl -D <pg 数据目录路径> init
```

![img](https://pic1.zhimg.com/v2-f9e9d9cef9434ca8bd773d97b3cd02e0_b.jpg)

初始化完成后，直接启动 postgres 的服务即可。

```Bash
pg_ctl -D pg-data start
```

![img](https://pic2.zhimg.com/v2-191fb93dec2aeba77400d4629119fc0d_b.jpg)

启动之后，可以查看 postgres 的进程状态。

![img](https://pic3.zhimg.com/v2-bb5be63d2a81ad70e73dbe6bce87f2f2_b.jpg)

也可以通过 psql 命令连接到数据库中：

```Bash
psql postgres
```

## 如何 Debug

有了源码环境之后，其实 Debug 调试就比较简单。

使用 psql 登录之后，后台会启动一个工作进程来服务于这个客户端的请求，可以通过 `pg_backend_pid()` 方法查看进程 id。

![img](https://pic4.zhimg.com/v2-3d2dd60c40de405ae4fa4d27d16f03f3_b.jpg)

这里我的进程 id 是 1857，直接通过 `gdb -p 1857` 即可对该进程进行 Debug。

我们可以在 gdb 中设置一个断点，比如 Postgres 的简单查询命令都会走 `exec_simple_query` 方法，可以直接对这个方法打断点，然后在客户端任意执行一个 select 语句，就会到 gdb 的断点中了：

![img](https://pic1.zhimg.com/v2-84457b76271ef309f399ded2cedaf474_b.jpg)

## 参考资料

https://www.postgresql.org/docs/current/installation.html
