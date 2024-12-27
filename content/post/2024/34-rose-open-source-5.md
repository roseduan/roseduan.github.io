---
title: rose 聊开源—5 折腾开源这些年的收获
date: 2024-11-15T16:51:56+08:00
categories:
    - 开源
tags:
    - 开源项目
---

前面的几篇文章主要讲述了：

[rose 聊开源—1 你为什么需要一个开源项目](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzI0Njg1MTUxOA%3D%3D%26mid%3D2247486603%26idx%3D1%26sn%3Dc28659da0778420b8cfd947df2c7b919%26chksm%3De9b9be5adece374c9b5b79e9a1c93e0be41bd3f0c0d314db36bc3fe71d6b43a7d20ea51ceb38%26scene%3D21%23wechat_redirect)

[rose 聊开源—2 如何快速上手一个开源项目](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzI0Njg1MTUxOA%3D%3D%26mid%3D2247486869%26idx%3D1%26sn%3D5a35d7b8045681abdb33808e1c1ff2b1%26chksm%3De9b9bf44dece36521312a6a744cdb974706a2dd686855634ae4c02ba8ecf8b2e8f91669f2532%26scene%3D21%23wechat_redirect)

[rose 聊开源—3 如何开始写自己的开源项目](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzI0Njg1MTUxOA%3D%3D%26mid%3D2247486913%26idx%3D1%26sn%3D344c2eece0e7abced4164a1d5a8ee901%26chksm%3De9b9bf10dece3606c8c4d4488a89230d2925b5e50e7b1ba5932c9f799f0ee986d0480614c2a8%26scene%3D21%23wechat_redirect)

[rose 聊开源—4 如何推广自己的开源项目](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzI0Njg1MTUxOA%3D%3D%26mid%3D2247486918%26idx%3D1%26sn%3D1c2696a14750075fc45c6161da535026%26chksm%3De9b9bf17dece3601b12cffb4b7e0f0748f36cad578e88f5e075a100333134ceee874477a7b51%26scene%3D21%23wechat_redirect)

这一篇来跟大家聊一聊我自己在这几年的开源项目折腾历程中，一些收获与做得不太好的地方，给大家一些参考和启示。

> 此篇文章过后，这个系列暂告一段落，因为经其他读者反馈，可能大部分人并没有自己的开源项目，或者说也是刚起步，在这方面的经验并不是很多。
> 前面几篇文章介绍的一些经验、[项目推广](https://zhida.zhihu.com/search?content_id=250342798&content_type=Article&match_order=1&q=项目推广&zhida_source=entity)等内容，还不能够立即用得上。所以有兴趣的同学，可以按照我之前的介绍去开始参与开源，或者写自己的开源项目。
> 当大家有了更多的经验之后，后续如果有其他的感想，我会不定期再次对这个系列进行更新。

### **1、项目现状**

我在这几年的开源历程中，主要是维护了两个 Go 语言的开源项目，分别是 rosedb 和 lotusdb，目前项目的关注度，完善程度其实还是不错的了，star 数量也都上千，并且多次登上过 Github Trending 榜单。

对于个人开源项目来说，能够做到这种规模，也算是不小的成就了吧。

![img](https://pic4.zhimg.com/v2-eec8ffe2f6dcdb1517af109b09c99b1f_1440w.jpg)

![img](https://pic3.zhimg.com/v2-b7535f75523c3c1afb7f3f659b50fde8_1440w.jpg)

除了这两个项目之之外，还有其他的一些组件，比如：

wal 预写日志组件

![img](https://pic1.zhimg.com/v2-b514bfcbf4566f007667e6b478cee080_1440w.jpg)

diskhash 基于磁盘的[哈希表](https://zhida.zhihu.com/search?content_id=250342798&content_type=Article&match_order=1&q=哈希表&zhida_source=entity)

![img](https://pic1.zhimg.com/v2-bc1b6976ffa9cc7b56ca08d7f38b4612_1440w.jpg)

还有一些其他的 tutorial 项目。

比如针对数据库学习路线，项目推荐等，有 database-learning。

![img](https://pic3.zhimg.com/v2-fe59958d19267eaca7922de10618734a_1440w.jpg)

针对 go 和 rust 的学习路线，也有对应的两个开源项目：

![img](https://pic4.zhimg.com/v2-8361b2775ad6b18565efe17be0ad436d_1440w.jpg)

mini-bitcask，是 rosedb 的迷你版本，也是一个教学性质的项目。

![img](https://pic4.zhimg.com/v2-a341f79bb4bcfd2b14a12f2173e89e77_1440w.jpg)

还有 rust 的练手项目，rust-practice

![img](https://pic1.zhimg.com/v2-5c13f28ee17ed052d972b868c417e22e_1440w.jpg)

还有一些早期的项目，比如 grpc-demo，当初学习 gRPC 的时候写的一些小的练习。

![img](https://pic1.zhimg.com/v2-bf75145c3521960305e12ed02edf7338_1440w.jpg)

algo-learn，当初刷 leetcode 的一些题解记录。

![img](https://pic2.zhimg.com/v2-6a7347159c996bf97da3d67d0058d853_1440w.jpg)

### **2、对自身技术能力的提升**

折腾这么多项目，其实对自己最直观的提升就是技术方面。

针对一些学习类的项目，其实我们能够在学习的时候做一些记录，写一些文章，然后记忆模糊的时候能够随时拿出来温习，是很不错的学习方式，也能够锻炼自己总结归纳的能力。

然后去折腾一些偏实战的开源项目，需要去解决各种问题，比如从理论到实践的鸿沟跨越，需要去进行[架构设计](https://zhida.zhihu.com/search?content_id=250342798&content_type=Article&match_order=1&q=架构设计&zhida_source=entity)，系统设计。

并且一些代码细节方面需要仔细考究，对代码规范，[单元测试](https://zhida.zhihu.com/search?content_id=250342798&content_type=Article&match_order=1&q=单元测试&zhida_source=entity)，可扩展，简洁易读等方面都有很高的要求。

经过这方面的折腾，也增强了我自己的代码洁癖，这一习惯也延续到了工作当中。

### **3、对职业发展的影响**

毫不隐瞒的说，其实有了开源项目，给了我的履历非常大的加成，当然这其实也有大环境、以及运气等因素的共同影响。

因为有了开源项目，我在找工作的过程当中能够有更多的底牌，因为这是别人了解你的一种很直观的方式。开源项目代码都是公开的，一个人的技术能力能够马上得到体现，这其实就是最大的技术能力背书，不需要其他任何的虚名。

在开源项目的长期维护和运营过程当中，我也找到了自己所热爱的事情，能够有机会持续的去折腾，也是挺不错的一种状态了。

### **4、维护社群，个人影响力**

目前我维护了 rosedb&lotusdb 的两个[开源社区](https://zhida.zhihu.com/search?content_id=250342798&content_type=Article&match_order=1&q=开源社区&zhida_source=entity)群，共计有 800+ 人，slack 虽然不太活跃，但是也有 150+ 人数。

而且我也靠着在开源项目的运营，产出了一些相关的内容，发布在了自己的公众号、B 站上面，进一步完善了我的个人 ip，我自己在这方面的影响力也得到了提升。

更重要的是，这是一段很宝贵的经历，注定给我留下很深的印象。

### **5、talk 能力、写作能力**

还有就是我也会写一些相关的文章，以及录制一些视频去讲述我自己的开源项目，在这个过程中我也锻炼了自己的写作能力。

并且我也在一些平台，无论是线上还是线下，分享了很多关于开源项目，以及我自身的开源经历，这进一步提升了我面对公众的 talk 能力。

这些软技能，虽然在短期内看不到什么效果，但确实是在潜移默化的影响着我，让我能够成为一个更加全面的人。

### **6、做得不好的地方**

当然，其实在这几年的开源项目历程中，也有做的不是很好的地方。

最遗憾的一点便是对于开源项目团队维护得很差，这其实是有多方面的原因吧。

因为外部贡献者，每个人的目的、状态、思维都是不一样的，很难去保持长期的合作关系，大多数人都只是在短期之内有一定的参与热情，但是久而久之可能因为各种原因就搁置了。

我其实也尝试去采用各种办法去推动一个长期合作的开源贡献团队，但最核心的一点是，如果没有金钱维系的话，每个人都会产生倦怠的情绪，靠热情是无法持久的。

当然还有一点就是项目本身的上限并不高，创新性比较有限，就算做到极致，也只能是一个无法盈利的简单项目，没有前瞻性的优势，同样也无法让其他贡献者保持长期的参与意愿。

只不过，我认为这倒也无可厚非，因为我们折腾的项目，本身也只是符合我们自己当前阶段的状态，以及技术和认知能力，能够在这个范围之内尽力做到最好，就已经很不错了。