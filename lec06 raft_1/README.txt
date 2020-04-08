Raft 论文和课程理解

背景
======

Raft 是一个使得分布式系统支持容错处理的一致性算法

看一下到目前为止已经学过的系统
- MapReduce 在 workers 之间 replicates computation，但是有 master 的单点故障风险
- GFS 在 chunkservers 之间 replicates data，但是依赖于 master 挑选 primaries，且 master 单点
- VMware FT 会在主、从之间 replicates service，但依赖于 test-and-set 机制来挑选 primary
- 以上三个系统都依赖于一个单个实体 make critical decisions，从而避免或者处理 split brain

split brain 问题
- 现在有两个客户端和两个 replicas 服务器 [C1, C2, S1, S2]
- 假设 client C1 能访问 S1, 但不能访问 S2，请问，是否 C1 能够只访问 S1 就够了
  + 如果此时 S2 crashed 了, 那么 C1 必须也只能只访问 S1 了，这也就是所谓的容错机制
  + 如果之后 S2 恢复了，但是网络问题使得 C1 还是无法连接 S2，但是另一个客户端 C2 能访问 S2
    * 此时，问题就来了，S1 和 S2 独立提供服务，倒是容错了，但是整个系统分成两个 partitions
    * (C1, S1) 和 (C2, S2) 之间不能相互访问和同步数据
- 如何解决 network partition 的问题？
  + 人工干预
  + FT's test-and-set server 来挑选其中一个作为 primary
  + 配置更好的服务器和网络，保证万无一失
  + GFS 的 primary lease timeout 机制，master 永远等待 lease 结束之后再重新指定 primary

随着研究的深入，我们发现奇数个服务器 + majority vote 机制是关键
- 做任何决策之前，需要经过 majority 的投票，比如 3 个服务器中至少两个达成一致
- 这样，从容错的角度看，对于 2*f +1 个服务器，我们容忍 f 个服务器出错甚至 crash
- 如何避免 split brain? 奇数保证至多一个分区包含 majority，打破了偶数个服务器产生的对称性
— 此时，经常也称为 "quorum" systems，因为对于 3 来说，2 也叫 quorum
- 一个核心关键点，在于即使 majority partition 成员发生变化，那么至少有一个成员是重复的
  这样该成员就能把之前的 majority 的决策带给变化之后的 partition 的所有成员，保证了状态延续

在 Raft 之前，有两个 partition-tolerant replication schemes 在 1990 年左右被发明出来
- Paxos and View-Stamped Replication (VSR)，尤其是 Paxos 被广泛使用
- Paxos 的概念过于复杂，不利于理解和实现
— Raft 的实现其实更接近于 VSR，而且 Raft 的论文本身也是很好的理论介绍

Paxos 的问题
- Paxos 过于复杂，难于理解，难于实现
  + Paxos 整体分为两个阶段
  + 阶段 1) 服务器间对一个 log 达成一致的决策，生成一个 replicated log entry，称 single-decree Paxos 
  + 阶段 2) 合并多个上面达成的决策，最终形成一系列的 log，称 multi-Paxos
  + Leslie Lamport 的论文仅详细的描述了第一个阶段，而没有详细说明第二个阶段的细节
  + 第一个阶段又可以细分为两个子阶段，而且几乎不能以很直觉的方式来理解
- Paxos 很难以在真实环境中来正确的实现
  + 首先是由于论文没有对第二个阶段做详细解释，故此不同的人会有不同的实现
    * Chubby 也实现了 Paxos-like 算法，但是并没有公布详细的细节
  + Paxos 这种独立处理单个 log entry，再 melding 在一起的方法增加了额外的复杂性，不如 append 更直观
  + Paxos 主要依赖 peer-to-peer 的对称性方法，并辅以 weak leader 提升效率


Raft overview
================

### 简单介绍

- Raft 会首先选举一个 leader，然后由 leader 全权控制 replicated logs
- 具体的 leader 会从 client 接受 log entries，复制到各服务器上，并适时告知后者执行这些 log entries
- leader 极大的简化了 replicated logs 的管理，比如
  + leader 可以独自决定在哪里放置新的 log entry，而不需要和其它服务器进行协商
  + 数据流方面，永远都是从 leader 到其他服务器，其他服务器必须强制性的 agree with leader
  + 简化了state space，减少了不确定性的 state，比如 logs 是不允许留空的 (not allowed to have holes)
- Safety 方面
  + 只要任何一个服务器已经执行了一个 log entry，那么其他服务器必须在同样位置执行相同的 log
  + 保证在所有 non-Byzantine 条件下 (如网络延迟、分区、丢包、重复消息、乱序等) 不会返回错误结果
  + 以上也是设计 Raft Safety 时的核心目标，只有满足了这个目标，才能达到模拟单服务器的程度

服务器处于3种状态之一
- follower：完全被动，不会发起请求，只回应 leader & candidate；如果被客户端请求，直接转发给 leader
- leader：系统的主导者，处理和回应所有客户端的请求
- candidate：当 follower 无法连接到 leader，会转为 candidate，并发起选举
`
  From \ To     Follower   			 		Candidate				          Leader
  Follower      --                          election timeout，发起选举        --
  Candidate     发现 leader / new term      同上                              收到多数票，获选成功
  Leader        发现更高 term 的服务器      --                                --
`	

### Raft 的架构和流程

Raft 本身是一个库，依附于某个分布式的服务器系统。我们举个例子，说是 K/V 个服务器
- 3 个 replicas 服务器，每个服务器上面都是一套 K/V 存储
- k/v layer 和 state 的下层是 raft layer 和 raft logs，Raft 被部署在每个 replica 服务器上

我们画一下服务器架构，以及请求处理的 Time Diagram

                     Leader                    Follower                 Follower
Client:          -------------------       -------------------       -------------------
 ^ 1|  Put(K,V)  |                 |       |                 |       |                 |
 |  |----------> | K/V: {k1: v1,   |       | K/V: {k1: v1,   |       | K/V: {k1: v1,   |
 |               |       k2: v2,   |       |       k2: v2,   |       |       k2: v2,   |
 ----------------|        ....     |       |        ....     |       |        ....     |
            5    | 2|   }     ^    |       |      }     ^    |       |      }     ^    |
                 |--|---------|----| 3     |------------|----|       |------------|----|
                 |  V  Raft:  |5   |-----> |     Raft:  | later      |     Raft:  | later
                 | [op1|op2|op3..] |     4 | [op1|op2|op3..] |       | [op1|op2|op3..] |
                 |                 | <-----|                 |       |                 |
                 |-----------------|       |-----------------|       |-----------------|
                                          

1. 客户端发送 Put/Get "command" 到 k/v layer in leader
2. leader 把 command 添加到 Raft log
3. leader 发送 AppendEntries RPCs 给所有的 followers
   followers add command to log
4. leader 等待 majority 个回复消息 (包含他自己，对上面的例子来说，只需要一个 Follower 回复)
   此时，log entry 被正式提交 "committed"
   其实，所谓 committed 就是表示这个 command 的 log entry 已经不会因为 failure 而丢失了
   进而，这个不会丢失是指一旦能够有 majority 个服务器组成分区，那么就一定能恢复状态
   那么，如果无法恢复 majority 以上的服务器的分区，其实还是会丢失，比如所有服务器都 crash 了
5. leader 执行 command, 然后回复给客户端

之后，leader "piggybacks" commit info in next AppendEntries，followers 执行 entry 中的命令

对于未回复 AppendEntries 的 followers (crash、slow、丢包等)，leader 还会负责重发，直到 log entry 被存储

很显然，followers 的状态和 leader 的状态会在一定时间窗口内不完全一致，但是最终会达成一致


### 一些要点分析和设计

##### 为何要使用 logs?

- logs 把 commands 之间的顺序记录了下来，而且这个顺序在 majority 服务器上是一致的
  + log 存储 tentative commands 直到收到 leader 的 committed 指令
  + replicas 从故障恢复的时候，保持在一个完全一致的执行 execution 顺序上，保证 followers 最终状态一致
  + log 存储 commands 以便 leader must re-send to followers (follower 可能在下个 term 变为 leader)

- 无论是 log entries 还是 log 中 commands 的执行位置，Leader 和 followers 之间都可能有一些不一致
  + leader 先添加 log entry，再下发给 followers，故此有个时间窗口不一致，更何况此时 leader 可能 crash
  + leader 收到 majority log entry append 回复后执行命令，再通知 followers 执行命令，同上会不一致
  + followers 之间由于 log entry append、执行命令的时机差别，是否发生 crash 情况，都可能会发生不一致
  + good news: they'll eventually converge to be identical

##### 上面流程中涉及的 Raft 的 interface 设计

Start(command) (index, term, isleader) 接口
- 由 k/v server's Put()/Get() RPC handlers 所调用，从 k/v layer 调用到 Raft layer
- 只有 leader 才会处理 Start() 接口请求
  + leader 执行流程中的 step 2/3，完成自己的 log entry append 后，发送 AppendEntries RPCs 请求
- Start() 返回，不必等待 AppendEntries RPC 的回复，故此 k/v 得到返回值后，不能立即向客户端回复
  因为，此时 Raft layer 只完成了 step 2/3，自己可能立即 crash，也可能无法得到 majority 服务器的回复
  也就是说，k/v layer 得到 Start() 的返回值时，并无法知道这个 command 是否达到了 commit 的阶段
- k/v layer's Put()/Get() 则会接着以同步阻塞的方式等待 command commit 的完成, 方法是等待 applyCh
  一旦得到了 commit 的信息，说明 Raft layer 达成一致，于是执行这个命令，leader 还会把结果返回客户端
- 由于 leader 可能会 crash，command 可能会丢失，故此客户端必须要实现 re-send 的机制
- 通过该接口的返回值，k/v 层能够做出一些状态的判断
  + isleader: false if 服务器不再是 leader, 客户端必须 re-send 给另外的服务器
  + term: currentTerm, 协助调用方检测是否 leader 后来被降级 demoted 了
  + index: 本消息中的 command 被放到 log entry 的哪个位置，这个位置会在 ApplyMsg 中确定所 commit 的命令
  
AppendEntries 接口
- 这个很简单，就是 leader 把命令加到自己的 log entry 之后，告知所有的 follower 也执行相同的 append 操作
- 只有 leader 才能调用本接口，故此各个服务器可以根据 AppendEntries 得知谁是当 term 的 leader
- 同时，该接口还起到一个 heartbeat 心跳检测的作用，此时请求不携带任何 log entries 信息

注：前面介绍了，没有专门的 commit 接口，commit 是 piggybacks 在 AppendEntries 中
  
ApplyMsg(Index, Command) 接口
- leader 或者 follower 收到 leader commit 的请求后，确信系统达成一致，就发送本消息到各自的 applyCh 中
- k/v layer 正在等待这个消息，一旦收到，那么就可以执行 index 位置对应的 command，更新 k/v state
- 如果是 leader，还会把结果返回给客户端


### leader election 算法

为什么需要一个 leader?
- 理论上并不需要，paxos 的最初一个版本的论文中，也没有 leader
- 但是，实现一个 leader 有助于简化问题，并更好地保证所有 replicas 执行相同的命令，且遵循一致的顺序
- leader 带来的 leader election 的问题，但是在每个请求中都减少了 replica peers 之间状态比较的流程

Raft terms 任期
- 每次有 new leader 被选举，就增加一个 new term
- a term has at most one leader; might have no leader
  比如 majority 服务器都已经 crash，或者多个 candidates 几乎同时发起候选的请求导致没人得到多数票等等
- term 有助于系统中的服务器 follow 最新的 leader

Raft 中服务器在什么情况下会发起 leader election?
- 当它在一个 "election timeout" 窗口后还没有收到当前 leader 的心跳时
- 它会增加其自己的 currentTerm, 进入 candidate 态，投票给自己，并通过 RequestVote 把投票发往各个服务器
- 其他服务器收到投票后，会根据 term 按先来后到的顺序进行投票，只投给第一个服务器
- 注: 以上过程可能发起一个并不需要的选举，比如只是 leader 卡了一下，leader election 也没关系
- 注: 此时，原来的 leader 可能还健在，并认为自己仍然是 leader
- 退出 candidate 态的条件：成功当选、其他服务器成功当选、一段时间后还没有新 leader 当选
  + 成功当选的标准：majority 服务器都回复说投票给该 candidate，当然 term 必须相同
    当选后，立即发送 AppendEntries 心跳请求，维护其权威，并防止其他服务器发起新的选举
  + 当 candidate 还未达到当选条件，此时收到一个心跳请求，而且请求的 term 大于或者等于其自身的 term
    那么，说明其他服务器当选了。该 candidate 退出 candidate 重新成为 follower
	如果请求的 term 小于其 term（可能是原来的服务器重新上线），那么它忽略掉这个请求，继续等待投票结果
  + 如果多个 followers 几乎同时成为 candidates，那么可能会 split votes，无人当选
    那么，candidates 会等待到下一个 election timeout 过期，然后继续进行下一轮选举，term 会继续被增加  

如何保证每个 term 最多只有一个 leader?
- leader 必须要通过 majority 服务器的支持，majority 保证了最多一个 leader
- 每个服务器对每个 term 只能投一票
  + if candidate, 也就是选举的发起者，那么就投票给自己
  + 如果不是 candidiate，那么投票给所收到的第一个投票发起者

Raft 的每个服务器如何获知最新 term 的 new leader?
- new leader 是获得 majority 投票的，故此投票的服务器肯定知道
- 其他的服务器会在 AppendEntries 心跳请求中得到一个更高的 term，故此获知 new leader 是谁

一次选举未必一定选出一个 new leader，选举可能失败
- 可能只有少于半数的服务器还活着
- 同 term 内有多个 candidates 瓜分选票，导致没有人得到 majority 的支持
- 如果当期 term 选举失败，那么下个 timeout 会发生新一轮的选举，更高 term 数。而之前 term 的候选者退出

那么，Raft 如何避免瓜分选票（split votes）?
- 每个服务器都挑选一个随机的 election timeout，随机性避免了服务器之间的对称性，避免 bad case 总是出现
- 每个服务器一旦收到 AppendEntries 或者其他服务器的选举请求，就立即刷新随机的 election timeout
- 希望有足够的时间来在下次 timeout expires 之前完成选举
- 一旦选举成功，其他服务器就会看到新 leader 的 AppendEntries 请求，从而放弃选举
- 一个核心就是如何选取 election timeout 值呢？
  + 至少也要有几个 heartbeat 请求的 intervals，以避免错过了一次心跳就要重新选举这种浪费时间的情况发生
  + 同样也要长到使得 candidate 能够在下次 timeout 之前完成选举
  + 但是也不能太长了，否则一旦发生故障，系统会中断服务很久后才能进入选举流程
  + 论文中 election timeout 设为 150～300 ms

选举发生后，如果原来的 leader 还没有意识到新的选举会怎样？
- 这种情况发生的可能性：
  + 可能原 leader 还未收到选举的消息 (选举请求的 term 会更高，于是 leader 意识到后会退出 leader 身份)
  + 可能原来的 leader 身处一个 minority 的 network partition
- new leader 之所以当选，是因为获得了 majority 的支持
  + 所以原来的 leader (with old term) 无法得到 majority 的 AppendEntries 回复，因此也无法 commit 和执行
  + 故此不会发生 split brain 的情况，但是 a minority partition 仍然可能听从原来 leader 的 AppendEntries
  + 故此，majority & minority 两个分区之间可能会有 log entries 上的差别，只有 majority 分区能够执行命令

  
### ensuring identical logs despite failures

前面说到，majority & minority partition 之间可能有 log entries 上的差别，那么如何一致化呢？

##### 一些 log entries 的场景分析

Scenario 1.     Server      Slot.12      Slot.13
                S1          term.3       
                S2          term.3       term.3
                S3          term.3       term.3
				
以上场景可能出现:
- 可能是 follower S1 掉线
- 也可能是 leader 比如说 S3 在成功发送给一个 follower 比如 S2 AppendEntries 之后 crash，而没给 S1 发送

Scenario 2.     Server      Slot.12      Slot.13      Slot.14      Slot.15
                S1          term.3                        
                S2          term.3       term.3       term.4       
                S3          term.3       term.3       term.5       
				
这种场景也可能出现：
- 我们看到 Slot 12 & 13 和前面的 scenario 一致，故此应该是前面两种情况之一
- 由于 term 数发生了变化，那么一定出现了 leader crash 而重新选举的情况，那么对应第二种情况
- 于是，S3 为 term.3 的 leader，发送 AppendEntries 请求给 S2 后 crash 下线，故此 S1 未收到消息
- S2 和 S1 进行 leader election，S2 获得全部两票，成为 new leader，term 增加到 4
- S3 恢复，并承认 S2 为新的 leader，并把自己的 term 更新为 4
- 客户端发起新请求，S2 把命令添加到自己的 log entry Slot.14 上，该命令的 term 为 4
- leader S2 在还未发送 AppendEntries 给 S1 & S3 之前也 crash 了，于是 S1 & S3 并未改变 log entry 
- S1 和 S3 进行 leader election，S3 获得全部两票，成为 new leader，term 增加到 5
- 客户端发起新请求，S3 把命令添加到自己的 log entry Slot.14 上，该命令的 term 为 5，然后 crash。达成！

对于 Scenario 2，我们看看目前各个服务器上 log entry slots 上的状态
- Slot.12 很简单，大家都一致，不会造成任何问题
- Slot.13 上 S2 & S3 一致，如果 S2 或 S3 当选新 leader，就只要让 S1 追加即可；否则 S1 当选可能有问题
- 注意，此时 Slot.13 上的 log 所对应的 command 可能是 commit 的，也可能没有 commit，这是无法推测的
- 甚至 Slot.12 上的 log 对应的 command 都有可能没有 commit，S3 还未接收到 S1 & S2 的回复就先 crash 了
  + 比如 Slot.12 和 13 两个命令由不同客户端发送，而且间隔时间非常非常之短
  + Slot.12 上的命令由 S3 发给了 S1 & S2，然后还未收到回复，或者收到了回复，但是还未执行或执行完命令
  + Slot.13 上的命令由 S3 发给了 S2，但在发送 S1 之前 S3 crash 掉了
- 由于 slot.12 & 13 中的命令是有可能已经 commit 执行了，我们认为这两个 slots 可能是需要保留下来的
- Slot.14 上的 S2 & S3 的 log 可以删除，因为它们必然没有被 commit 执行
  也可以保留其中的一条，但不能两条都保留，因为都保留的话，违背了 Safety 原则，造成了 log entries 不一致
- 到此，我们无法确认一些动作是否已经 commit 执行，也无法确知应该在相互矛盾的动作中选择哪条
  + 也就是说，leader crash 后，可能某些客户端的请求会被重复执行，有些客户端的请求会被丢掉
  + 本部分重点是如何保证各服务器间 log entry 的一致性，暂时先不 care 客户端相关的问题
  
Scenario 3.     Server      Slot.12      Slot.13      Slot.14      Slot.15
                S1          term.5       term.6       term.7          
                S2          term.5       term.8       
                S3          term.5       term.8

这种场景也可能出现：
- S1 在 term 6 成为新 leader，然后在收到 Slot.13 中命令时 crash，并未发送给 S2 & S3
- S1 reboot 然后又在 term 7 成为新 leader，然后在收到 Slot.14 中命令时 crash，并未发送给 S2 & S3
- S2 或者 S3 当选 new leader，S2 & S3 两个服务器都执行了 Slot.13 中 term 8 的命令，然后 S1 reboot
- 然后所有服务器 crash，然后又全部 reboot

##### Raft 保持 log entries 一致的原则和方法

原则和目标
- Raft 保证已经 committed 的 log entries 是持久化的，而且最终将会被所有可用 available 的服务器执行

- leader 会记录目前 commit 的最大的 log entry index，且会把该 index 带到后续 AppendEntries 请求中
- follower 从 AppendEntries 请求中读取该 index，然后执行该 index 对应的 log entry 并执行其中的命令

Raft Log Matching Property
- Raft log mechanism 中两个重要的属性，称为 Raft Log Matching Property
  + 如果两个服务器的 logs 中，两个 log entries 有相同的 index (位置) 和 term，那么其中的命令一定也相同
    * 这是因为 leader 在其 term 中，在同一个 index 位置上，最多只会创建一个 log entry
	* 如果有其他 leader 在这个 index 位置上创建了一个其他的 log entry，那么 term 就一定不相同
	* leader 上的 log entries 不会改变它们的位置，因为 leader 只 append log entries，而不会删除和覆盖！
  + 如果上面的条件满足，那么不止这个 index 位置，在该 index 之前的所有位置中的命令也一定都相同
    * leader 的 AppendEntries 请求在发送 new log entry 时，还会带上前一个位置的 term 和 index
	* follower 收到请求，需要核查其中前一个位置的 term 和 index 是否和自己 log 中的一致
	* 如果是一致的，那么就接受请求，进行 new log entry appending；否则就会拒绝这个 AppendEntries 请求
	* 根据数据归纳法，该 Property 一定成立，这非常巧妙！

- 由上，如果 leader 从未 crash 的话，各个服务器的 logs 一定最终一致，客户端也只能看到 leader 上的状态
- 反过来，如果服务器间有 logs 的不一致，某 index 上 log entries 有不同的 term，一定发生过 leader crash
  + follower 可能丢掉一些 log entries，因为它没收到老 leader 的 AppendEntries，而新 leader 收到了
  + follower 可能多一些 log entries，因为它收到了老 leader 的 AppendEntries，而新的 leader 没收到
  + 还可能两种情况同时发生，此时一定发生过多次的 leader crash，结果上面两种情况发生在不同的 terms

logs 一致化的方法
- 同样是由 leader 来主导 logs 的一致化，它强制 followers 复制它自己的 log
- 基本的方法论是：
  + leader 首先找到它和某个 follower 之间完全一致的最后一个 log entry index
  + 删除 follower 在这个 index 之后的 log entries
  + 然后把 leader 在这个 index 之后的 log entries 追加到 follower 的 log 上
  
- 具体的实现是 follower 在处理 AppendEntries 请求时，进行 consistency check
  + leader 为每个 follower 维护 nextIndex 变量，指 leader 要发送给 follower new log entry 时的位置
  + 初始时指向 leader 自己 log 的最后一个位置之后的那个位置。显然如果有 new log entry，必然放到这个位置
  + AppendEntries 会携带这个变量，然后 follower 根据这个变量通过 log Matching 进行 consistency check
    * 如果发现一致，那么如果有 new log entry， follower 就进行 appending；如果是心跳，那么就返回正常
	* 如果不一致，返回拒绝，于是 leader 就对应的把 nextIndex 变量往回挪一位，继续下一次 AppendEntries
      最终，nextIndex 会达到 leader 和 follower 完全一致的 index 位置，就找到了一致性调整的初始位置
	  于是，就可以进行多余 log entries 的删除，以及缺失 log entries 的追加，最终返回成功，logs 达成一致
  + 我们看到，整个一致化过程中，leader 不会对自己的 log 进行删除或者覆盖，只有 append new entry 操作
    一致化的主导者是 leader，而操作者其实是 follower，其间 leader 并不需要做任何其他特别的操作

- Scenario 2. 的例子
  + 如果 S3 crash 之后，重新上线，然后又被选为 leader，进入 term 6 的情况
    * S3 发送 AppendEntries 给 S2，nextIndex[S2] = 15，同时携带前一个 entry 的状态 {index: 14, term: 5}
    * S2 回复 false，因为它 index 14 上的状态中 term = 4，不满足 Raft Log Matching Property
	* S3 设置 nextIndex[S2] = 14，并把自己 log 中的 slot.14 & 15 通过 AppendEntries 发给 S2
    * 同时 AppendEntries 请求中还携带前一个 entry 的状态 {index: 13, term: 3}
    * S2 检查发现自己的 index 13 也是 term 3，一致，根据 Log Matching Property，说明之前也一定都一致
	* 于是，S2 删除 index 13 之后的所有 log entries，然后添加 S3 发送过来的 slot.14 & 15 中的内容
	* 到此， S2 & S3 完全一致！
	* 于此同时，S3 也会和 S1 做完全类似的操作，S1 在 Slot.12 上和 S3 一致，并追加 Slot.13 & 14 & 15
	
  + 如果 S3 crash 之后，重新上线，发现此时 S1 被选为 leader，进入 term 6 的情况
    * 类似上面的情况，此时 S2 & S3 理应完全复制 S1 服务器的 log，删除自己 Slot.13 & 14 & 15 的 entry
    * 这里就会有一个问题，前面说过 Slot.13 	是有可能会被执行了的，如果删除，会违背一致性的原则
	* 前面场景分析以及原则部分都已经说过，有可能被执行的 log entry 必须保留，不能删除
	* 解决方案：S1 不能当选为 leader！下面会介绍 Raft 保证新当选的 leader 包含所有可能被执行的 entries


### Safety 和 leader election 中的限制

前面在实现 log 一致性的时候，方法是让 follower 无条件的复制 new leader 的 log entries

然而遇到一个问题：可能会丢掉已经 committed 的命令，这是不能容忍的。于是需要在 leader 选举时有一些限制

##### 先看一下，是不是选 log entries 最长的服务器就 ok 了？

错！参见 scenario 3 中的情况：
- 当所有服务器重启后，S1 有最长的 log，但不能选 S1 做 leader，否则会丢掉已经被 committed 的 term 8 命令
- S2 & S3 都可以做新的 leader，它们的 log entries 中都有 term 8 命令，而且在相同的 index 上

##### Leader Completeness Property 

在 leader election 时就保证：所选举的 leader 包含之前所有 terms 中所 commit 的所有 log entries
- 在 leader election 的 voting 阶段，一个 candidate 必须获得 majority 的选票才能当选
- majority 意味着，任何 committed log entries 都至少会在这些投赞同票的服务器中的某一个上
- 具体如何保证呢？首先定义两个服务器的 log entries 谁更 up-to-date 的算法：(leader election)
  + 比较两个服务器 logs 的各自最后一个 entry 的 index 和 term
  + 首先，如果 term 不同，那么 term 更大的更加 up-to-date
  + 其次，如果 term 相同，那么 index 更大的，也就是说 log 更长的，更加 up-to-date
- 当收到 candidate 的投票请求时，如果判断自己比 candidate 更 up-to-date，则拒绝投票；否则，投赞同票

由此，我们看一下 scenario 中的场景分析：
- scenario 2 中
  + S2 可能被选中，因为 S1 和 S2 会投票
  + S3 可能被选中，因为 S1 S2 和 S3 都会投票
  + S1 不会被选中，因为 S2 和 S3 都不会投票，其 term 过低
- scenario 3 中
  + S2 和 S3 都可能被选中，因为 S1 S2 和 S3 都会投票
  + S1 不会被选中，因为 S2 和 S3 都不会投票，其 term 过低


### Fast Backup（快速完成 logs 的一致性）

前面说过，在保证 log 一致性的时候，Raft 开始时的做法是每个 AppendEntries 判断一次，如果不一致往回挪一位

但是这个做法在 follower 的 log 和 leader 差距过大时，会非常没有效率

这里提供一个更有效率的算法：让 AppendEntries 的回复包含更多信息，使得 leader 能够更快的找到不一致
- XTerm：follower 和 leader 冲突的 term
- XIndex：follower 中 XTerm 第一个 entry 的 index
- XLen：follower log 的总长度

##### Case 1.

- Follower  4  5  5
- Leader    4  6  6  6

此时 XTerm 为 5，由于 Leader 都没有 5，于是直接 nextIndex = XIndex，来继续后续的比较

##### Case 2.

- Follower  4  4  4
- Leader    4  6  6  6

此时 XTerm 为 4，而 Leader 也有 4，那么直接 nextIndex = leader's Term 4 的最后一个 index，继续后续比较

##### Case 3.

- Follower  4
- Leader    4  6  6  6

当 Follower 的 log 特别短时，让 XTerm = -1，leader 就把 nextIndex 跳到 XLen 所在的位置，进行后续比较


### 提交以前 term 的 log entry

问题的场景见论文的 Figure 8.
1) S1 ~ S5 5 个服务器, S1 获选 term 2 的 leader
2) S1 在 log index 2 的位置上执行一个命令，并把该命令 replicate 到 S2 上，但是显然未达到 majority
3) S1 crash，S5 在 term 3 里通过 S3、S4 和自己的选票当选 Leader，然后收到新的命令放到 index 2 处
4) S5 在开始 replicate 之前就 crash 了，S1 当选 term 4 leader (只有 S5 有权拒绝 S1 而已 term 3 > term 2)
5) S1 把 term 2 index 2 的 entry replicate 到 majority 机器 S2 S3 上，但是还没有被提交就又 crash 了
6) S5 term 5 当选，并 replicate 它之前 term 3 index 2 的 entry，成功覆盖了所有服务器的 index 2 位置 

按截止到现在的逻辑来说，即使 step 5 中的命令被 commit 也同样无法避免在 step 6 中被覆盖的情况发生

为什么？因为其 term 为 2，即使 S5 只在 term 3 短暂当选，并且其 entry 并没有 replicate，仍然有权覆盖

如何避免恢复提交以前 term 的 log entry 由于 term 过低而被覆盖的情况呢？
- Raft 不会通过计算备份的数目，来提交之前 term 的 log entry
- 只有 leader 的当前 term 的 log entry 才会计算备份数并 commit

由上：
- 到 step 5 时，即使 term 2 index 2 的 entry 已经复制到 3 个服务器，但是仍然不能 commit
- 故此，到 step 6 时，即使 S5 覆盖了上面的 log entry，也不会有任何问题，因为该 entry 一定不会被 commit
- 如果在 step 6 之前，S1 并没有 crash ，而是收到新的命令，并成功复制 majority，那么可以 commit 该命令
  在执行该命令的时候，就可以提交 term 2 index 2 的命令了。就是说之前 term 的 pending 命令都可以提交
  提交之后，各个成功复制的 majority 服务器的最后 log entry 的 term 为 4，这样 S1 不会再被选举为 leader


### 为什么 Leader Completeness 属性成立？

Leader Completeness 属性保证所有 committed log entries 一定都在被选中的 leader 中，而不会被丢掉

我们采用反证法：
- 假设 Term T 时的 leader_t 提交的一个 log entry，不在后面 Term U leader_u 的 log 记录中
- 同时，假设 Term U 是第一个满足上面反证法假设的 Term

在反证法的假设下，我们可以推导出如下的推论
- 由于 leader 不会删除和覆盖 entries，故此 leader_u 当选时，该 entry 就一定不在 leader_u 的 log 记录中
- 由于被 leader_t 提交了，故此该 entry 一定已经被 replicate 到 majority 服务器上
- leader_u 既然当选，故此一定有 majority 服务器投票，那么其中必然有一个服务器 S 的 log 中有该 entry
- S 一定先 AppendEntries 这个 entry，之后才 vote leader_u，因为 Term T < Term U
- 由于假设了 Term U 是第一个满足反证法的 Term，那么 T & U 之间的 Terms 的 leaders 都含有该 entry
- 由上，故此当 S 投票给 leader_u 的时候，S 的 log 记录中仍然有该 entry，因为它不会和这些 leaders 冲突
- 由于 S 给 leader_u 投票，说明 leader_u 是等于或者 up-to-date 于 S 的
  + case 1：S 和 leader_u 的 logs 最后一个 entries 的 terms 相同，但是 leader_u 的 log 长度 >= S 的
  + case 2：S log 最后一个 entry 的 term 小于 leader_u log 最后一个 entry 的 term

由上面的推论，我们可以得到 contradictions：
- 先看 case 1：
  + 根据 Log Matching Property，同样 term 同样 index 的命令相同，且前面历史命令也都一定相同
  + 由于 leader_u 的 log 长度 >= S 的，故此 leader_u 的 log 必然包括或者等于 S 的 log
  + 这里就矛盾了，因为 entry 在 S 中，但是不在 leader_u 中 
- 再看 case 2:
  + 当选之前，leader_u log 的最后一个 entry 的 term (也即 U - 1) 一定大于 T
  + 那么，(U-1) 这个 term 的 leader 的 log 一定包含该 entry，因为假设了 Term U 是第一个满足反证法的
  + 由于 (U-1) term 的 leader 创建了 leader_u 当选之前 log 的最后一个 entry，
    那么根据 Log Matching 属性，此时 leader_u 和 (U-1) term 的 leader 应该有完全相同的命令
	于是，当选之前 leader_u 应该含有该 entry，这就矛盾了，因为当选之后，leader_u 不会删除其 log entry
  
故此，反证法不成立，其假设必然会导致矛盾的结果，进而 Leader Completeness 属性成立

由此，我们进而得到所谓的 State Machine Safety Property:
- 如果一个服务器执行了某个 index 上的 log entry，那么在这个 index 上就不会再有其他服务器执行其他命令
- 很容易由 Leader Completeness 来证明：每个 leader 当选时都会包含所有历史 commits，不会丢失和搞错位置


### 各个服务器需要持久化的变量

##### 服务器从 crash 恢复之后如何重新加入 Raft 集群工作？

- two strategies:
  + 恢复后进入初始化或者 snapshot 状态，leader 通过一致化 replay 所有或者 snapshot 之后的 log entries
    这个是必须支持的，否则无法处理永久损坏的服务器被替换，或者集群扩容的情况
  + 恢复后，从持久化的数据中更新状态，直接回到 Raft 集群工作
    这个也必须支持，否则集群全部 power off 时，系统全部清空或者进入 snapshot，丢掉后面 committed logs

- 于是，我们需要服务器持久化一些必要的状态
  + log[]：服务器 reboot 之后，将来的 leader 能够保证看到所有 committed log entries
  + votedFor：防止重启之后忘记了自己曾经 vote A，收到 B 的请求后又投给 B，导致多个 leader 出现
  + currentTerm：crash 之前的 Term 号必须持久化，否则很多事情都会错，因为用到 Term 的地方太多了

- 有一些状态并不需要持久化，称为 volatile 状态，它们可以在 reboot 后通过持久化的数据和 Raft 来重建
  + commitIndex：目前已经 committed 的最高 index 的 log entry，可以由 leader AppendEntries 请求恢复
  + lastApplied：目前已经执行完命令的最高 index 的 log entry，可以由 leader AppendEntries 请求恢复
  + nextIndex[] (only on leader)：leader crash 后会重新选举 leader，然后重新初始化这个变量
  + matchIndex[] (only on leader)：保存每个服务器已经 replicated 的最高位置的 log entry，AE 请求重建

##### 持久化的一些讨论

持久化的方式通常是每次改变变量，那么就持久化，只有持久化成功后，才能算完成本次操作

持久化通常是写入 disk，而这就会成为系统性能的瓶颈
- 机械硬盘写入 10 ms, 这样的话，这个服务每秒最多能服务 100 个请求，性能很差
- SSD 写入 0.1 ms，这样每秒最多能服务 10000 个请求 (如果其他方面不会成为瓶颈的话)
  (the other potential bottleneck is RPC, which takes << 1 ms on a LAN)
- 很多小 tricks 可以克服持久化的性能问题
  + batch many new log entries per disk write
  + persist to battery-backed RAM, not disk


### log compaction & Snapshots

##### 背景

- 如果只持久化 logs，这样服务器替换或者扩容时，新服务器重建会非常的慢，要 replay 所有 logs
- logs 可能会非常的巨大，尤其是相对于后台服务的状态来说。那么与其持久化操作(logs)，不如持久化状态
- 这样，恢复服务时直接加载最近的 snapshot 会快很多

##### snapshot 解决方案

- 各个服务器周期性创建持久化的 snapshot，保存后台服务的状态到 disk 上
- 同时记住 snapshot 所包含的最后一个 index (lastIncludedIndex)以及该位置对应的 term (lastIncludedTerm)
- 上面这两个变量会被用于 AppendEntries 请求，因为 AE 请求要前一个 entry 的 index/term 进行一致性检查
- snapshot 中还会记住 lastIncludedIndex 位置所对应的 config，以支持后续集群的 membership change (后面讲)
- 然后删除 snapshot 所覆盖的那些 log entries，也即 last index 及其之前的 entries，只保留后面的 entries

##### snapshot 的一些相关事项

在后台服务的某个状态上完成 snapshot 之后，应该要删除 snapshot 点之前的 log，但是
- un-executed entries 不能丢：log entry 没有执行的话，其结果就没有反应到后台状态，也就不在 snapshot 中
- un-committed entries 不能丢：因为这个 log entry 可能将会是 majority 的一部分，且一定未被执行

crash+restart 的流程
- 服务器从 disk 上读取最近的 snapshot
— 服务器从 disk 上读取持久化的 Logs，也就是 snapshot 之后的那些 Log entries
- 服务器设置 lastApplied 为 last included index 以避免重新执行已经被执行了的 entries

##### InstallSnapshot 请求

问题来了：
- 假设某个 follower crash 很久，reboot 之后 log entries 的最后一个 index 为 2
- 而目前 leader 以及其它服务器都已经 snapshot 完 index 3 之前的状态了
- 请问：此时该 follower 如何重新回到集群恢复工作状态？

为什么这是个问题？
- 由于 leader 等服务器都已经完成 snapshot，就是说都已删除了 index 3 之前的 log entries
- 故此，leader 已经无法 AppendEntries 请求来恢复这个 follower 的 log entries 了，因为 index 3 无法找回

如何解决？
- 一个方法是 leader 不删除 followers 可能没有 catch up 的那些 log entries，比如上面例子中的 index 3
- 这个方法有个问题：如果 follower 长期离线，导致 leader 必须长期保留 logs，会引发内存负担过大的问题
- Raft 采取的方法是：当 AppendEntries 请求失败，无法恢复 log entries 时，会发送 InstallSnapshot 请求
- 这个请求中， leader 会把自己的 snapshot 发送给 follower，使得 stale follower 恢复到 snapshot 状态
- 然后，再发送 AppendEntries 请求，就可以通过 logs 一致化的流程，逐步恢复 follower 到最新进度

如果是正常的 follower 而不是 staled follower，收到 InstallSnapshot 请求的话：
- 可以保存 snapshot，然后在 logs 中删除那些被 snapshot 所 cover 的 entries，保留其他后续 entries

##### snapshot 与 Raft 的模块化

- snapshot 工作密切和 Raft 所支持的后台服务相关，不同服务 snapshot 的方法可能会非常不同
- 故此，snapshot 使得 Raft 无法完全独立地实现模块化，必须和后台服务进行交互
- 最后，有一些后台服务的状态也非常大，比如数据库，对其进行 snapshot 也是非常耗时耗空间的操作
  + 有一些 incremental approaches，比如 Log Cleaning 或者 Log-Structured Merge Tree (LSM Tree)
  + 或者一些基于 B-Tree 的数据库其本身就已经在 disk 上了，不再需要另外 snapshot

##### Alternative 解决方案？？

- 一个其他方法是只让 leader 进行 snapshot，然后通过 InstallSnapshot 命令把 snapshot 发送给 followers
- 显然问题一：通过网络发送 snapshot 会极大占用网络带宽，而且减慢 snapshot 的运行时间
- 显然问题二：leader 的实现会变得更加复杂，snapshot 时间过长，会和其他 RPC 同步运行，如果不中断服务的话
  
  
### linearizability 的保证

前面我们介绍了 Raft 如何保证 leader 的唯一性、committed logs 不会丢失性和 logs 一致性

现在要讨论的是，对于客户端来讲，调用 Raft 后台服务的正确性。何为 correctness？需要一个定义

"linearizability"：对 Raft 集群的调用行为就和调用一个 single server 一样，更具体的定义如下：
- 多个客户端对 Raft 集群并发调用，最终得到若干个请求和回复
- 能够找到一个全部 operations 的整体执行序列，使所有读请求的结果都是序列中该读请求之前的写请求的写入值

下面我们会讨论一些例子，看它们是否能保证 linearizability 的要求。之前我们统一一下写作规范
- |-Wx1-|：第一个竖线表示发送请求的时间点，第二个竖线表示收到请求的时间点，Wx1 表示写请求，写入 x=1
- |-Rx1-|：竖线含义同上，Rx1 表示读请求，读取 x 值，结果为 1
- 注意，两个竖线只是发送请求和收到结果的点，而命令真正执行的时间点可能在两个竖线之间的任意位置上
- |-×-Wx1-|：表示在 × 点上命令真正的执行，比如说 1 在此时真正写入了 x 变量中

##### example history 1 （可行）

  |-Wx1-| |-Wx2-|
    |---Rx2---|
      |-Rx1-|
	  
由上我们能看到：
- 1 一定在 2 之前被写入
- Rx1 和 Rx2 请求发生在 Wx1 请求的执行时间内，Rx1 和 Rx2 请求的回复发生在 Wx2 请求的执行时间内
- Rx1 请求的发生和回复都发生在 Rx2 请求的执行时间内

我们可以实现一个执行序列： Wx1 Rx1 Wx2 Rx2，就能够满足上图 4 个请求的执行情况

  |-Wx1-×--| |-×-Wx2--|
     |----Rx2----×-|
       |--×-Rx1-|

##### example history 2 （不可行）

  |-Wx1-| |-Wx2-|
    |--Rx2--|
              |-Rx1-|
			  
由上图，我们可以看到：
- 从时间轴上看，Wx1 先于 Wx2 执行完成
- 从时间轴上看，Rx2 先于 Rx1 执行完成
这显然发生了冲突，因为 Rx2 必然在 Wx2 之后才能完成，此时 x=1 的值已经被 x=2 覆盖，不可能之后读到 Rx1

##### example history 3 （可行）

|--Wx0--|  |--Wx1--|
            |--Wx2--|
        |-Rx2-| |-Rx1-|
		
由上图，我们可以看到
- Rx2 一定比 Rx1 执行完毕，那么意味着如果可行，那么必然 Wx2 应该比 Wx1 更早执行完毕
- 有图上可以看到，Wx2 请求的回复时间更晚，但这和前面所说的更早执行完毕并不冲突，只是其他方面拖慢了请求

我们能找到一个可行的执行顺序: Wx0 Wx2 Rx2 Wx1 Rx1

|--Wx0--|  |---Wx1--×--|
            |-×--Wx2-----|
        |-Rx2---×-| |-×-Rx1-|

##### example history 4 （不可行）

|--Wx0--|  |--Wx1--|
            |--Wx2--|
C1:     |-Rx2-| |-Rx1-|
C2:     |-Rx1-| |-Rx2-|

由上图，我们可以看到两个客户端 C1 & C2 各发起了两次读请求，并得到完全相反的结果
- 由 C1 的请求，Wx2 必然要比 Wx1 先执行完毕，才能得到 Rx2 比 Rx1 更早执行完毕的结果
- 由 C2 的请求，Wx1 必然要比 Wx2 先执行完毕，才能得到 Rx1 比 Rx2 更早执行完毕的结果

矛盾，所以不可行
- 单看 C1 或者 C2，都能得到可行的执行序列，但是 linearizability 不允许不同的客户端看到不同的顺序结果
- 有 replicas 服务器的系统，出现不同客户端不同结果的问题是可能的，但是 linearizability 要求次序必须一致

##### example history 5 （不可行）

|-Wx1-|
        |-Wx2-|
                |-Rx1-|

这个显然也是不可行的，但是之所以提出来，是要告诉我们，linearizability 不允许：
- 读请求返回 staled 结果，比如 from lagging replicas
- split brain (two active leaders，或者从进度不同的 replicated 服务器读取数据)
- forgetting committed writes after a reboot

##### example history 6 （允许 re-send 的情况下读取 staled 数据）

- 假设 C2 在 Wx3 完成后，Wx4 请求之前发送了 Rx 读请求，但是没有收到回复，于是又发送了一次
- 结果，在 C1 的 Wx4 执行完成之后才返回，且返回值为 3，而不是 4

C1: |-Wx3-|          |-Wx4-|
C2:          |-Rx3-------------|

这个是 linearizability 所允许的
- 很可能服务器已经对第一个 Rx 请求返回了结果 3，并进行了缓存
- 当 C2 re-send Rx 请求时，服务器中 x 已经被覆盖为 4 了，但服务器看到这是个 re-send，故此返回缓存值 3

linearizability 认为执行序列其实是：Wx3 Rx3 Wx4，仍然认为 Rx3 发生在 Wx4 之前，不违反次序

更多 linearizability 的话题参见：
- https://www.anishathalye.com/2017/06/04/testing-distributed-systems-for-linearizability/


### 其他的一些相关问题

##### duplicate RPC detection 重发请求的检测

由上面的 example history 6，我们知道 Raft 支持对重发的请求通过缓存直接返回结果

为什么要支持？
- 客户端可能未收到服务器返回的结果，可能是服务器挂掉了，可能是服务器返回了，但是网络出了问题
- 两种情况对于客户端来看，结果都是一样的，无法分辨是哪种情况，故此它只能 re-send 重发请求
- 但是，对于服务器来说，如果不分辨是否是重发的请求，就必须支持相同的命令重复执行的幂等性，否则会出问题
- 然而，上面的要求是个非常强的要求，很难满足，还不如支持对于重发请求的检测来得更容易一些

实现方法 duplicate RPC detection
- 客户端要对每个请求分配一个唯一 ID，重发的请求共享之前的 ID
- 服务器端对每个客户端都维护一个 table，保存对每个请求的返回结果
- 如果某个客户端发送的请求 ID 已经在其 table 中，直接返回 table 中的结果
- 一旦客户端发送了一个请求 ID，那么该 ID 之前的那些请求在 table 中的记录都可以删除了，说明客户端已收到
  （其实这里暗含一些要求：每个客户端在同一时间里只有一个 outstanding RPC 请求，且客户端请求 ID 递增）
- 如果重发的请求过来时，第一个请求还未结束怎么办？就不取缓存直接运行呗，第二次的结果会覆盖第一次结果
- new leader 如何获取缓存 table 呢？我们让每个 replica 服务器都更新自己的 tables，new leader 自然就有了
- 服务器 crash 了，如何重建 table？
  + 如果没有 snapshot，则 replay log 并使用 replay 的结果来重新生成这个 table
  + 如果有 snapshot，让 snapshot 中也包含一份 table 的 copy 即可 

##### More about 只读操作

这里的问题是: 对于只读的操作来说，先要经过 majority commit 才能执行的要求是有必要么？

是有必要的！
- 有时 old leader 在 minority 分区中或者联系不上其他服务器，有一个时间窗口内，它仍不知道
- 如果不需要 commit，那么这个 old leader 就可能会响应客户端的请求，并返回结果，而这个结果可能是 staled
- 同时，new leader 可能更新了数据，同样也会响应其他客户端的请求，于是形成了 split brain
   
只有先经过 majority commit，old leader 才能发现它无法对请求获得多数票，也就无法响应客户端的请求

对于一些应用，可能会有非常 heavy 的 read 只读请求，而如果获取 majority commit 的话，会比较花费时间
- 方法一：一些实践中的应用，其调用者并不是很 care 是否得到相对 staled 的数据，而是更追求性能
- 方法二：lease
  + 需要修改 Raft 协议，增加 lease 的概念，比如定义每个 lease 为 5 秒的时间
  + 每次 leader 的只读操作在 AppendEntries RPC 中获得了 majority，进入了 commit 状态，就发一个 lease
  + 在 Lease 的时间段里，后续的只读操作不再需要 majority commit 的要求
  + 为了避免 staled 数据，在 lease 时间段内，同样不允许写入操作，保证只读操作的结果都正确

##### Follower 或者 Candidates crash 会如何？

前面介绍了 leader crash 后要进行的 leader election 等问题，那么非 leader crash 会有什么影响呢？

此时，发送给他的 AppendEntries 和 RequestVote 请求会失败，那么也没什么，再重发给他就是了，一直重发！

那么，如果他是在完成了请求但是发送回复消息之前 crash 的怎么办？reboot 后不就会再次执行一遍请求了么？
- 是的，但是没关系，我们要求 AppendEntries 和 RequestVote 请求的执行是幂等性的，因为并不会影响服务状态
  + 对于 AppendEntries 请求，它要判断是不是对应的命令已经加入到自己的 logs 中了，如果是的话直接返回即可
  + 对于 RequestVote 请求，类似判断是否已经在对应 term 中选举了相同的调用服务器
    * 如果选举了，说明 crash 之前就已经执行选举了，但是调用服务器未收到回复，那么直接回复 yes 即可
	* 如果选举了其他服务器，那么一个 term 只能选举一次，那么返回 no 即可
  + 非 leader 不会处理客户端的命令，即使接收到客户端的请求，也只会转发给 leader，故此不会有什么影响

##### RPC 执行时间和 leader election timout 的选择

RPC 的执行时间称为 broadcast time，指一个服务器并发给其他每个服务器发送 RPC 请求并最终得到所有回复的时间

leader election timeout 前面说过，是指一个服务器等待多久没有收到 leader 的消息就会重新发起选举的时间

我们希望 broadcast time << leader election timeout，通常要少一个数量级，比如小 20 倍为佳
- broadcast time 是服务器之间的网络以及 Raft 请求实现算法决定的，通常为 0.5 ~ 20 ms
- leader election timeout 是完全由实现者所选择的，按上面的要求，通常为 10 ~ 500 ms


### Raft 集群成员变化 (服务器替换、扩容、减容等)

当集群服务器成员发生变化时，最保险的方法是停止集群，然后调整 configuration，最后重启集群

问题在于，上面的方法会短暂的停止服务，影响集群的可用性。有办法不停止服务就能自动调整服务器成员么？
- 难点在于，任何方法也不可能在一瞬间更新所有服务器上的 configuration
- 于是很可能会有一个时间窗口上，有的服务器还是使用 old configuration，有的已经用了新的
- 这样，如果这个窗口发生了 leader election，可能会在两个 configurations 上各自产生一个 leader
- 比如 3 个增加到 5 个，原来两个还是老的配置，会产生一个 leader，剩下三个又产生一个新配置下的 leader

Raft 采用 two-phase 方法，在新旧配置交接的时间段内引入 transitional configuration (称 joint consensus)
- Raft 会把 configuration 作为一种 log entry
  + 类似普通 log entry，当 majority 收到 AppendEntries 请求中的配置，该配置就达到 commit 状态
  + 不同的是，只要 follower 收到配置数据，就会立即使用新配置来做后续决策，不管是否达到 commit 状态
- joint consensus 会合并 old and new configuraitons
  + Log entries 会被 replicated 到新配置和旧配置中的所有的服务器上去
  + 新、旧配置中的任何一个服务器都可能被选为 leader
  + 期间 RequestVote 和 AppendEntries 请求的 commit 需要新、旧配置中的服务器都各自达到 majority 才行
- 好处就是通过这个方法，在配置调整的过程中都不会中断客户端的请求，而且不会影响之前说过的 Safety

Raft 的论文并没有对集群成员变化的流程和细节做很详细的介绍，有很多语焉不详的地方，需要仔细看论文和代码

https://github.com/logcabin/logcabin


### Raft 的一些小结

到此，我们已经学习完整个 Raft 的设计，包括：
- 客户端发送请求后，Raft 集群处理请求的整个流程
- Leader election 的算法，保证每个 term 最多只有一个 leader 产生
- Raft 如何保证 logs 即使经历 failures 仍能保持一致化
- Log Matching Property
- Leader election 时的限制，使得 leader 一定会包含所有的 commits （Leader Completeness)
- 各个服务器要持久化的数据，以免 reboot 后丢失
- Log compaction 和 Snapshot
- Raft 如何保证 Linearizability 属性
- 一些其他方面的讨论
  + Fast Backup 的实现 
  + 如何提交之前 term 的 log entries
  + duplicate RPC detection 重发请求的检测
  + More about 只读操作
  + Follower 或者 Candidates crash 会如何？
  + RPC 执行时间和 leader election timout 的选择
  + Raft 集群成员变化

##### committed v.s. applied

- committed 表示 leader 的 entry 已经被 majority 服务器 replicated 的状态，不会丢失了
- applied 表示 entry (上的命令) 被执行 executed，致使后台服务比如 K/V 的 state 已经发生了变化

##### 重要属性

- Election Safety：每个 term 最多只有一个服务器会被选为 leader
- Leader Append-Only：leader 不会覆盖或者删除其 log 中的 entries，只会 append 新的 entries
- Log Matching：两个 logs 包含 index & term 都相同的 entry，那么该 entry 之前的所有 entries 命令都一致
- Leader Completeness：一旦某 log entry 被 commit，那么它必然会出现在后面所有 terms 的 leader logs 中
- State Machine Safety：在某 index 上的 entry 一旦被应用，则该 index 不会再有别的服务器应用其他 entry



