---
title: 我的编程故事—6 转岗 & rosedb持续维护
date: 2023-08-27 18:43:33
categories:
    - 关于我
tags:
    - 我的编程故事
---

上一次说到，毕业一年多之后，我经历了一次跳槽，从 Java 也转到了 Go 语言，从事普通的后端开发工作。

在工作之余，我还是会在自己的业余时间写写 rosedb 项目，当然这仅仅是一些兴趣而已，并且在 Github 基本没有获得任何的关注。

后来，我在自己的公众号上写了一篇文章，名为[我写了一个数据库。。。](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzI0Njg1MTUxOA%3D%3D%26mid%3D2247484475%26idx%3D1%26sn%3D5d8d7bb5821729f62739a7be315ee3ac%26chksm%3De9b9b6eadece3ffc785b65d4ece5df41c3650d3c4fba037759ea9c1463764351aac0088df724%26scene%3D21%23wechat_redirect)，稍微有点标题党，但也不算太偏题，就是这篇文章带来了一些初始流量的积累。

那时候我的公众号也就几百人，写出来之后，我就在很多的群里都转发，期望获得更多的曝光，可能是因为这个标题比较吸引人，还是获得了不少的关注，并且承蒙不少人的支持，点了一些 star，这样 rosedb 的关注后面就越来越多了。

基于此，我还在 B 站录制了一个系列视频，讲述了这个项目的大致结构，以及一些设计的要点。

![img](https://pic3.zhimg.com/80/v2-2ee6b4576641f66fe481c5b63e0810e6_1440w.webp)

然后我还到 Go 夜读做了一期分享，讲述了这个项目的大致设计和源码。

**[https://www.bilibili.com/video/BV1ih411h7yC](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1ih411h7yC)**

几波操作一下来，rosedb 就吸引了更多人的关注，后面就频繁的登上了 Github 的 Trending 榜单，当时我还专门写了文章做个纪念。

[rosedb 连续两天上榜](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzI0Njg1MTUxOA%3D%3D%26mid%3D2247484615%26idx%3D1%26sn%3Dde582d35dc435e89ca8e9cd9a58d8838%26chksm%3De9b9b616dece3f00f2f7e45a96a4e52701e7138f3ce96a21d1e2b3ce5cf6b14ea8b40e6adc84%26scene%3D21%23wechat_redirect)

然后在 21 年 6 月份的时候，也就是开源七个月之后，rosedb 的 star 数就到了 1000。

[七个月，从零到一千](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzI0Njg1MTUxOA%3D%3D%26mid%3D2247484729%26idx%3D1%26sn%3D586e1f0f09314fd4f1a5ca3fab226d83%26chksm%3De9b9b7e8dece3efea904d674a159f51dcc539462477de908c483407bc2f99f5caff6505dfe6e%26scene%3D21%23wechat_redirect)

获得了这些关注之后，对于我自己其实是非常大的鼓励，当然也感觉到很意外，完全没想到会达到这样的效果，这也刺激了我在这个领域去深耕，继续钻研。

当然我的工作还是普通的后端开发，直到有一次，我参与了公司内部的一次技术分享，让我了解到公司的基础架构部，是有在做分布式 KV 相关的内容的，这也引起了我极大的兴趣，于是想着能不能内部转岗过去，关于这次转岗的经历，我之前也写过文章记录，这里就不再赘述了。

[转岗记](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzI0Njg1MTUxOA%3D%3D%26mid%3D2247485133%26idx%3D1%26sn%3De269f156294c9de5a5f6f235d81ca700%26chksm%3De9b9b41cdece3d0af9c8214ebcaf6cf40e5a9d27986cd90df3027417171c3f8175daae28228f%26scene%3D21%23wechat_redirect)

最近其实也有很多人咨询我，关于如何从业务开发转到其他基础架构相关方向的，但是我的经历具有很大的偶然性，并不具有特别的参考价值，因为当时以我自身的水平，如果专门出去找存储相关的工作的话，可能还是很困难的。

但还是可以提取一些通用的建议给更多有同样需求的人，首先就是兴趣非常重要，这能够驱使你即使在下班后，或者其他闲暇时间能够投入更多的精力来做自己感兴趣的事情，在做这些事情的事情，在前期可能是见不到任何成效的，并且可能也并没有什么直接的收益。

但是如果能够凭借兴趣和热爱坚持下去，或者也能看到一些曙光。

然后比较重要的就是寻找正向反馈，比如我在做出 rosedb 项目之后，不遗余力的去宣传，尽可能的去获取更多的关注，当有更多的人关注到我的项目，我就能获得更多的成就感，才能够更好的坚持下去。

现在回过来看，如果没有当时的坚持，或许 rosedb 也不会发展到现在，我也不会转到存储，然后一步步到现在做数据库内核。

所以如果你有自己感兴趣的事情，尽可能去折腾和尝试，并且不断宣传出去让更多人知道，获得正向反馈，这样你的职业方向或许就能够得到更多、更好的发展。

欲知我转岗到分布式存储之后的事情，以及 rosedb 的后续，且听下回分解。

