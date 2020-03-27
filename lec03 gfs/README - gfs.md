# 课程 03

## 课前思考

阅读论文 [GFS(2003)](gfs.pdf)，可以思考以下问题:

1. Why is atomic record append at-least-once, rather than exactly once?
1. How does an application know what sections of a chunk consist of padding and duplicate records?
1. How can clients find their data given that atomic record append writes it at an unpredictable offset in the file?
1. The paper mentions reference counts -- what are they?
1. If an application uses the standard POSIX file APIs, would it need to be modified in order to use GFS?
1. How does GFS determine the location of the nearest replica?
1. Does Google still use GFS?
1. Won't the master be a performance bottleneck?
1. How acceptable is it that GFS trades correctness for performance and simplicity?
1. What if the master fails?

问题的答案在[这里](gfs-faq.txt)

在阅读[讲义](l-gfs.txt)请先思考以下问题：

- Why are we reading this paper?
- What is consistency?
- "Ideal" consistency model
- Challenges to achieving ideal consistency
- GFS goals:
- High-level design / Reads / Writes
- Record append
- Housekeeping
- Failures
- Does GFS achieve "ideal" consistency?
- Authors claims weak consistency is not a big problems for apps
- Performance (Figure 3)



## gfs 论文研究

### 分布式系统的难题

- 希望 high performance => 分片 shard data over many servers => 服务器多了就 constant faults
  => 需要容错 fault tolerance => 具体方法就是 replication => 多副本的一致性 potential inconsistencies
  => 希望有更好的一致性 better consistency => 付出成本，而成本就是会降低性能 low performance

  
### 设计概要

##### Assumptions 设计时的一些假设

- 机器出错很常见，必须保持监控，并有容错、自动恢复功能
- gfs 系统主要针对大文件，不需要为小文件做优化
- 主要支持大量的流式文件读取，而不是小量的随机读取
- 主要支持大量的顺序追加写入，通常文件写入后就不再修改，而小量的任意位置写入不必有效率
- 系统必须支持多个客户端对同一个文件的并发写入或者多路合并，最小化并发成本下的原子性是核心要素
- 高带宽、高吞吐量比低延迟、响应时间更重要

##### 支持的接口

- 支持部分的 POSIX 标准接口，如 create, delete, open, close, read, and write files
- snapshot: 支持低成本的文件或者目录树的快照 copy
- record append: 支持多个客户端并发追加相同的文件，同时保证每个客户端追加写入时的原子性
- 后两个接口对于大规模分布式系统非常重要

##### 架构和设计

- 单个 DataCenter，系统不会横跨多个 DataCenter
- 1 个 master + 多个 chunkservers
- 文件会被切分为多个固定大小的 chunks，通常 64 MB，保存在 chunkservers 上
- 在 chunk 被创建的时候，会由 master 分配一个不可变的、全局唯一的、64 bit 的 chunk handle
- chunkservers 会把 chunks 保存为本地硬盘文件，标识为 chunk handle
- 为了可靠性，每个 chunk 会被复制到多个 chunkservers 上保存，通常是 3 个

- 单个 master 极大简化了设计，同时能够实现 sophisticated 的 chunk placement & replication 决策
- master 维护所有的 metadata，包括 namespace, access control, 文件和 chunk 间的映射、chunk 的位置等
- master 还控制系统行为，包括 chunk lease 管理、孤儿 chunk GC 回收、chunkservers 之间的 chunk 迁移
- master 和 chunkservers 之间保持周期性的 Heartbeat 消息，以便收集状态和下达指令
- 为减少 master 的负担，client 操作文件时，master 只回复文件的 metadata 以及负责的 chunkservers
  后续文件的实际操作过程，由 client 和返回的 chunkservers 直接联系，而不再通过 master

- clients 和 chunkservers 都不会缓存 cache 数据
   + clients 通常处理流式数据（无法缓存）或者大数据文件（太大了），故此不使用 cache
   + chunkservers 本身把 chunk 保存为 linux 本地文件，linux 自身的 buffer cache 已经自带缓存
   
- 64 MB 大小的 chunk size，远大于文件系统本身的 block size，这对于大数据来说，带来了一些好处
   + 一次请求可以处理更多的文件大小，或者说减少了 client 和 master 之间的交互频率
   + 减少了 metadata 的 size，以便 master 可以存放更多的 metadata，以及更快的访问速度
  当然，也带来了一些问题，最主要的是对于小文件来说，很可能就只有几个 chunks，会形成 hot-spot
   + 当很多 clients 访问同一个文件时，chunks 越多就越容易被并发处理，多个 clients 负责不同 chunks
   + 但是，如果文件很小，chunks 数量很少，这个文件就会变成 hot-spot 热点，访问速度变慢
   
##### Metadata (由 master 维护)

- Metadata 包括 1) 文件和 chunks namespace  2) 文件和 chunks 的映射  3) chunk replicas 的位置
  所有的 Metadata 都会放到 master 的内存中，前两种数据还会被以 operation log 的形式落地在文件系统中
  master 不会把第三种数据落地到文件，初始化以及新 chunkserver 加入时，master 会从 chunkservers 同步
- metadata 常驻内存，使得 master 访问速度很快，维护系统状态效率也高
  维护工作包括：孤儿 chunk 回收、chunkserver 挂掉后的 re-replica、chunk 迁移以负载均衡等
- chunks 位置数据不会落地文件，这简化了设计，因为分布式系统节点失败过多，同步和维护这类数据成本太高
- 落地的 operation log 可用于 master 出错后重新恢复系统，它持久化了系统操作的 time line
  为了系统的可用性，master 还会把 operation log 同步到其他多个远程服务器，以免丢失
  而且，只有成功的把 operation log flush 到 master 和远程节点后，才会给 clients 返回请求结果信息
  为了增加系统的整体吞吐量，master 其实会把几个 operation logs batch 到一起 flush
- 系统恢复的方法是 replay operation logs，为了减少启动时间，我们必须减少 logs 的数量
  具体方法是，每当 log 增加到一定程度后，就把当前所有状态生成一个 checkpoint
  这样系统恢复时，其实是先恢复最新的 checkpoint，然后再 replay checkpoint 之后的 operation logs
  checkpoint 类似一个压缩的 b-tree，能够直接映射到内存中，这加速了恢复过程，增加了可用性
- checkpoint 通常会持续一段时间，故此 master 内部状态的实现非常有讲究：
  确保新 checkpoint 被生成的时候，并不影响和耽误系统响应客户端新的请求和文件处理
  master 其实会切换到一个新的 log file，新的 checkpoint 会被放到一个单独的线程中生成
  新的 checkpoint 会记录发生切换之前的所有系统状态和 operations，完成后，也会落地到本地和远程文件
  
小结： Master Metadata
- file name --> array of chunk handle (memory & disk)
- chunk handle --> list of chunkservers (memory only)
                   version (memory & disk，以便重启后和 chunkservers 发送来的 chunks 版本进行校验)
				   primay & lease expiration  (memory only，后面会介绍)
- operation log & checkpoint (disk only，用于恢复系统状态)

##### 一致性模型（一个放松了 relexed 的一致性模型）

- 操作包括对 namespace 的（如文件创建）和对 region 的（如随机 write 和 record append，统称 mutation）
- file namespace 的操作是原子性的，由 Master 所完全控制，Operation log 定义了这些操作的全局顺序
- file region 操作后的状态依赖于操作的类型、操作是否成功、是串行还是并行三个方面所决定
- 所谓 file region 的状态，包括以下几种：
   * consistent 一致的：当所有 clients，无论从哪个 replica 读取，读到的数据都一样
   * defined 已定义的：region 是 consistent，而且 clients 能够看到操作写入的所有内容
- 那么，操作后的状态包括以下情况：
   * 未受并发 writer 干扰的 mutation，操作成功后，能够达到 defined 状态（自然也是 consistent 的）
     所有 clients 都能从任何的 replica 读取相同的数据，而且 clients 清楚写入的内容 (毕竟串行)
   * 多个并发 writers 成功写入后的状态是 consistent 的，但在并发写入下，clients 无法预知写入顺序
     具体，为什么并发下能达到 consistent 的状态？这是因为：
       + GFS 保证在所有的 replica chunkservers 都应用完全相同顺序的 mutations
	   + 用 chunk 的版本号检测 replica 是否因其所在 chunkserver 宕机而导致数据修改滞后或者遗漏
	   + staled 失效的 replicas 不会再执行任何 mutations，也不会被返回给客户端，还会被 GC 尽快回收
   * 失败的 mutation 连 consistent 的状态也无法达到，可能某个 chunkserver 成功了，但其他的写入失败
     那么，不同的 clients 连接其接近的 chunkservers，就会看到不一致的结果
- record append 这个操作和常规的文件 append 并不完全一致
   * GFS 保证 at least once 原子性追加，位置由 master 决定，而不是常规上所认为的文件末尾
   * GFS 返回给客户端一个位置偏移量，标记了一个包含 record 的 defined region 的起点，用于追加
   * GFS 还可能会在文件中间加入一些 padding 或者 duplicated 数据记录，它们会导致微小的不一致
- clients 可能会缓存 chunks 的位置信息 (某 replica 服务器)，那么有可能会在失效期前读取失效数据
  当 clients 想再次打开文件以及联系 master 时，会立即清理缓存，并更新 chunk 的最新位置
- GFS 和 chunkservers 之间的常规握手时，会通过 checksumming 机制来检测出数据损坏
  一旦检测到数据损坏，会通过正确的 replicas 来恢复数据。只有所有 replicas 都损坏才表示数据丢失
  此时，客户端是明确的受到数据损坏的响应，而不是不知情的情况下接收到损坏的数据
- GFS 通过 record append (instead of overwriting)、checkpoints、自验证自标识的写入机制来实现一致性
   * append 能很容易的实现周期性的保存 checkpoint，并在 checkpoint 的基础上继续新的增量 mutations 
   * checkpoint 配合 application-level checksums 保证了其中 file regions 的状态一定是 defined
   * append 满足了我们一致性和并发处理的要求，比随机位置的 write 更有效率，对失败的处理更具有弹性
   * checkpoint 允许 Writer 增量式重新开始，并防止 Reader 读取已成功写入但以应用程序看还未完成的数据
   * writers 可以并发写入同一个文件，写入数据中带有 checksum，也由 GFS 填入一些 padding 或者重复数据
   * reader 则可以根据 checksum 的值来检验读取的数据，进而去掉额外的 padding 和重复数据

   
### 系统交互

GFS 的系统设计要点就是最小化单点 master 的交互压力，下面会介绍 master/chunkserver/client 间的交互

##### client 读被执行的流程

- client 向 master 询问目标文件以及 offset 位置的信息，如果它之前没有缓存过的话
- master 找到内存中文件以及 offset 对应的 chunk 信息，进而找到 chunkservers
- master 把 chunkservers 回复给 client，当然只返回最新版本 chunk 对应的 chunkserver
- client 缓存 chunks 和 chunkserver 信息
- client 把 chunk、offsert 发送给最近的 chunkserver
- chunkserver 从硬盘读取 chunks 数据，返回给 client

##### Lease（租约）和 Mutation Order

- 已知 mutations 需要在所有 replica 上应用，GFS 使用 lease 来保证 replicas 间一致的操作顺序
- lease 是对应 chunk 的，master 会选择 chunk 的一个 replica 来颁发 lease，这个 replica 称为 primary
- primary replica 会决定该 chunk 上一系列 mutations 的顺序，其他 replicas 会遵守这个顺序来执行
- 这样就保证了全局的 mutations 都是按顺序执行的，操作完成后达到 defined 的状态
- lease 的主要作用就是减轻 master 的负担，把执行顺序的决定权下放到 primary 上
- lease 初始超时为 60 秒，不过，只要 chunk 被修改了，primary 就可以申请延期，通常会得到 Master 确认
- lease extension 请求和批准通常都是附加(piggyback)在 master 和 chunkservers 之间的心跳消中传递的
- master 可能提前取消 revoke lease（如 master 想取消在一个已经被改名的文件上的修改操作）
  即使 master 和 primary 失去联系，它仍然可以安全地在旧 lease 到期后和其他 chunkserver 签 lease

client mutation 被执行的流程 (即写数据的流程)
1. client 向 master 咨询哪个 chunkserver 是对应 chunk 的 primary，以及所有 replica chunkservers 位置
   如果此时还不存在 lease 和 primary，master 会立即授权一个 replica
   然后 master 给 chunk 版本号 +1，并告知 primary & 其他 chunkservers
   primary 和其他 chunkservers 也把对应 chunk 的版本号 +1，并把版本号写到 disk 上，并回复 master
   master 此时也把版本号写到 disk 以避免丢失 (这也是为什么如果之前 master 掉线，版本号可能会落后)
2. master 回复 client 以上的信息，client 缓存这些信息，只有 primary 失去联系时，才会重新向 master 获取
3. client 推送操作所需的数据到所有的 replica，每个 chunkserver 保存数据到其内部 LRU buffer cache
   看到数据流 dataflow 是推送到所有 replica 的，和 control flow 解耦以提升效率，后续会有详细介绍
4. 3 完成后，client 发送操作请求给 primary，后者安排当前所有要执行的操作的顺序和位置，然后实施操作
5. primary 把这些操作和位置、次序信息 forward 给所有 replicas，由后者按同样的次序实施这些操作，达成一致
6. 所有 replica 回复 primary 已经实施完毕或者发生失败
7. primary 回复 client 成功和失败的 replicas，其实到这里时 primary 一定实施成功，否则 step 4 就结束了
   如果有失败的节点，client 认为请求失败，对应的 file region 处于不一致状态，会 retry 这些 mutations
   
- 如果写入的数据过大，或追加点位置已经接近 chunk 的边界，GFS client 会把 mutation 切分为多个 mutations
- 所有 mutations 都遵守上述的控制流，连同其他 clients 所请求的 mutations 一起，按 primary 排序执行

- 前面说过 lease 和 primary 都保存为 memory only，这样 master 重启后就不再知道之前的 primary
  处理不好的话，可能会有 split brain 的问题，比如 master 挂掉前指定了 primary，重启后又指定一个新的
  那么，由于 client 会缓存 primary，之前就访问过老 primary 的 client 还是会访问老的 primary
  而没有之前缓存的 client 又会访问新的 primary，造成 split brain 的问题，这是分布式常见的问题
  那么，GFS 的解决方法很简单，由于 primary lease 只有 60 秒，故此只要等到其过期，再重新分配即可

##### Data Flow

- 控制流是从 client 到 primary 到 replicas，而数据流则是 client 直接到 replicas
- 数据是以 pipeline 的方式，顺序的沿着精心选择的 chunkserver 链路推送，目标是充分发挥每个机器的带宽
- 每台机器所有的出口带宽都用于以最快的速度传输数据，而不是在多个接受者之间分配带宽，方法是：
   * 每台机器都在网络拓扑中选择一台还没有接收到数据的、离自己最近的机器作为目标推送数据
   * 通过 IP 地址就可以计算出节点的“距离”
   * 采用全双工的交换网络。接收到数据后立刻向前推送，而不会降低同时接收的速度

##### Atomic Record Appends

- GFS 的应用场景，通常是多个生产者+单消费者的队列系统，或者是来自多个 clients 数据的结果合并
- 前面说过，append 的位置是由 master 决定的，并且 GFS 保证 at least once 的原子写入
- 类似 Unix 系统编程中，对以 O_APPEND 模式打开的文件，多个并发写操作在没有 race condition 下的行为
- 如果使用传统的文件写入方式，clients 需要需要额外的复杂、昂贵的同步机制，如分布式的锁管理器等
- Atomic record append 是 mutation 的一种，完全遵守上面 lease 和 mutation 执行的流程
- 如上，在 apend 操作失败的情况下，client 会进行 retry，而则可能会产生 padding 和重复数据
   GFS 不保证所有 replicas 都是 bytewise identical 的，只保证 at least once 的原子性写入

##### Snapshot 快照

- snapshot 能够几乎瞬间完成指定文件或者目录的快照，且同时最小化对其他正在进行的操作所造成的干扰
- 使用 snapshot 创建巨大文件集的 copy，或者创建当前状态的 checkpoint，便于后续改变不理想时进行回滚
- 采用 copy-on-write 的 lazy 模式实现 snapshot
   * 当 master 收到 snapshot 请求时，它会取消 snapshot 所涉及的文件的 outstanding 租约
   * 保证了后续的 writer client 都必须向 master 索取 lease primary，这就给 master 一个时机复制 chunk
   * master 取消 lease 后，master 通过复制文件、目录 metadata 进行快照操作，并把变化反映在内存状态中
     具体来说，就是把 snapshot 操作的目标文件、目录的引用计数 reference count + 1，这样就大于 1 了
   * 之后，当 client 想对相关文件进行 mutation 时，必须向请求当前 lease primary
     master 不会马上回复请求，而是复制出新的 chunk，并且要求每个相关的 replicas 都复制新的 chunk
	 区分一个 chunk 是否需要复制的标准，就看 reference count 是否大于 1。大于 1 的就是快照状态，要复制
	 这就是 copy on write，新的 chunk 用于响应 client 的后续请求和变化，而原来的 chunk 对应 snapshot
   * 看到 chunk 复制都发生在原来持有对应文件的 chunkservers 上，故此复制操作是本地进行，没有网络延迟
   * 这个过程中，client 完全不知道它是从一个已存在的 chunk 克隆出来的

##### 一个不一致的场景分析

- 比如数据 A、B、C 分别 append 到 primary 1 和 chunkservers 2 & 3 上，结果 server 3 上的 B 追加失败
- 由于 server 3 返回失败，故此客户端重试数据记录 B 的追加，于是，3 个 servers 的文件结构如下：
  primary 1 & server 2 为 [A | B | C | B]；而 chunkserver 3 为 [A | | C | B]，注意不是 [A | C | B]
- 注意，此时 1 & 2 服务器上就出现了所谓“重复”数据，而 3 上出现 padding，且结构和位置顺序是不一致的
  重复的数据也说明了，GFS 是 at least once 的，而不是 exactly once 的
- GFS 可以通过特殊字符和 checksum 的方式来识别 padding，通过在数据记录中加入唯一性标识来识别重复数据
  从而容忍 replica 之间的不一致，GFS 提供了一个库来 handle 这些情况
- GFS 放松了一致性的要求，导致了上面的情况发生，但是也换来了简单且效率很高的实现，因为强一致性成本很高
- 如果要实现强一致性，我们可能要做类似下面的事情和限制
   * 不允许多个 clients 的并行 mutations
   * primary 需要能够检测到重复数据，比如上面场景中，1 & 2 不应该允许数据记录 B 的第二次追加
   * master 可能要把有问题的 chunkservers 踢掉，比如让 server 3 停止服务，直到恢复到一致的状态
   * 当 primary 让其他 chunkservers 做操作时，chunkserver 先不要做任何事，而是告知 primary 可以做
     然后 primary 再次通知 chunkservers 可以去做了。这就是多阶段 commit 的模型，比如 2-phase commit
   * 如果 primary 掉线，那么 replica chunkservers 间可能会发生不一致。新的 primary 被指定后，
     必须联系所有 chunkservers，了解它们所做的操作，并完成所有 replicas 的一致
   * 为了避免 client 从 stale replica chunkserver 中读取到过时数据，限制只能从 primary 读取数据
   * 等等 ...，要实现强一致性，需要考虑很多种异常情况，而且要做其它方面的限制和妥协


### Master 节点的操作

master 执行所有 namespace 操作，管理 chunk replicas，协调系统行为，调度负载均衡和垃圾回收等

##### Namespace Management & Locking

- namespace 就是文件、目录命名规则。GFS 没有普通文件系统中的目录和对应操作，比如列出目录下所有文件。
  虽然文件路径非常像传统 linux 文件，比如 /root/abc/d 文件，其中 /root/abc/ 并不能说是个目录。
  /root/abc 这种前缀压缩，只是为了有效的减小 metadata 的大小，以便保存在内存中。
- 一些 master 操作是较花时间的，比如快照时取消相关 chunks 的 lease。为了不影响并发操作，需要加锁
- 比如某个 master operation 的对象是 /d1/d2/leaf，那么：
  要获取 /d1 和 /d1/d2 的读锁，以及 /d1/d2/leaf 的写锁。注意，这里 leaf 即可能是文件也可能是目录
- 举个具体的例子，比如对 /home/user snapshot 到 /save/user，此时又有请求创建 /home/user/foo
   * snapshot 操作下会创建 /home 和 /save 的读锁，以及 /home/user 和 /save/user 的写锁
   * 创建文件操作下会创建 /home 和 /home/user 的读锁，以及 /home/user/foo 的写锁
   * 由于在 /home/user 上的锁发生了冲突，那么创建文件下的读锁必须等到 snapshot 的写锁解锁
   * 同时，/home/user 上的读锁也足以保证这个目录不会被其他操作删除、改名
- 这种读写锁的设置，也能很好的支持同一个目录下，并发创建多个文件，而不相互干扰，且保证目录不会被删除
- 大数据下，namespace 可能过多，故此读写锁会 lazy 方式分配，而且一旦不再使用会被立即删除
- 最后，为了避免死锁，读写锁都是按 namespace 的目录级别按序获取，同级的锁按字母顺序获取

##### Replica Placement

- GFS 是高度分布式和大规模的，拥有众多的服务器，那么这些服务器会横跨多个机架 racks
- 那么，chunk replicas 位置选择的策略就是要最大化数据可靠性和最大化网络带宽利用率
- 必须在多个机架间储存 chunk 的副本，即使在整个机架被破坏或掉线的情况下依然保持可用状态
- 另一方面，必须付出写操作要在多个机架上的设备进行网络通信的代价

##### Creation, Re-replication, Rebalancing: 三种创建 chunk replica 的场景

1) 创建 chunk 时，会创建 chunk replica。如何选择创建 replica 的 chunkservers？
- 在低于平均硬盘使用率的 chunkservers 上，以平衡 chunkservers 间的硬盘使用率
- 限制每个 chunkservers 上近期创建 replicas 的次数，因为创建之后就将是数据写入，需要平均写入负担
- 把 replica 的创建工作分散在不同的机架之上

2) 当 chunk 的 replicas 数少于复制因数时，master 会重新复制它，创建新的 chunk replica
- 可能原因是：某 chunkserver 挂掉了、chunkserver 包括某个损坏的 chunk、chunk 的复制因数增加了
- 需要重新复制的 chunks 可能有很多，那么如何给它们分配优先级呢？
   * 现存 replica 数和复制因数之间的差距越大，优先级越高
   * 活跃的文件相较于最近刚被删除的文件有更好地优先级
   * 为了最小化失效 chunk 对现在运行的应用的影响，提高会被 client 应用程序使用的 chunks 的优先级
- master 排好优先级后，命令某个 chunkserver 直接从可用的 replicas 中 clone 一个新的副本出来
- 选择新副本位置的策略，和上面 1) 中创建 chunk 时的策略一致
- 为避免 clone 操作占用太多网络带宽，会限制整个集群和各个 chunkservers 同时进行的 clone 操作的数量
- chunkserver 也会通过调节它对源 replica chunkserver 的读请求的频率来限制自己 clone 操作的带宽

3) master 会周期性的负载均衡，此时也可能会创建 chunk replica
- master 会检查当前 replicas 的分布情况，移动 replica 以便更好地利用硬盘空间
- master 会逐渐填满一个新的 chunkserver，而不是在短时间内就用新的 chunks 填满它
- 过载时，master 要考虑哪个 replica 要被移走，策略是移走剩余空间低于平均值的 chunkserver 的 replica
- 至于移到哪儿，也就是新的 replica 放到哪儿，这个策略和 1) 中也是相同的

##### Garbage Collection

- 文件删除后不会立刻回收 reclaim 物理空间，而是采用惰性策略，在常规垃圾收集时进行文件和 chunk 级回收
- master 把文件删除操作视为改名操作，记录删除 log 后把文件改名为一个包含删除时间戳的隐藏名字
- 当 master 进行常规的文件 namespace 扫描时，会删除三天前的隐藏文件，此时该文件的 metadata 才真正被删除
- 直到真正删除之前，其实它们还可以用新的特殊名字读取，也可以通过把隐藏名字改为正常文件名来反删除
- 在对 chunk namespace 进行常规扫描时，还会找到孤儿 chunks，不被任何文件所包含，会删除它们的 metadata
- 具体的，chunkserver 的 Heartbeat 请求中会报告它拥有的 chunk 子集，master 回复其中哪些是孤儿可以删除

垃圾回收相较直接删除的优点：
- 在组件失效是常态的大规模分布式系统中，GC 更简单可靠，而且提供了一致、可靠的清除无用 replica 的方法
- GC 相关的扫描可以放到 master 空闲的时候，这样 master 可以提高对 client 的响应速度，减少延迟
- GC 为意外的、不可逆转的删除操作提供了安全和恢复的保障

##### Stale Replica Detection

- 前面说过 chunkserver 失效时会引起其 replica 错过某系 mutations 而失效。master 使用版本号来区分失效
- 只要 master 和 chunk 签订了新的 lease，就会增加版本号，然后通知最新的 replicas 都记录这个新的版本号
- 这些动作发生在任何 client 得到回复之前，以保证一致性。当 client 得到通知，一定是所有版本号都更新了
- 如果某个 chunkserver 曾失效而导致其上的 replica 失效，那么其 replica 的版本号不会被增加
- 下次这个 chunkserver 发起 Heartbeat 请求时，会汇报其 replica 子集及其版本号，于是 master 会检测出来
- 特殊情况：master 看到一个比它记录的版本号更高的版本号，它会认为 lease 操作失败，而选择该更高版本号
- 在失效的 replica 被 GC 清除之前，这些 replicas 不会被 client 访问，master 会当它们不存在
- 而且 master 给 client 回复 primary、replica 的 chunkserver 时也带上版本号，client 要检查后续数据版本


### FAULT TOLERANCE AND DIAGNOSIS

设计 GFS 时遇到的最大挑战之一是如何处理频繁发生的组件失效，以及进而产生的不完整数据
对于大规模分布式系统，不能完全依赖机器的稳定性，也不能完全相信硬盘的可靠性

##### 高可用性 High Availability 的关键 fast recovery & replication

- Fast Recovery
   * master 和 chunkserver 都设计为只要关闭了都会在数秒内恢复状态重新启动，不区分正常还是异常关闭
   * 通常直接 kill 进程来关闭服务器，clients 和其它服务器会感受到系统有一些颠簸 (hiccup)
   *具体来说，就是发出的请求会超时，需要重新连接到重启后的服务器后，重试这个请求
- chunk replica
   * 这个前面已经充分的介绍过了，包括 replica 的位置选择、checksum 和版本的验证、replica 复制等
- master replica
   * 为了保证 master 的可靠性，master 的状态也要复制，包括所有的 operation logs & checkpoints 
   * 对 master 状态的修改操作能够提交成功的前提是，operation logs 写入到 master 本机磁盘和备节点
   * 因此，master 失效时能立即重启；如 master 服务器、磁盘失效，会在有完整日志的机器上重启 master
   * client 通过规范的名字访问 master，master 改变位置时会更改别名，以让 client 自动连接新位置
   * 另外，GFS 中还存在 shadow master 节点，会在 master 失效时提供短暂的文件系统的只读访问
   * 平时，shadow master 为了保持状态的更新，会:
      + 读取当前正在进行的操作的日志副本，并且依照和 master 完全相同的顺序来更改内部的数据结构
	  + shadow master 也会在启动时从 chunkserver 轮询数据，并在之后定期拉取数据，维护 replica 信息
	  + 也会使用心跳的握手方式来确定 chunkserver 的状态
	  + 在 master 因创建和删除副本导致副本位置信息更新时，才会和 master 通信来更新自身状态

##### Data Integrity 数据完整性

- chunkserver 使用 checksum 来检查其中的 replica 数据是否损坏。其背景和原因如下：
   * 原则上可以通过别的 Chunk replica 来解决数据损坏，但跨越 Chunkserver 比较 replicas 很不实际
   * GFS 允许有歧义的副本存在，原子性 append 操作并不保证 replica byte-wise 完全一致
   * 由上两点，检查 replica 完整性的任务就落在每个 chunkserver 自己了，具体方法就是 checksum
- 每个 64 MB 的 chunk 又被分为 64 KB 大小的块，每个块对应一个 32 位 checksum
- checksum 和其它 metadata 一样，也是和用户数据分开的，并保存在内存和硬盘上，同时也记录 Operation log

- 对于读取数据的操作，chunkserver 会检查操作所涉及数据块的 checksum，如果有错误的数据：
   * chunkserver 不会把错误数据传递到其他机器上，而是返回给请求者错误信息；请求者则会去读取其他副本
   * 并且通知 master 这个错误，后者会从其它 replica clone 数据进行恢复
   * 当新的 replica 就绪后，master 会通知发生错误的 chunkserver 删掉错误的 replica
   
- checksum 的计算针对在 chunk 尾部的 append 写入操作做了高度优化，因为追加是 GFS 的绝大部分场景
   * 只增量更新最后一个不完整的块的 checksum，因为已完整的块的 checksum 已经在之前的增量更新中完成
   * 即使最后的不完整块的 checksum 已损坏，且错误还未被发现，没关系，下次该块被读取时校验数据会失败
- chunkserver 空闲时会扫描和检验平时不活跃的 chunks，以确保很少被读取的 chunks 的数据完整性
- 一旦检测到损坏，master 会重新创建 replica 并删除损坏的 replica

















   
   
   
   
   
   
   