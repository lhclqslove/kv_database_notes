# 算法概述

  raft是工程上使用较为广泛的强一致性、[去中心化](https://so.csdn.net/so/search?q=%E5%8E%BB%E4%B8%AD%E5%BF%83%E5%8C%96&spm=1001.2101.3001.7020)、高可用的分布式协议。在这里强调了是在工程上，因为在学术理论界，最耀眼的还是大名鼎鼎的Paxos。但Paxos是：少数真正理解的人觉得简单，尚未理解的人觉得很难，大多数人都是一知半解。本人也花了很多时间、看了很多材料也没有真正理解。直到看到raft的[论文](https://web.stanford.edu/~ouster/cgi-bin/papers/raft-atc14)，两位研究者也提到，他们也花了很长的时间来理解Paxos，他们也觉得很难理解，于是研究出了raft算法。
 可以通过[动画](http://thesecretlivesofdata.com/raft/)快速了解协议

#  leader election

raft协议中，一个节点任一时刻处于以下三个状态之一：

-   leader
-   follower
-   candidate
![[Pasted image 20221018203734.png]]
所有节点启动时都是follower状态；在一段时间(一般是100-300ms)内如果没有收到来自leader的心跳，从follower切换到candidate，发起选举；如果收到majority的造成票（含自己的一票）则切换到leader状态；如果发现其他节点比自己更新，则主动切换到follower。

总之，系统中最多只有一个leader，如果在一段时间里发现没有leader，则大家通过选举-投票选出leader。leader会不停的给follower发心跳消息，表明自己的存活状态。如果leader故障，那么follower会转换成candidate，重新选出leader。

## term

   从上面可以看出，哪个节点做leader是大家投票选举出来的，每个leader工作一段时间，然后选出新的leader继续负责。这根民主社会的选举很像，每一届新的履职期称之为一届任期，在raft协议中，也是这样的，对应的术语叫_**term**_。

![](https://img-blog.csdnimg.cn/img_convert/7c90e1746aebb941a3a6a1b315cb0e2c.png)

   term（任期）以选举（election）开始，然后就是一段或长或短的稳定工作期（normal Operation）。从上图可以看到，任期是递增的，这就充当了逻辑时钟的作用；另外，term 3展示了一种情况，就是说没有选举出leader就结束了，然后会发起新的选举，后面会解释这种_split vote_的情况。

## 选举过程详解 

上面已经说过，如果follower在_election timeout_内没有收到来自leader的心跳，（也许此时还没有选出leader，大家都在等；也许leader挂了；也许只是leader与该follower之间网络故障），则会主动发起选举。步骤如下：

-   增加节点本地的 _current term_ ，切换到candidate状态
-   投自己一票
-   并行给其他节点发送 _RequestVote RPCs_
-   等待其他节点的回复

   在这个过程中，根据来自其他节点的消息，可能出现三种结果

1.  收到majority的投票（含自己的一票），则赢得选举，成为leader
2.  被告知别人已当选，那么自行切换到follower
3.  一段时间内没有收到majority投票，则保持candidate状态，重新发出选举

   第一种情况，赢得了选举之后，新的leader会立刻给所有节点发消息，广而告之，避免其余节点触发新的选举。在这里，先回到投票者的视角，投票者如何决定是否给一个选举请求投票呢，有以下约束：

-   在任一任期内，单个节点最多只能投一票
-   候选人知道的信息不能比自己的少（这一部分，后面介绍log replication和safety的时候会详细介绍）
-   first-come-first-served 先来先得

   第二种情况，比如有三个节点A B C。A B同时发起选举，而A的选举消息先到达C，C给A投了一票，当B的消息到达C时，已经不能满足上面提到的第一个约束，即C不会给B投票，而A和B显然都不会给对方投票。A胜出之后，会给B,C发心跳消息，节点B发现节点A的term不低于自己的term，知道有已经有Leader了，于是转换成follower。

   第三种情况，没有任何节点获得majority投票，比如下图这种情况：

![](https://img-blog.csdnimg.cn/img_convert/d0b9b031c8964d00ab536c58977f4044.png)

   总共有四个节点，Node C、Node D同时成为了candidate，进入了term 4，但Node A投了NodeD一票，NodeB投了Node C一票，这就出现了平票 split vote的情况。这个时候大家都在等啊等，直到超时后重新发起选举。如果出现平票的情况，那么就延长了系统不可用的时间（没有leader是不能处理客户端写请求的），因此raft引入了randomized election timeouts来尽量避免平票情况。同时，leader-based 共识算法中，节点的数目都是奇数个，尽量保证majority的出现。

  #  log replication
     当有了leader，系统应该进入对外工作期了。客户端的一切请求来发送到leader，leader来调度这些并发请求的顺序，并且保证leader与followers状态的一致性。raft中的做法是，将这些请求以及执行顺序告知followers。leader和followers以相同的顺序来执行这些请求，保证状态一致。
## 请求完整流程 
  当系统（leader）收到一个来自客户端的写请求，到返回给客户端，整个过程从leader的视角来看会经历以下步骤：

-   leader append log entry
-   leader issue AppendEntries RPC in parallel
-   leader wait for majority response
-   leader apply entry to state machine
-   leader reply to client
-   leader notify follower apply log

  可以看到日志的提交过程有点类似两阶段提交(2PC)，不过与2PC的区别在于，leader只需要大多数（majority）节点的回复即可，这样只要超过一半节点处于工作状态则系统就是可用的，而2PC需要所有节点回复OK才能提交，否则要回滚。

  那么日志在每个节点上是什么样子的呢

![](https://img-blog.csdnimg.cn/img_convert/0706503779fb39e474d3c67e6c052750.png)

  不难看到，logs由顺序编号的log entry组成 ，每个log entry除了包含command，还包含产生该log entry时的leader term。从上图可以看到，五个节点的日志并不完全一致，raft算法为了保证高可用，并不是强一致性，而是最终一致性（CAP理论：一致性（C）->在分布式系统中的所有数据备份，在同一时刻是否同样的值），leader会不断尝试给follower发log entries，直到所有节点的log entries都相同。

  在上面的流程中，leader只需要日志被复制到大多数节点即可向客户端返回，一旦向客户端返回成功消息，那么系统就必须保证log（其实是log所包含的command）在任何异常的情况下都不会发生回滚。这里有两个词：commit（committed），apply(applied)，前者是指日志被复制到了大多数节点后日志的状态；而后者则是节点将日志应用到状态机，真正影响到节点状态。
## safety
在上面提到只要日志被复制到majority节点，就能保证不会被回滚，即使在各种异常情况下，这根leader election提到的选举约束有关。在这一部分，主要讨论raft协议在各种各样的异常情况下如何工作的。

  衡量一个分布式算法，有许多属性，如

-   safety：nothing bad happens,
-   liveness： something good eventually happens.

  在任何系统模型下，都需要满足safety属性，即在任何情况下，系统都不能出现不可逆的错误，也不能向客户端返回错误的内容。比如，raft保证被复制到大多数节点的日志不会被回滚，那么就是safety属性。而raft最终会让所有节点状态一致，这属于liveness属性。

  raft协议会保证以下属性
  ![[Pasted image 20221018211542.png]]
  ### Election safety

  选举安全性，即任一任期内最多一个leader被选出。这一点非常重要，在一个复制集中任何时刻只能有一个leader。系统中同时有多余一个leader，被称之为脑裂（brain split），这是非常严重的问题，会导致数据的覆盖丢失。在raft中，两点保证了这个属性：

-   一个节点某一任期内最多只能投一票；
-   只有获得majority投票的节点才会成为leader。

  因此，**某一任期内一定只有一个leader**。

在leader执行更新命令时，需要超过majority节点响应才会更新成功，这样即使出现网络分区情况产生了两个leader(不同term)，那么有只有最新leader才能更新成功。

### leader completeness vs elcetion restriction

  leader完整性：如果一个log entry在某个任期被提交（commited apply?），那么这条日志一定会出现在所有更高term的leader的日志里面。这个跟leader election、log replication都有关。

-   一个日志被复制到majority节点才算committed
-   一个节点得到majority的投票才能成为leader，而节点A给节点B投票的其中一个前提是，B的日志不能比A的日志旧。下面的引文指处了如何判断日志的新旧
## State Machine Safety
 前面在介绍safety的时候有一条属性没有详细介绍，那就是State Machine Safety：

> State Machine Safety: if a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index.

  如果节点将某一位置的log entry应用到了状态机，那么其他节点在同一位置不能应用不同的日志。简单点来说，所有节点在同一位置（index in log entries）应该应用同样的日志。但是似乎有某些情况会违背这个原则：

![](https://img-blog.csdnimg.cn/img_convert/dc287de73993a26ae43581b29f54280a.png)

  上图是一个较为复杂的情况。在时刻(a), s1是leader，在term2提交的日志只赋值到了s1 s2两个节点就crash了。在时刻（b), s5成为了term 3的leader，日志只赋值到了s5，然后crash。然后在(c)时刻，s1又成为了term 4的leader，开始赋值日志，于是把term2的日志复制到了s3，此刻，可以看出term2对应的日志已经被复制到了majority，因此是committed，可以被状态机应用。不幸的是，接下来（d）时刻，s1又crash了，s5重新当选（备注：节点只会投给比自己log更新的节点，这里的log包含未提交的log!!而不是已提交的log，因此在这里s5能够当选），然后将term3的日志复制到所有节点，这就出现了一种奇怪的现象：被复制到大多数节点（或者说可能已经应用）的日志被回滚。

  究其根本，是因为term4时的leader s1在（C）时刻提交了之前term2任期的日志。这种情况跟在term2提交黄色节点2是有区别的，这种情况下不含黄色节点2的节点是有可能当选的，因为不含黄色节点的节点有可能在term2-term4之间增加了更新的节点，根据选举规则（每个节点只投给比自己日志新的节点，这里的日志可以为未提交的日志）其他节点是有可能选举成功的，为了杜绝这种情况的发生：

> **Raft never commits log entries from previous terms by counting replicas**.  
> Only log entries from the leader’s current term are committed by counting replicas; once an entry from the current term has been committed in this way, then all prior entries are committed indirectly because of the Log Matching Property.

  也就是说，某个leader选举成功之后，不会直接提交前任leader时期的日志，而是通过提交当前任期的日志的时候“顺手”把之前的日志也提交了，具体怎么实现了，在log matching部分有详细介绍。那么问题来了，如果leader被选举后没有收到客户端的请求呢，论文中有提到，在任期开始的时候发立即尝试复制、提交一条空的log

> Raft handles this by having each leader commit a blank no-op entry into the log at the start of its term.

  因此，在上图中，不会出现（C）时刻的情况，即term4任期的leader s1不会复制term2的日志到s3。而是如同(e)描述的情况，通过复制-提交 term4的日志顺便提交term2的日志。如果term4的日志提交成功，那么term2的日志也一定提交成功，此时即使s1crash，s5也不会重新当选。