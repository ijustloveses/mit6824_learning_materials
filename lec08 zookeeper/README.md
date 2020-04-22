# Lecture 8: Zookeeper Case Study

## FAQ

1. What does wait-free mean?
1. What is the reason for implementing 'fuzzy snapshots'? How can state changes be idempotent?
1. How does a client know when to leave a barrier (top of page 7)?
1. Is it possible to add more servers into an existing ZooKeeper without taking the service down for a period of time?
1. How are watches implemented in the client library?


## 论文研究和课堂笔记

### 整个 Zookeeper 系统架构如下，Zookeeper 的底层依赖一个 ZAB layer，和之前的 Raft 非常类似
                                                                 
                Leader                      Follower                   Follower 
            |-------------|             |-------------|            |-------------|
  C1 ------>|  |  ZK      |             |     ZK      |            |     ZK      |
            |--|----------|             |-------------|            |-------------|
  C2        |  V  ZAB     |             |    ZAB      |            |    ZAB      |
            |             |             |             |            |             |
  C3        | |---------| |             | |---------| |            | |---------| |
            | | | | ... |-------------->| | | | ... | |      |---->| | | | ... | |
            | |---------| |    |        | |---------| |      |     | |---------| |
            |             |    |        |             |      |     |             |
            |-------------|    |        |-------------|      |     |-------------| 
                               |-----------------------------|
           
		   
### 问题：如果确实有很多的客户端需要被服务，当我们增加服务器时，系统性能是否会随之增加?
- 写操作可能会变得更慢，why？因为 update 需要同步到更多的服务器上，显然会增加响应的延迟
- 读操作呢？如果还是像 Raft 那样，请求都要向 Leader 发起，那么增加服务器似乎没有什么意义
  + 如果能实现客户端从所连接的服务器直接读取数据，而不涉及到 Leader 及其它的 Replicas，性能会大幅提升
  + 从 Raft 的介绍中我们知道，follower 的数据可能是 staled、uncommitted 等情况，不符合 Linearizability
  + Zookeeper 允许从所连接的服务器直接读取数据，前提是允许读取 staled 数据，允许不满足 Linearizability

  
### 我们详细看一下 Zookeeper 对 Order Guarantees 方面的设计
- Linearizable Writes：写操作是保证 Linearizability
  + 客户端可能并发发起写请求，但所有请求都会由 Leader 处理
  + Leader 会为并发的写请求安排顺序, numbered by "zxid" (对应 Log entry)，并保证一个完成了才进行下一个 
  + 把每个写请求的结果发送给 replicas，后者都会按 zxid 的顺序来执行
  + 把结果而不是操作本身发送给 replicas 的好处是：实现了幂等性，多次执行同一个 zxid 结果也不会发生改变
- FIFO client order：对任何一个特定的客户端所发送的请求，Zookeeper 保证这些请求都按发送的顺序被执行
  + 无论是读操作还是写操作，对于一个客户端来说，其请求都是顺序执行的
  + 潜台词：不同的客户端的操作并不一定能够保证时间顺序，前面说过多个客户端并发的请求由 Leader 安排顺序
  + 自然的，写操作都是按 client-specified order 顺序执行，前面说过，都是按 zxid 的顺序执行
  + 对读操作，由于客户端允许从连接的服务器直接读取数据，而不通过 Leader，事情就 tricky 一些了
    - Zookeeper 同样要保证读操作在请求序列中的顺序
	- 比如写入一个数据后马上读取，那么读取数据时“当前状态对应的 zxid” 一定“不能低于”写操作时的 zxid
	- 如果此时 replica 还没有同步好 Leader 的最新状态，那么读操作必须等待，等着前面的写操作都执行完
	- 即使客户端连接的服务器挂掉，重连到其他服务器，也必须等待那个服务器上执行完前面的写操作之后才能读
  + 我们看到，Zookeeper 只保证同一个客户端自身的请求顺序，并不保证其某一个请求的结果一定是当时的最新状态
  + 具体来说，读操作的结果可能并不是最新的结果，但是对于这个客户端来说足够新了，反映了它的写操作的结果
  + 这是一种松弛的 Linearizability，不同于严格的 Linearizability (Async Linearizability)
  + 例子：客户端 C1 依次发起 W1 & R1 请求，同时 C2 发起 W2 请求，Leader 安排的写入顺序为 W1 W2
    - 假如 R1 请求实际上是 W2 请求之后发送的，那么严格的 Linearizability 要求 R1 返回 W2 的结果
	- Zookeeper 只保证不返回 W1 之前的写入数据，也就是说返回 W1 或者 W2 的结果都认为是对的
- Sync：如果客户端确实希望得到当前的最新状态下的数据结果
  + 上面的例子中，如果我们希望得到 W2 的结果怎么办？调用 sync() 再调用读请求
  + sync() 请求保证客户端所连接的 replica 首先同步到 sync 请求时的最新状态之后，才处理其后的读请求

  
### Zookeeper 的 Order Guarantees 如何实现所维护的数据 Atomic Update？
- 如 ZK 实现配置管理服务，Master 维护一系列配置数据，如何保证更新配置时，workers 不会看到部分更新的结果
- 在 ZK 中维护一个额外的 "ready" 文件
  + 读操作时首先 check exists("ready")，如果有这个文件，就可以读，FIFO 保证读到之前的写入的配置数据
  + 写操作首先删除 ready 文件，再进行配置数据写入，最后创建 ready 文件，以保证可以开始进行读操作
- 比如在新的配置还未更新完毕时，Leader crash 了，如何保证之前进行的一部分更新结果不会被 workers 看到？
  + 显然的，如果 Leader 在更新完毕前挂掉，那么就不会生成 ready 文件，故此读操作不允许访问配置
- 然而，还有个意外情况：读操作首先 check ready 文件发现存在，然后写操作开始删除 ready 文件并开始更新
  + 如何防止读操作读取到部分更新的配置文件呢？ZK 的解决方法是提供一个 watch 机制
  + 在读操作 check ready exists 的时候，会同时放一个 watch 标志，每次 ready 发生变化，客户端会被通知
  + 一旦写操作开始执行，删除 ready 文件，此时，读操作的客户端马上得到通知，客户端需要写逻辑处理这种情况！
  + 而且，ZK 的 FIFO 机制保证上面的这个通知会先于该客户端后续对配置的读请求之前到来
    ```
         Write order:      Read order:
                           exists("ready", watch=true)
                           read f1
         delete("ready")
         write f1
         write f2
                           read f2
    ```
- 为保证 ZK 的机制正确运行，需要
  + Leader 必须安排和保存客户端的 write 顺序，across leader failure
  + Replicas 必须保证每个客户端的读操作不会读到对于该客户端的 staled 数据，despite replica failure
    还会维护 watch table。注意 watch table 不会同步到所有服务器；反正 replica 一旦挂了客户端是知道的
  + 客户端必须 track 其当前最高 zxid，以保证下一个读操作不会读取到 staled 数据，即使连接不同的 replica

	
### ZK 的 API 设计得非常巧妙，帮助使用者设计各种分布式系统应用
- create(path, data, flags)
  + exclusive -- 如果 path 存在返回 no；只有成功创建才返回 yes；这样并发的创建就只有一个客户端能成功
- delete(path, version)
  + if znode.version = version, then delete
- exists(path, watch)
  + watch=true means also send notification if path is later created/deleted
- getData(path, watch)
- setData(path, data, version)
  + if znode.version = version, then update
- getChildren(path, watch)
- sync()
  + "sync then read" 保证所有客户端在这个 sync 之前的写入都会被发送 sync 的这个客户端所看到


### ZK API 能实现的一些例子
- Example: add one to a number stored in a ZooKeeper znode
  + 我们想一下普通的接口比如 SET(k, v) 和 GET(k) - >v 为什么不能正常的处理
    * 客户端 C1 发起 v = GET(k)，并增加该 k 的数值 SET(k, v+1)；同时 C2 也发起了完全一样的请求
	* 如果请求的顺序是 C1.GET -> C2.GET -> C1.SET -> C2.SET，那么 k 值对应的数值会最终为 v+1
	* 如果请求的顺序是 C1.GET -> C1.SET -> C2.GET -> C2.SET，那么 k 值对应的数值会最终为 v+2
	* 无法明确的判断以上两种并发请求得到的最终结果，这是非常不好的设计
  + 利用 ZK 的 API 就能通过 version 来解决这个问题
  ```
  	while true:                          // 一次无法保证操作正常处理完毕，故此使用循环
        x, v := getData("f")             // 获取数据值以及版本
        if setData(x + 1, version=v):    // 设置数值 + 1，并保证 version 一致；如果设置成功，那么结束
            break                        // 否则，说明其他客户端已经设置过了，回头重新读取数据和新版本
  ```
  + 这种 "mini-transaction" effect 保证了 atomic read-modify-write

- Example: Simple Locks (Section 2.4)
  acquire():    // 获取锁函数
    while true:
      if create("lf", ephemeral=true), success    // 一旦成功创建 lockfile 则成功获取
      if exists("lf", watch=true)    // 再次检查，如存在则设置 watch，等待 lockfile 删除；否则下一轮循环
        wait for notification        // 一旦收到消息，立即进入下一轮循环，尝试创建 lockfile
		
  + 这里，exists("lf", watch=true) 的原子性非常重要，保证如果存在就设置 watch，中间不会有其他客户端操作
  + 否则，比如在检查到存在和设置 watch 之间其他客户端删掉了 lockfile，则这个客户端就等不到这条删除消息了

  release(): (voluntarily or session timeout)    // 释放锁，简单的删除 lockfile 即可
    delete("lf")

- 前面两个例子都存在 herd effect：除了成功的客户端，其他客户端都会一起重新发起新请求，造成服务器访问压力
	
- Example: Locks without Herd Effect (look at pseudo-code in paper, Section 2.4, page 6)
  acquire():      // 获取锁函数
    1. create a "sequential" file："lf-#{n}"
    2. list files，获取当前全部的 sequential files
    3. if no lower-numbered, lock is acquired!    // 小于n的文件都被删除了，也即对应的锁都释放了，成功
    4. if exists(next-lower-numbered, watch=true) // 否则，就找到小于 n 的文件中最大的那个，watch 它
    5.   wait for event...                        // 一旦获取，此刻小于 n 的那些文件的锁"可能"都释放了
    6. goto 2                                     // 那么，重新获取文件列表，并开始比较
	
  release(): 
    delete("lf-#{n}")   // 删除其对应的那个 lockfile
	
  + ZK 保证 steps 2 and 3 之间系统不会产生其他小于 n 的 sequential file
  + 注意，next-lower-numbered 对应客户端不一定是获取了锁，也可能是挂掉了，故此需要回到 2 重新进行判断
     lock-10 <- current lock holder
     lock-11 <- next one
     lock-12 <- my request
	 此时，lock-11 挂掉了，但是其实 lock-12 还没有获得锁的权力，故此必须重新判断，会重新 watch lock-10

  
### 一些实现方面的细节
- 读请求：非常简单，直接返回服务器本地 cache 中的结果
- 写请求：Request Processor --> Atomic BroadCast --> Replicated Database 
- replicated 数据的层次
  + in-memory replicated database (1MB for each tree-node by default)
  + write-ahead replay log，ZK 保证在写入 replay log 之后才写入 in-memory database
  + periodic snapshots of in-memory database
- 客户端就只连接某一个服务器，该服务器通过 watch table 以及 zxid 维护客户端的 FIFO 
- 前面说过，如果客户端连接的服务器挂了，那么 watch table 就丢失了，客户端连接新的服务器，并开始新的请求
  + 如果新服务器的 last zxid 比客户端的还要小，那么在服务器赶上进度之前，不会和这个客户端真正建立联系
  + ZK 保证客户端能够找到另一个服务器，使得其 last zxid >= 客户端的 last zxid

##### Request Processor
- 只强调一点：为了保证 idempotent 幂等性，Leader 会首先根据写请求计算出完成后的系统状态并把状态发送出去
- 幂等性使得请求可以多次被处理，只要保证所有请求都是按顺序来执行的

##### Atomic BroadCast
- ZAB atomic broadcast protocol 默认使用简单的 majority quorums 来决定一个 proposal 是否 committed
- ZAB 保证所有客户端的写请求按序执行，如更换 leader，新 leader 在同步老 leader 的更新后才广播自己的更新

##### Replicated Database 
- 为了更快的从失败中恢复，使用 periodic snapshots，这样恢复时只要从最新的 snapshot 加上之后的 logs 即可
- ZK 使用所谓的 fuzzy snapshots，就是说在进行 snapshot 时，并不锁住 ZK 的系统状态
- 上述的结果是：snapshot 中的状态甚至可能不对应任一时刻系统的状态 ...
  + 举个例子，初始状态 {/foo f1 1 + /goo g1 1}  (三个字段分别为 path value version) 进行 snapshot
    * 状态 t1： SetDataTXN(/foo f2 2) ==> {/foo f2 2 + /goo g1 1} 
    * 状态 t2： SetDataTXN(/goo g2 2) ==> {/foo f2 2 + /goo g2 2} 
    * 状态 t3： SetDataTXN(/foo f3 3) ==> {/foo f3 3 + /goo g2 2} 
    * snapshot 完毕时，系统状态甚至可能是 {/foo f3 3 + /goo g1 1}，看到和上面任何时刻的状态都不同
  + 系统知道进行 snapshot 时状态为 {/foo f1 1 + /goo g1 1} 以及后续的 t1 t2 t3 3 次请求，完全可以恢复状态

##### Client/Server Interaction
- Leader 按顺序处理写请求时，不会并发处理其他写请求和读请求，这保证了 Write Linearizability 和 watch 机制
- 每个读请求会使用当时服务器上的最新的 transaction 的 zxid 来标记，这样可以确定读请求在写请求中的位置
- 论文中还在这部分中大概介绍了一些 sync 的实现方式，写的不是很清楚
- Leader 通过 timeout 能够意识到自己已经不再是 leader，而且还是在 followers 放弃它之前就能意识到
  
  
### ZK 实现的分布式锁并不能保证原子性的 transaction
- ZK 一旦获得锁，确实就进入了所谓 critical section，但在后续操作的时候有可能发生网络错误甚至 crash
- 这样即使释放了锁，很可能使得系统处于一个 partial update 的状态
- 故此，ZK 不适合需要原子性更新的系统，一般适合 ZK 的系统有两种
  + 1. 系统有能力在处理之前先判断是否处于非正常状态，如果是的话，知道如何 fix it 再进行后续处理
  + 2. 所谓 soft lock，只 protect something not really matter，比如 mapreduce 中 worker 出错也没关系

  
References:
  https://zookeeper.apache.org/doc/r3.4.8/api/org/apache/zookeeper/ZooKeeper.html
  ZAB: http://dl.acm.org/citation.cfm?id=2056409
  https://zookeeper.apache.org/
  https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf  (wait free, universal objects, etc.)
