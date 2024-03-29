---
title: 分布式 KV 面试汇总
date: 2024-03-04T22:51:56+08:00
categories:
    - KV 存储
    - 分布式 KV
tags:
    - KV 存储
    - 分布式 KV
---


> 本文选自《从零实现分布式 KV》课程的加餐文章。 从零开始，手写基于 raft 的分布式 KV 系统，课程详情可以看这里：[https://av6huf2e1k.feishu.cn/docx/JCssdlgF4oRADcxxLqncPpRCn5b](https://link.zhihu.com/?target=https%3A//av6huf2e1k.feishu.cn/docx/JCssdlgF4oRADcxxLqncPpRCn5b)

1. ## 在简历上如何写这个项目？

**项目概述**

基于 MIT 6824 课程 lab 框架，实现一个基于 raft 共识算法、高性能、可容错的分布式 KV 存储系统，保证系统的一致性和可靠性。

**设计细节**

- 设计基于 Raft 一致性算法的分布式系统架构。
- 支持分布式数据存储和检索的 KV 存储引擎，采用 Raft 协议确保数据的强一致性。
- 实现数据分片和自动故障转移机制，以实现系统的高可用性和容错性。
- 使用 Go 语言编写，工程级代码可靠性和简洁性。

**结果**

- 参照 Raft 论文使用 Golang 实现了领导选举、日志同步、宕机重启和日志压缩等主要功能。熟悉 Raft 算法的基本原理和实现细节，熟悉 Golang 并发编程和分布式调试。
- 实现了一个高性能的分布式键值存储系统，保证数据的一致性和可靠性。
- 通过所有代码测试，在负载测试中表现出良好的性能和稳定性，能够有效地应对并发访问和故障情况。

1. ## 可能的面试问题&回答

> 以下我们每个节点统称为 Peer，面试官可能会叫节点、副本(Replica)、Node 等等术语，记得和面试官对齐就好。

### Raft 主要在什么场景中使用？

通常有两种用途：

1. **元信息服务**，也称为配置服务（configuration services）、分布式协调服务（coordinator services）等等。如 etcd。用以追踪集群中元信息（比如副本位置等等）、多副本选主、通知（Watch）等等。
2. **数据****复制****（Data replication）**。如 TiKV、CockroachDB 和本课程中的 ShardKV，使用 Raft 作为数据复制和冗余的一种手段。与之相对，GFS 使用简单的主从复制的方法来冗余数据，可用性和一致性都比 Raft 要差。

> 注：在分布式系统中，数据指的是外界用户存在系统中的数据；元数据指的是用户维护集群运转的内部信息，比如有哪些机器、哪些副本放在哪里等等。

### Raft 为了简洁性做了哪些牺牲（即有哪些性能问题）？

1. **每个操作都要落盘**。如果想提高性能可能需要将多个操作 batch 起来。
2. **主从同步数据较慢**。在每个 Leader 和 Follower 之间只允许有一个已经发出 AppendEntries；只有收到确认了，Leader 才能发下一个。类似 TCP 中的“停等协议”。如果写入速度较大，可能将所有的 AppendEntries Pipeline 起来性能会好一些（即 Leader 不等收到上一个 AppendEntries 的 RPC Reply，就开始发下一个）
3. **只支持全量快照**。如果状态机比较小这种方式还可以接受，如果数据量较大，就得支持增量快照。
4. **全量快照同步代价大**。如果快照数据量很大，每次全量发送代价会过高。尤其是如果 Follower 本地有一些较老的快照时，我们只需要发增量部分即可。
5. **难以利用多核**。因为 log 只有一个写入点，所有操作都得抢这一个写入点。

### Raft 在选举时是不能正常对外提供服务的，这在工程中影响大吗？

不太大，因为只有网络故障、机器宕机等事件才会引起宕机。这些故障的发生率可能在数天到数月一次，但 Raft 选主在秒级就能完成。因此，在实践中，这通常不是一个问题。

### 有其他不基于 Leader 的共识协议吗？

原始的 Paxos 就是无主的（区别于有主的 MultiPaxos）。因此不会有选举时的服务停顿，但也有代价——每次数据同步时都要达成共识，则数据同步代价会更大（所需要的 RPC 更多，因为每次同步消息都是两阶段的）。

### 论文提到 Raft 只在非拜占庭的条件下才能正常工作，什么是拜占庭条件？为什么 Raft 会出问题？

“非拜占庭条件”（Non-Byzantine conditions）是指所有的服务器都是“宕机-停止”（ fail stop）模型（更多模型参见[这里](https://ddia.qtmuniao.com/#/ch08?id=系统模型和现实)）：即每个服务器要么严格遵循 Raft 协议，要么停止服务。例如，服务器断电就是一个非拜占庭条件，此时服务器会停止执行指令，则 Raft 也会停止运行，且不会发送错误结果给客户端。

拜占庭故障（Byzantine failure）是指有些服务器不好好干活了——可能是代码因为有 bug，也可能是混入了恶意节点。如果出现这种类型节点，Raft 可能会发送错误的结果给客户端。

### 通常来说，Raft 的所有节点都期望部署在一个数据中心吗？

是的。跨数据中心的部署可能会有一些问题。有些系统，如原始的 Paxos（由于是 Leaderless）可以跨数据中心部署。因为客户端可以和本地的 Peer 进行通信。

### 如果发生网络分区，Raft 会出现两个 Leader ，即脑裂的情况吗？

不会，被分到少数派分区的 Leader 会发现日志不能同步到大多数节点，从而不能提交任何日志。一种优化是，如果一个 Leader 持续不能联系到多数节点，就自动变为 Follower。

### 当集群中有些 Peer 宕机后，此时的“多数派”是指所有节点的多数，还是指存活节点的多数？

所有节点的多数。比如集群总共有五个 Peer，则多数派永远是指不低于 3 个 Peer。

如果是后者，考虑这样一个例子。集群中有五个 Peer，有两个 Peer 被分到一个分区，他们就会认为其他三个 Peer 都宕机了，则这两个 Peer 仍然会选出 Leader ，这明显是不符合预期的。

### 选举超时间隔选择的过短会有什么后果？会导致 Raft 算法出错吗？

选举超时间隔选的不好，只会影响服务的可用性（liveness），而不会影响正确性（safety）。

如果选举间隔过小，则所有的 Follower 可能会频繁的发起选举。这样，Raft 的时间都耗在了选举上，而不能正常的对外提供服务。

如果选举间隔过大，则当老 Leader 故障之后、新 Leader 当选之前，会有一个不必要的过长等待。

### 为什么使用随机超时间隔？

为了避免多个 Candidate 一直出现平票的情况，导致一直选不出主。

### Candidate 可以在收到多数票后，不等其余 Follower 的回复就直接变成 Leader 吗？

可以的。首先，多数票就足够成为主了；其次，想等所有票也是不对的，因为可能有些 Peer 已经宕机或者发生网络隔离了。

### Raft 对网络有什么假设？

网络是不可靠的：可能会丢失 RPC 请求和回复，也可能会经历任意延迟后请求才到达。

但网络是有界的（bounded）：在一段时间内请求总会到达，如果还不到达，我们就认为该 RPC 丢失了。

### votedFor 在 requestVote RPC 中起什么作用？

保证每个 Peer 在一个 Term 中只能投一次票。即，如果在某个 term 中，出现了两个 Candidate，那么 Follower 只能投其中一人。

且 votedFor 要进行持久化，即不能说某个 Peer 之前投过一次票，宕机重启后就又可以投票了。

### 即使服务器不宕机，Leader 也可能会下台吗？

是的，比如说 Leader 所在服务器可能 CPU 负载太高、响应速度过慢，或者网络出现故障，或者丢包太严重，都有可能造成其他 Peer 不能及时收到其 AppendEntries，从而造成超时，发起选举。

### 如果 Raft 进群中有过半数的 Peer 宕机会发生什么？

Raft 集群不能正常对外提供服务。所有剩余的节点会不断尝试发起选举，但都由于不能获得多数票而当选。

但只要有足够多的服务器恢复正常，就能再次选出 Leader，继续对外提供服务。

### 请简单说说 Raft 中的选举流程？

所有的 Peer 都会初始化为 Follower，且每个 Peer 都会有一个内置的选举超时的 Timer。

当一段时间没有收到领导者的心跳或者没有投给其他 Candidate 票时，选举时钟就会超时。

该 Peer 就会由 Follower 变为 Candidate，Term++，然后向其他 Peer 要票（带上自己的 Term 和最后一条日志信息）

其他 Peer 收到请求后，如果发现 Term 不大于该 Candidate、日志也没有该 Candidate 新、本 Term 中也没有投过票，就投给该 Term 票。

如果该 Peer 能收集到多数票，则为成为 Leader。

### 如果所有 Peer 初始化时不为 Follower、而都是 Candidate，其他部分保持不变，算法还正确吗？

正确，但是效率会变低一些。

因为这相当于在原来的基础上，所有 Peer 的第一轮选举超时是一样：同时变为 Candidate。则谁都要不到多数票，会浪费一些时间。之后就又会变成原来的选举流程。

### 如何避免出现网络分区的 Peer 恢复通信时将整体 Term 推高？

**问题解释**：如果某个 Peer （我们不妨称其为 A）和其他 Peers 隔离后，也就是出现了网络分区，会不断推高 Term，发起选举。由于持续要不到其他 Peer 的票，因此会持续推高 Term。一旦其之后某个时刻恢复和其他 Peer 的通信，而由于 Term 是 Raft 中的[第一优先级](https://av6huf2e1k.feishu.cn/docx/ZYMGdQMA2ouPnKxfh56cUvpwnAg#BkIfd6wS8o2x1LxstRYcr82PnSe)，因此会强迫当前的 Leader 下台。但问题是，由于在隔离期间日志被落下很多，Peer A 通常也无法成为 Leader。最终结果大概率是原来的 Leader 的 Term 被拉上来之后，重新当选为 Leader。有的人也将这个过程形象的称之为“惊群效应”。

**解决办法**：**PrevVote**。每次 Candidate 发起选举时，不再推高 Term，但是会拿着 Term+1 去跟其他 Peer 要票，如果能要到合法的票数，再去推高 Term（Term+1）。而如果能要到多数票，其实就保证该 Candidate 没有发生网络隔离、日志是最新的。如果要不到多数票，就不能推高 Term，这样会保证发生了网络隔离的 Peer 不会一直推高自己的 Term。

### Raft 和 Paxos 有什么区别？

首先，Raft 和 Paxos 都是共识协议，而所有的共识协议在原理上都可以等价为 Paxos，所以才有共识协议本质上都是 Paxos 一说。

如 Raft 论文中提到的，Raft 是为了解决 Paxos 理解和实现都相对复杂的问题。将共识协议拆成两个相对独立的过程：领导者选举和日志复制，以降低理解和实现的复杂度。当然，如果要想工程可用，Raft 的优化也是无止境的大坑，也并非像论文声称的那么简单。因此，有人说，Raft 看起来简单只是因为论文叙述的更清楚，而非算法本身更为简洁。

Raft 其实是和 Multi-Paxos 等价，因为 Paxos 只解决单个值的共识问题。

Raft 和 Paxos 的角色分法也不太相同，Raft 的每个 Peer 都可以有  Leader，Candidate 和 Follower 三种状态；而 Paxos 是将系统分为 Proposer，Acceptor 和 Learner 三种角色，实现时可以按需组合角色。

在 Paxos 中，一旦某个日志在多数节点存在后就可以安全的提交；但在 Raft 中，不总是这样，比如一条日志在多数节点中存在后，但不是当前 Leader 任期的日志，也不能进行直接提交；而只能通过提交当前任期的日志来间接提交。

在Paxos 在选举时，Leader 可能需要借机补足日志，但 Raft 中选举过程完全不涉及日志复制（这也是 Raft 进行拆分的初衷）。这是因为 Raft 只允许具有最新日志的 Candidate 成为 Leader，而 Paxos 不限制这一点。

在 Paxos 中，允许乱序 commit 日志，而 Raft 只允许顺序提交。

在 Paxos 中，每个 Peer 的 term 是不一致的，全局自增的；在 Raft 中 term 是每个 Peer 独立自增的，但需要对齐。

更多区别，可以参考文末给出的资料。

### Raft 在工程中有哪些常见的优化？

由于领导者选举是个低频操作，主要 IO 路径优化还是集中在日志同步流程上。

- **batch**：Leader 每次攒一批再刷盘和对 Follower 进行同步。降低刷盘和 RPC 开销。
- **pipeline**：每次发送日志时类似 TCP 的“停-等”协议，收到 Follower 确认后才更新 nextIndex，发送后面日志。其实可以改成流水线式的，不等前面日志确认就更新 nextIndex 继续发后面的。当然，如果后面发现之前日志同步出错，就要回退 nextIndex 重发之前日志——而原始版本 nextIndex 在**同步阶段**是单调递增的。
- **并行 append**：Leader 在 append 日志到本地前，就先发送日志给所有 Follower。

### 请简单描述基于 raft 的分布式 KV 系统的架构？

一个基于 raft 的分布式 KV 系统，实际上是由一组使用 raft 算法进行状态复制的节点组成。客户端会选择将请求发送到 Leader 节点，然后由 Leader 节点进行状态复制，即发送日志，当收到多数的节点成功提交日志的响应之后，Leader 会更新自己的 commitIndex，表示这条日志提交成功，并且 apply 到状态机中，然后返回结果给客户端。

以上是单个 raft 集群的分布式 KV 系统架构。

如果系统中数据量较大，一个 raft 集群可能无法承受大量的数据，性能也会受到影响。因此还基于此设计了可分片的分布式 shardkv 系统。shardkv 由多个 raft 集群组成，每个集群负责一部分 shard 数据。

Shard 到 raft 集群的映射关系，保存在独立的配置服务中。

### 分布式系统中读数据的流程是什么样的，如何优化？

为了保证线性一致性，目前的实现是利用了 raft 算法，将读请求传入到 raft 并进行状态复制，这样能够保证读到的数据一定是最新的。

但是由于读请求也进行了一次日志复制，执行效率会受到影响，业界常用的两种优化方式是 ReadIndex 和 LeaseRead。

> https://cn.pingcap.com/blog/linearizability-and-raft/
>
> https://www.sofastack.tech/blog/sofa-jraft-linear-consistent-read-implementation/

### 客户端发送请求的时候，如何知道集群中的 Leader 节点是哪个？

在没有任何前置条件的情况下，客户端会轮询集群中的每个节点并发送请求，如果非 Leader 节点收到请求，会返回一个错误给客户端。客户端然后挑选下一个 server 进行重试，直到得到了正确的响应。

然后会将 Leader 节点的 id 保存起来，下次发送请求的时候，优先选择这个节点发送。

### 如果 raft 集群的 Leader 节点发生故障，客户端如何处理？

对于一个可容错的分布式 KV 系统，需要能够应对这种故障发生，并且在多数节点正常的情况下，需要依然提供服务。

得益于 raft 共识算法的特性，在某个节点故障后，其他节点会由于收不到心跳消息而超时，并重新发起选举。

所以客户端会在得不到正常响应的时候轮询重试，直到 raft 集群中的 Leader 节点重新选举完成并提供正常服务。

### 如何处理客户端的重复请求？

如果客户端的请求已经提交，但是 server 返回的过程中结果丢失，那么客户端会发起重试，导致这个请求在状态机中被执行了两次，会违背线性一致性。

因此我们需要保证客户端的请求只能被状态机应用一次，我们可以维护一个去重哈希表，客户端 ID + 命令 ID 组成一个唯一的标识符，如果判断到命令是已经被执行过的，则直接返回对应的结果。

### Shardkv 的问题：为什么需要对分布式 KV 系统进行分片？

一是单个 raft 集群实际存储数据的引擎是单机的，能够存储的数据量有限。二是在不分区的情况下，所有数据的读写请求都会在一个分片中，这在并发量较大的情况下可能存在一定的瓶颈。

如果对数据做了分区，那么不同分区之间的数据读写请求是可以并行的，这能够较大的提升 KV 系统的并发能力。

### Shardkv 的配置怎么保存？

Shardkv 的配置是单独保存在一个服务中，客户端会向这个服务发起请求，查询 key 所属的 shard 应该在哪个 raft 集群中，并向这个集群发起请求。

配置服务也需要高可用特性，因为配置服务如果发生故障不可用的话，那么整个分布式 kv 服务都会无法提供服务，因此也使用 raft 算法保证高可用，构建了一个单 raft 集群来存储配置信息。

### Shard 数据如何迁移？

启动一个后台定时任务，定期从配置服务中获取最新的配置，如果检测到配置发生变更，则变更对应 shard 的状态，标记为需要进行迁移。

同时启动另一个后台定时任务，定期扫描 shard 的状态，如果检测到需要进行迁移的 shard，则发送消息，通过 raft 模块进行同步。然后在 Leader 节点中处理 shard 迁移的请求，将 shard 数据从原所属的 raft 集群中迁移到新的集群中。

### Shard 迁移的时候，客户端的请求会受到影响吗？

如果客户端请求的 key 所属的 shard 并没有在迁移中，那么可以正常提供服务。

否则，说明客户端请求的 key 在迁移中，则返回错误，让客户端进行重试。

### 如果有并发的客户端请求和 shard 迁移请求，应该怎么处理？

客户端请求和 shard 迁移请求的确存在并发情况，如果处理顺序不一致，会违背线性一致性。

我们将 shard 迁移的请求也传入到 raft 模块进行同步，这样和客户端的请求是一致的，利用 raft 的一致性来保证两种不同请求的先后顺序，前面的执行结果一定对后续的请求可见。

### 如果某个 Shard 已经迁移了，那么它还会占存储空间吗？

不会，我们实现了 shard 清理的完整流程，会启动一个后台定时任务，定期扫描 shard 的状态，如果检测到 shard 是需要进行清理的，则也会发送 shard 清理消息进行处理。

## 参考资料

1. Paxos vs Raft：https://ics.uci.edu/~cs237/reading/Paxos_vs_Raft.pdf
2. TiKV Raft 的优化：https://zhuanlan.zhihu.com/p/25735592
3. Raft FAQ：https://pdos.csail.mit.edu/6.824/papers/raft-faq.txt
