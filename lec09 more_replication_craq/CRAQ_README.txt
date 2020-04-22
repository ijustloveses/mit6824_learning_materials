## Introduction

### 传统的 chain replication method

- 把所有要 replicate 数据的节点组织为一个 chain
- 写操作从第一个节点开始，每一个节点写好以后，传给下一个节点，直到最后一个节点也写入，称为 committed
- 然后写入成功的回复从最后一个节点依次往前发送，直到第一个节点接收到成功写入的回复后，返回给客户端
  当然，也可以由 tail 直接回复给客户端，更快一些
- 读操作从最后一个节点请求，只有所有节点都更新好后才会返回新数据，否则返回老数据

优点 (vs raft)：
- 客户端的读写请求分开由 head 和 tail 分别处理，比起 raft 的话，raft leader 要处理所有请求，负担更大
- CR 的 head 节点只给后续一个服务器发送写请求，而 raft 要群发
- CR 的读请求只会涉及到一个服务器，而 raft 的读请求同样要求 majority，故此涉及更多的服务器
- Failure Recovery 比起 Raft 更简单，我们会在 CRAQ 算法中介绍

主要的问题：
- 读操作集中在最后一个节点上，这样导致最后一个节点成为 hotspot，压力过大

### CRAQ - Chain Replication with Apportioned Queries

- 保持强一致性的同时，提供读操作的低延迟和高吞吐。对于主要是读的系统，性能甚至匹敌只保证最终一致性的系统
- 方法是 apportioned queries: 把读操作分布到所有节点上
- 代价是读操作可能会 stale，读取到过时的数据，CRAQ 支持配置最大的 staleness acceptence 参数
- 底层使用了 ZK(ZooKeeper) 作为 configuration manager，当然原则上也可以用 raft、paxos 等
- 一些更高级的优化还在进行当中
  + mini-transaction 支持：多个操作的 Atomically 执行
  + multicast 支持：对 large-object update 的写入优化
  
CRAQ v.s. Raft or ZooKeeper
- Raft：强一致性，但是只能从 leader 读、写
- ZooKeeper：妥协了读请求的一致性，换取可以在所有节点做读取
- CRAQ：强一致性，而且可以在所有节点上做读取
- CRAQ 比 Raft or ZooKeeper 的优点

看上去 CRAQ 比 Raft 和 ZooKeeper 都要好啊，是这样么？不是，也有缺点
- Raft & ZooKeeper 都只要 majority quorum 达成共识就可以了
- CRAQ 要所有节点都达成共识，而且只要有一个节点慢了，整个写请求的响应就会被拖慢，木桶效应
- 当一个节点挂掉，在被 ZK 判断出来并踢出集群之前，写操作只能 hold 等待


## System Model

### Object-based Storage 系统的操作原语和一致模型

操作原语
- write(objID, V) 
- read(objID) -> V

consistency model
- Strong Consistency：读操作总是读取到最新的写入
- Eventual Consistency：可能读取到 stale，但最终所有 replicas 会写入最新数据，之后不再会读取 staled 数据

### 传统的 chain replication 

- 流程如 Introduction 部分所示，就不赘述了。显然满足强一致性，committed 之后最后一个节点一定返回最新数据

### CRAQ

读、写操作原语的实现
- 每个节点都保存数据的多个版本，对于每个 object，版本都是单调递增的，每个版本都有一个 clean/dirty 标识
- 当写请求沿着 chain 传递到某个节点，该节点就收到待写入 object 的一个新版本，并把这个新版本加入版本 list
  + 如果节点不是 tail (最后一个节点)，版本标识为 dirty，然后会把写入请求往下一个节点传递
  + 如果是 tail，标识为 clean，该版本号视为 committed，于是 tail 把 committed 信息往前面的节点传递
- 当 committed 信息被前面的节点接收，该节点就把对应版本标识为 clean，并把之前的版本都安全删除掉
- 允许 chain 中的任何一个节点处理读请求，这样极大的增加了读请求的承载能力
- 当节点接收到一个读取请求
  + 如果版本号最高的最新版本标识为 clean，那么直接返回这个版本的数据
  + 否则向 tail 节点发送请求，询问 tail 节点当前最新 committed 的版本号，然后返回这个版本号的数据
    * 即使询问到最新版本号后、返回客户端之前，tail 节点马上又 committed 了更新的版本，这并不算违反一致性

性能分析
- Read-Mostly Workloads：多数节点都是 clean 的，可以直接返回结果，故此极大的提升了读请求的性能
- Write-Heavy Workloads：
  + 多数节点都是 dirty 的，故此都要咨询 tail 节点，查询最新的 committed 版本，这样 tail 会成为 hotspot
  + 但是这个咨询操作可以高性能完成，故此性能仍然比单点处理读请求的传统 chain replication 更高
  + 我们甚至可以进一步的限制 tail，让其只处理咨询请求，而不处理读请求

CRAQ 支持三种类型的一致性模型
- Strong Consistency：正如上面所介绍的流程，保证了强一致性
- Eventual Consistency
  + 有些系统允许返回 staled 数据，以换取性能(高吞吐量和低延迟)，那么我们可以放弃掉向 tail 咨询的逻辑
  + 每个节点直接返回该节点上最新 committed 版本的数据，即使有更新的 dirty 版本存在也不会去 tail 询问
  + 这样，客户端从不同节点上取相同的数据，可能会得到不同版本的结果，有些结果是 staled 的，这是不好的
  + 好的地方是，客户端永远从某一个节点读取某个数据的话，这个数据的版本一定单调递增的，最终得到最新结果
- Eventual Consistency with Maximum-Bounded Inconsistency
  + 这个和上面正相反，它会返回节点最新版本的数据，不管这个版本是 clean 的还是 dirty 的
  + 但是需要有一个限制，所谓 maximum inconsistency period 以内的最新数据，通常由时间或者版本来定义

Failure Recovery in CRAQ
- 前提 1：客户端和各个服务器之间的请求，以及各个服务器相互的请求，都可以通过 Response 来知道请求是否成功
- 前提 2：CRAQ 中每个节点需要知道其在 chain 上的前、后节点（predecessor and successor）以及 Head and Tail
- 如果 Head failed，那么其 successor 就会取而代之，committed write 一定不会丢，中途的 write 也不会丢
  + 最多就是 Head 刚处理完某请求，还没发给下个节点就挂了，此时客户端一定不会收到回复，只要 resend 即可
- 如果 Tail failed，更简单了，其 predecessor 取而代之，任何 write 都不会丢
  + 最多就是新的 Tail 节点需要判断一下，是不是有某个 write 自己已经确认，但是还未往回确认，可以回溯确认之
- 中间节点的新增、替换或者删除逻辑就和双向链表相同操作的逻辑完全一样
  + 此时被替换节点之前的节点需要重发 recent writes (比如全部 dirty write) 给之后的节点

Split-Brain？
- 一个场景：后面的节点如果收不到 Head 的请求，怎么知道 Head 是不是挂掉了？如果取而代之，而 Head 还在？
- 如果让节点自己判断节点的状态是不合适的，处理起来并不容易，而且容易引发 split-brain 和网络分区
- 实际的做法是使用 ZooKeeper 作为配置管理工作，以 higher level 管理整个集群的节点状态

可能的优化点
- Mini-Transaction on CRAQ
  + 希望支持多个请求的原子性操作，全部成功或者全部失败
  + 如果多个请求都是针对同一个 Key ，也就是同一个 object，此时是很 trivial to implement
    * 包括 Prepend/Append、 Incr/Decr、 Test-and-set，这样的多请求可以作为一个 batch 操作
	* 代价是写请求的吞吐量会被影响，如果 head 需要等待 transaction 中的某些 keys 变成 clean 状态
  + 如果多个请求都在同一个 chain 上
    * Sinfonia 实现了一个 mini-transaction 算法，使用了 optimistic 2-phase commit 协议
	* 代价是写请求的吞吐量会被影响，如果 head 需要等待 transaction 中的某些 keys 变成 clean 状态
  + 如果多个请求分布在不同的 chains 上
    * 同样用 optimistic 2-phase commit 协议实现，只要在各个 chains 的 heads 上锁住 transaction 相关的 keys
- Lowering Write Latency with Multicast
  + 写请求的时候，不采用链式传播，而是一次性 Multi-cast 给各个节点
  + 只有一小部分 metadata message 需要通过链式传播，而 commit 确认可以由 Tail Multi-cast 给所有节点
  

## ZooKeeper Coordination Service in CRAQ

### 作用 
- tracking 和监控 group membership，看哪个节点挂掉了，以便进行 Failure Recovery
- ZooKeeper 的 watch 机制保证了 CRAQ 节点会在集群添加和删除节点时收到通知
- store meta-data，同样当 meta-data 发生变化时，所关注的节点也会收到通知

ZooKeeper 通常没有对用于多 DataCenters 时的情况进行优化
- ZooKeeper 的一般实现中，并不知道 DataCenters Topology & Hierarchy，其消息都是在单层 WAN 上传播
- CRAQ 的实现中，跨 DC 的 ZooKeeper traffic，每个 DC 维护自己的 local ZooKeeper
- 保证节点总是收到 local ZK nodes 的通知，对其他的节点，只接收其相关的 global ZK nodes 的通知

### 实现

Datacenter 中的节点信息
- 一个 CRAQ 节点会创建一个 Ephemeral file in /nodes/dc_name/node_id，文件内容为 ip port 信息
- 同时会 watch /nodes/dc_name，这样就保证了当这个 DC 增加或者减少了服务器节点时，其所有节点都会收到通知

chain 信息
- 当一个节点收到请求要创建一个 chain 时，创建一个文件在 /chains/chain_id
- 该文件的内容由 CRAQ chain 的部署策略所决定，注意其中只有 chain 的配置信息，而不会有 chain 当前的节点列表
- 所有要加入到这个 chain 的节点，都会 watch 这个文件，这样就可以收到 metadata 发生变化的通知















