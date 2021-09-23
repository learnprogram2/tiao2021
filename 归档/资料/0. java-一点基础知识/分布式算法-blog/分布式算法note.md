## 分布式协议和算法

### 四大基础理论

- 拜占庭将军问题
- CAP 理论
- ACID 理论
- BASE 理论

### 八大分布式协议和算法

- Paxos 算法
- Raft 算法
- 一致性 Hash 算法
- Gossip 协议算法
- Quorum NWR 算法
- FBFT 算法
- POW 算法
- ZAB 协议



## 基础: 拜占庭将军问题-共识 - ByzantineFaultTolerance

这个问题是关于叛徒的问题, 多者协商, 共同一致后才会做决策, 但是会有叛徒出现, 这就是拜占庭将军问题.

**拜占庭将军问题很好地抽象出了分布式系统面临的共识问题。**

1. **可能出现的问题:** 多instance在制作共识的时候, 可能有叛徒投反对票, 投不一样的票, 造成共识达不成或者是错的.

2. **解决方案一:** 

   **[多轮投票](https://my.oschina.net/u/4499317/blog/4791527)**, 叛徒人数m, 总instance >= 3m + 1, 那么投票m+1轮就可以把叛徒的影响投出去. 达成共识.

3. **解决方案二:** 

   **签名**, 保证通讯间的安全...后续

   **其他算法**, FBFT, Pow, Paxos, Raft, ZAB.

4. **除了拜占庭算法,** 还有不包含恶意误导的的容错: Crash Fault Tolerance CFT 算法.



## 基础: CFT问题: CAP, BASE, ACID

1. CAP

   - 一致性（Consistency）
   - 可用性（Availability）
   - 分区容错性（Partition Tolerance)

2. BASE: NoSQL的理论依据.

   基本可用 basic availability

   软状态 soft state

   最终一致性 eventually consistency.

3. ACID: DB内的事务的概念.

   可用性, availability

   一致性, consistency

   隔离性, isolation

   持久性, durability









## Paxos算法  “与其预测未来，不如限制未来” - 协商共识

1. Paxos算法: 包含两个部分
   - Basic-Paxos: 多个节点就某个值达成一致
   - Multi-Paxos: 执行多个Basic-Paxos实力, 就一系列值达成共识.

2. 三个角色:

   - 提议者: 提议一个值, 供大家投票表决. 接入和协调,接受client请求后发起商议共识.
   - 接收者: 对提议的值投票, 存储接受的值. 
   - 学习者: 直接拿到并存储投票结果.

   提议者可以是接收者. 提议只不过是发起商议请求.

3. 达成共识过程:

   - 准备阶段 prepare: 
     1. 提议者(s)准备好提案(no, v=null), 向所有的接收者发送.
     
     2. 接收者: 
     
        - 如果没有提案, 或者提案no小的的就**响应: Promise**
     
        - 有更大提案no的, 不响应
     
   - 接受阶段 accept: 
   
     1. 提议者S接收到大多数响应Promise之后, 准备(no, v). 给所有的acceptor发送propose请求.
     2. acceptor接收到propose, 再检查一下no
        - 如果比自己已经有的小, 就忽略
        - **如果no不小, 响应Accept**
   
   - Learn阶段: 
   
     proposer在acceptor统一同意决议后, 发送给所有learner.
   
     learner接收到就学习最新的了.

4. **我的理解: Pasox是大多数+最新版本的二阶段提交**

5. 这只是初步初步的理解, 接下来看的:

   https://www.jianshu.com/p/d9d067a8a086



## [Raft](https://my.oschina.net/u/4499317/blog/4926112) **一切以Leader为准** - 协商共识

**分布式系统开发首选的`共识算法`。**

1. 三个角色: 

   - follower: 普通群众, 相当于learner, 但是每个任期term有一票投给candidate.
   - candidate: 候选人, follower心跳超时就变成candidate投自己
   - leader: 管理

2. **先保证在同一时间, 集群中只有一个leader**

   **选举过程**

   1. 所有节点都是follower, 每个follower的Term=0;
   2. follower等待leader的心跳, 超时(随机间隔)之后, 变成candidate.
   3. candidate把自己的term(任期)+1, 然后投自己一票, 给大家发送投自己票请求
   4. 其他的follower接受term=1的新candidate,(只能投一次)
   5. candidate变成了leader. 

<img src="%E5%88%86%E5%B8%83%E5%BC%8F%E7%AE%97%E6%B3%95note.assets/88de001f-29e8-4c70-bcab-84c0b447e927.gif" alt="img" style="zoom: 33%;" />



3. term:

   自动增加, 推荐自己的时候就+1, 发现更高的term的时候就变为最大者. 变成follower.

   拒绝小的term的请求.

   每个follower每个term'里只有一张票.

4. 心跳超时时间是随机的

5. **我的理解: Raft是大多数+最新版本+超时候选+leader统治一切的一阶段投票**

   从Paoxs中吸取了大多数+最新版本的商议.



## [一致性哈希](https://my.oschina.net/u/4499317/blog/4943846) - 负载均衡路由寻址

 **路由寻址算法，适合简单的路由寻址场景**

1. Hash算法: 计算hash值, 然后取余

   **缺点:**  扩容很难, 都要遍历重算一遍.

2. 一致性Hash: 

   - **哈希算法**：对节点的数量进行取模运算。
   - **一致性哈希算法**：对 2^32 进行取模运算。

   **缺点:** 如果扩容什么的很难弄均匀. 容易出现冷热数据.

3. 虚拟节点:  把2^32个slot分配给别人.

   <img src="%E5%88%86%E5%B8%83%E5%BC%8F%E7%AE%97%E6%B3%95note.assets/f4f78122-3212-4bab-8ce8-a3fadbdf2527.png" alt="img" style="zoom:50%;" />





## Gossip

**Gossip (流言蜚语)协议利用一种`随机`、带有`传染性`的方式, 将信息传播到整个网络中,  实现了`最终一致性`的协议。**

普通的直接投递, 相当于API调用, 调用失败放在buffer里重新调用

反熵的概念: 通过pull和push来消除不同节点的差异, 提升相似度.

Gossip的第三种传播功能: 流行病传播: 

![](%E5%88%86%E5%B8%83%E5%BC%8F%E7%AE%97%E6%B3%95note.assets/4sxb9i1nxj.gif)









## Quorum NWR

分布式中的一致性又分为`最终一致性`和`强一致性`. **控制NWR三个参数的quorum(法定人数) 来实现可控的数据一致性**

N 表示副本数，又叫做**复制因子（Replication Factor）**。也就是说，N 表示集群中同一份数据有多少个副本，

W，又称写**一致性级别（Write Consistency Level）**，表示成功完成 W 个副本更新，才完成写操作：

R，又称**读一致性级别（Read Consistency Level）**，表示读取一个数据对象时需要读 R个副本

参数 N、W、R 的不同组合将会带来不同的一致性效果。

- N = 3，W = 2，R = 2，W + R > N，对于客户端来讲，整个系统能保证强一致性，一定能返回更新后的那份数据。
- 当 W + R <= N 时，对于客户端来讲，整个系统只能保证最终一致性，访问数据期间可能会返回旧数据。

应用: InfluxDB 企业版是时序数据库，它有四种写一致性级别





## Pow(Proof of Work) 工作量证明













## ZAB(ZooKeeper Atomic Broadcast protocol)









## FBFT(Practical Byzantine Fault Tolerance) 实用拜占庭容错































