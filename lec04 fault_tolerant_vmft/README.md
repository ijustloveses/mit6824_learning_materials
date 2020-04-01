# Lecture 4: Primary/Backup Replication

## FAQ

1. The introduction says that it is more difficult to ensure deterministic execution on physical servers than on VMs. Why is this the case?
1. What is a hypervisor?
1. Both GFS and VMware FT provide fault tolerance. How should we think about when one or the other is better?
1. How do Section 3.4's bounce buffers help avoid races?
1. What is "an atomic test-and-set operation on the shared storage"?
1. How much performance is lost by following the Output Rule?
1. What if the application calls a random number generator? Won't that yield different results on primary and backup and cause the executions to diverge?
1. How were the creators certain that they captured all possible forms of non-determinism?
1. What happens if the primary fails just after it sends output to the external world?
1. Section 3.4 talks about disk I/Os that are outstanding on the primary when a failure happens; it says "Instead, we re-issue the pending I/Os during the go-live process of the backup VM." Where are the pending I/Os located/stored, and how far back does the re-issuing need to go?
1. How secure is this system?
1. Is it reasonable to address only the fail-stop failures? What are other type of failures?


论文阅读
==========

论文来自 VMWare，介绍了一个自动容错的虚拟机主从备份系统

目标：实现一台server的备份，使得primary挂了之后备份机能马上接手继续运行，而客户端不受影响

通常有两种做法
1. 复制状态：状态包含很多内容，CPU寄存器、内存等，仅一个内存就大到没有办法足够快的复制了
2. 复制操作指令：就是所谓的 Replicated State Machine，把 server 看成一个确定性状态机
   a) 如果 primary 和 backup 的初始状态一致，且后续指令明确而且一致，那么最后达到的状态也一致
   b) 多数分布式系统论文里，这个“状态转移”是比较high level的（比如插入一条数据库记录）
   c) 本文把它推向了一个low level的极端：状态转移基本就是一条x86 CPU指令
   d) 因为主从服务都是跑在可控的虚拟机上（VMWare的老本行），这个方案是可行的
   e) 因此，理论上任何可以在x86 CPU上跑的程序，不用做任何修改就可以自动变成带主从容错的
   
- 核心是消除非确定性的操作影响，简单地复制CPU指令是不够的，需要小心处理随机数生成、时钟中断等
  + 这里的挑战就是能够正确的记录这些非确定性的操作和输入，以能够确定性的在 backup 上重放
  + 因为程序是跑在可控的VM上的，我们总有办法把在primary上发生的这些事件记录下来，在backup上面重放
     * VMware deterministic replay for x86 VM 会记录输入和所有相关联的不确定性，并保存在 log 文件
	   这样，只需要读取文件，并执行重放 replay 即可
	 * 对于非确定性操作，比如计时器 timer 和 IO 完成中断等，同样会在事件发生的时间被记录下来
	   并在 replay 时，这些事件同样会在指令流中的相同时间点上被触发，保证系统保持一致的状态
  + 对于 VM-FT，primary 产生的 log 不写在 disk 上，而是通过一个 logging channel 写入 backup
  + backup 就可以重放这些 log，以达到和 primary 相一致的状态

- 另外一个有趣的点是如何处理向外部的输出
  + 在正常运行的运行的情况下，输入输出都是通过primary的，备份机的输出指令会被 hypervisor 忽略
  + primary挂掉的情况下备份机就要接管过来，并且从外界不能看出异常
  + Output Rule：primary 并不马上输出，先放到log里，直到backup确认收到这条log后再真的输出给外界
  + 注意：这样不能避免重复输出的情况，可能primary输出之后挂掉了，backup接手之后又输出了一次
  + 认为重复输出可接受，因为输出设备通常允许这样（如TCP能判断重复序列号，磁盘允许重复写同样内容）

还有可能 primary 正常收到输入但新接手的 backup 不知道，这种也被认为是可接受的，好比TCP允许丢包

- 处理 Split Brain (primary和backup之间的通信断了，但两个其实都活着)
  + 让primary和backup共享的磁盘处理
  + 脑裂时，我们用它们共享的磁盘上的Test-And-Set原子操作来保证只能有一个是primary
     * 逻辑就是：如果谁不能访问到这个磁盘，就不算活着
	 * Test-And-Set 成功，说明没有占用，故此可以放心的提供 primary 服务
	 * 如果失败，说明磁盘已经被占用，那么 backup VM 就 halt itself，停止 go live 上线

- primary 和 backup VM 之间的 log channel
  + 由 primary 提供，并写入操作；backup 会读取 channel 中的操作并执行
  + 如果 backup 发现 channel 中没有 log entries 了，那么会 hold 等待新操作的到来
  + 如果 primary 发现 channel 满了， 也会暂停执行，等待 backup 消化 channel 中的操作
  + 其核心宗旨是认为：primary & backup VM 两者执行操作的效率应该是接近的，故此堆积或者空都不正常
  + channel 空，说明 primary 负载过重
  + 操作堆积说明 backup 能力不足，一旦 primary 掉线，backup 要很久才能处理完操作，转为 primary
  + time for backup to go live == failure detection time + backup operation execution lag time
  + VM-FT 还会检测两者操作之间的 lag，如果 lag 过大会降低 primary 运行的 cpu limit 避免差距过大

- VMotion
  + primary 启动后，或者 backup 成为 Primary 后，VMotion 负责创建一个和 primary 相同状态的 backup
  + VMotion将虚拟机复制到另一个物理服务器上，将源 VM 作为 primary，将目标 VM 作为 backup
  + backup 的操作都通过同步并执行 primary 发送的 log 来实现，包括重启、关机、资源分配等
 
- Disk I/O
  + 磁盘操作是非阻塞的，所以可以并行写入，但是由此引入了不确定性。解决方案是强制磁盘操作串行进行
  + 磁盘操作（DMA）和应用程序会并行操作同一块内存，引发数据竞争
     * 方案是使用bounce buffer，读取磁盘时先将数据读入额外缓冲，写入磁盘时先将数据复制到额外缓冲
  + 当 primary 故障，backup 替代时，磁盘I/O可能没有完成
     * 方案是重新执行磁盘操作，因为前两个方案已经避免了数据竞争，因此磁盘操作是可重入的
	 
- 网络 I/O
  + vSphere实现了一些网络方面的优化，如直接从 hypervisor 取走数据，但这带来不确定，故此此异步优化
  + 通过批量操作降低虚拟机 trap 和中断次数
  + 降低传输数据包的延迟：将发送操作和接收操作注册到TCP协议栈之中，保证立即发送和接收日志
  
- 存在的问题
  + benchmark给出的数据是需要20Mbit/s的带宽，这个对于很多互联网服务器并不是很够
      * 可以用于吞吐量不巨大但很重要的服务，尤其是在你不能随便改这个服务的代码的情况下
  + 搞不定多核处理器，因为此时要消除非确定性的代价太大了
      * 作者argue说你可以搞多台单核，scale out而不是scale up

