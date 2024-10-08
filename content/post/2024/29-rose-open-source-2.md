---
title: rose 聊开源—2 如何快速上手一个开源项目
date: 2024-09-11T16:51:56+08:00
categories:
    - 开源
tags:
    - 开源项目
---

在前面的一篇开源项目系列中，主要介绍了目前开源项目蓬勃发展的态势，并且拥有一个开源项目，对我们个人履历、职业发展等都有非常多的好处。

这一次就来跟大家分享一下，面对一个开源项目，我们应该如何上手，快速看懂项目的源代码，并且能够参与到开源项目的社区中。

首先，如果我们通过 Github 找到了一些项目，如何判断这个项目是否符合我们的需求，是否值得我们花时间去投入呢?

在我看来，有几个比较重要的点需要关注。

一是项目是否对新人友好，是否有比较完善的文档和上手教程。其实一般做的比较规范的项目，都会有这部分内容，会在项目的 README 文档中，说明上手项目的整体流程，以及如何参与到社区当中等等。

如果连这些基本的内容都没有的话，说明社区的建设并不是很好，或者说只是一些年久失修的项目，想要上手的话难度会很大。

二是，如果想要参与到开源项目当中的话，则需要关注这个项目是否接受外部贡献者。因为有的项目实际上是公司内部在维护，对外部贡献者可能并没有完善的流程、规范等等。

第三点，可以关注是否有适合的 Good First Issue，一般来说，开源社区会将一些适合新人上手的 Issue 标记为 Good First Issue，从这些 Issue 开始上手会是一个不错的选择。

在这个基础之上，如果你已经选择了一个不错的开源项目，如何快速上手呢？我认为搞懂下面几个问题是比较关键的。

首先，面对一个开源项目，其实最基础的问题，就是搞懂这个项目的基本背景，就是这个项目是基于什么契机建立的，比如是为了解决什么实际的问题，或者是对某个系统进行优化，或者是使用新的语言进行重写等等。

通过这些背景，需要搞懂项目主要的功能是什么，用来解决什么样的问题。

其次，需要搞懂项目的大致架构，当然在初期不太可能将每个模块都能够完全搞明白，特别是如果一个项目比较庞大的情况下，但是尽量搞清楚架构图中每个模块的大致功能，先从宏观上有个大致的了解。

这个理论背景的基础之上，我们需要动手实践起来。

比如，首先需要能够编译并且运行这个项目，这是最基础的一个步骤。并且针对项目当中的一些核心流程和方法，能够去进行调试，一步一步的去查看执行过程中的状态转换。

比如针对一个数据库项目，或者存储引擎的项目，其实最基础的流程就是，建立表，并且向表中插入数据，然后能够从表中读取数据，这是一个最基础的步骤。

当然，这个链路当中可能又涉及到了很多的不同的内部组件和方法等等，需要我们在调试的过程中去慢慢熟悉。

以上是我对如何入门开源项目的一点建议，我在 B 站也发布了一个视频，使用一个 Rust 项目作为示例，用上面我提到的步骤，一步步调试 Rust 项目，向大家详细演示了如何搞懂一个开源项目。

![](https://pica.zhimg.com/80/v2-1439355f6784c468ef62c088770b5306_1440w.webp)

感兴趣可以观看下：

https://www.bilibili.com/video/BV1Qc4YevEVy
