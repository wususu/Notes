# Raft协议


## Raft节点

### 一个Raft节点可以有三种状态：

1. Follower（跟随者）
2. Candidate（候选者）
3. Leader（领导者）

### Leader Election（领导选举）

1. 所有节点一开始都是Follower
2. 如果Follower与Leader失去连接（即在定时器时间内无法接收or回复Follwer的心跳包）则变为Candidate（初始化时通过随机定时器来避免产生多个Candidate）， 一个Candidate将会要求其他节点向他投票
3. 如果一个Candidate节点获得超过节点总数一半的选票，则变为Leader，选举失败的节点变为Follower。
4. 如果没有一个Candidate节点获得超过半数的选票，则重新开始新的一轮选举

    + #### 选举过程有两种Timeout：
        1. election timeout（选举超时）： Follower等待（等待与Leader通信）的时间，若超时则转换为Candidate并开始一个新的选举任期.
        <br/>Timeout时长为150ms~300ms不等，每个节点都是随机值.
        2.  heartbeat timeout（心跳超时）: Leader与Follower心跳链接过程中有一个心跳超时，若Follower在超时时间内未收到Leader的心跳包，则自己转为Candidate发起选票请求;若收到Leader的心跳包。则重置time.


### Log Replication（日志复制）

1. 所有客户端对这个系统的请求都会先经过Leader，每个修改都会在Leader节点增加一条日志(log entry)， 但不会提交该日志，所以不会改变Leader节点当前的数据状态。
2. 接下来，Leader将该条日志通过Leader的心跳包复制到他的Follower节点，Follower收到该日志后写入自己的日志文件中，向Leader返回一条类似ACK的答复。
3. 当Leader收到绝对大多数的Follower的写入答复后就在自己的节点上提交该条日志记录，并相应客户端结果（异步），然后在通过下次心跳包通知Follower节点该条记录已被提交，Follower节点也跟着一起提交该条日志

### Consitent in Network partitions（网络分区下保持一致性）

Raft能够正确地处理网络分区（“脑裂”）问题

+ A~E五个结点，B是leader。如果发生“脑裂”，A、B成为一个子分区，C、D、E成为一个子分区。此时C、D、E会发生选举，选出C作为新term的leader。这样我们在两个子分区内就有了不同term的两个leader。这时如果有客户端写A时，因为B无法复制日志到大部分follower所以日志处于uncommitted未提交状态。而同时另一个客户端对C的写操作却能够正确完成，因为C是新的leader，它只知道D和E。

+ 当网络分区消除时,A,B能与所有节点发送心跳包，此时发现彼此term不一致，于是会以term最大的为Leader，另一个及其Follower会rollback在分区时uncommited的值成为其Follower。  （如果term比较小的那部分分区有超过总数一半的节点，已经commit，如何保持更新一致呢？）


## Raft特性：

+ 强领导者（Strong Leader）：Raft 使用一种比其他算法更强的领导形式。例如，日志条目只从领导者发送向其他服务器。这样就简化了对日志复制的管理，使得 Raft 更易于理解。

+ 领导选取（Leader Selection）：Raft 使用随机定时器来选取领导者。这种方式仅仅是在所有算法都需要实现的心跳机制上增加了一点变化，它使得在解决冲突时更简单和快速。

+ 成员变化（Membership Change）：Raft 为了调整集群中成员关系使用了新的联合一致性（joint consensus）的方法，这种方法中大多数不同配置的机器在转换关系的时候会交迭（overlap）。这使得在配置改变的时候，集群能够继续操作。



#### 相关文档
> [Raft 一致性算法论文译文](http://www.infoq.com/cn/articles/raft-paper)

> [Raf 动图](http://thesecretlivesofdata.com/raft/)

>[Raft 为什么是更易理解的分布式一致性算法](https://www.cnblogs.com/mindwind/p/5231986.html)