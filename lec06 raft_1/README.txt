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

对于 Scenario 2，我们看到了 3 个服务器的 log entries 都完全不同，此时应该如何处理？
- Slot.12 很简单，大家都一致，可以保留
- Slot.13 上，S2 & S3 是一致的，我们可以保留，只要 leader 主动让大家保持同步，也就是让 S1 追加 log 即可
- 注意，此时 Slot.13 上的 log 所对应的 command 可能是 commit 的，也可能没有 commit，这是无法推测的
- 甚至 Slot.12 上的 log 对应的 command 都有可能没有 commit，S3 还未接收到 S1 & S2 的回复就先 crash 了
- 关键点是：只要某个 slot 中的命令是有可能已经 commit 执行了，就必须保留，否则一定和系统状态不一致
  在这条原则之下，我们认为 Slot.12 & 13 必须保留下来
- Slot.14 上的 S2 & S3 的 log 可以删除，这两条必然没有被 commit 执行，完全可以删除掉，让客户端重新请求

这样，我们可以实现 S1 S2 & S3 三个服务器之间的 log entries 一致性

但是问题是，我们无法确认 log entries 中的动作是否已经 commit 还是没有 commit，这对整个程序来讲还有隐患

##### Raft 保持 log entries 一致的原则和方法

原则
- Raft 保证已经 committed 的 log entries 是持久化的，而且最终将会被所有可用 available 的服务器执行

- leader 会记录目前 commit 的最大的 log entry index，且会把该 index 带到后续 AppendEntries 请求中
- follower 从 AppendEntries 请求中读取该 index，然后执行该 index 对应的 log entry 并执行其中的命令

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

- 由上面的属性，那么如果 leader 从未 crash 的话，各个服务器的 logs 一定最终一致
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

























