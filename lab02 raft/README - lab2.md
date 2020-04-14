Lab 02
========

[Lab 02 的上机说明](6.824 Lab 2: Raft.html)

## Lab 02 准备工作

- 阅读 [extended Raft paper](raft-extended.pdf)
- 阅读 [LEC 05](../LEC05/README.md) 和 [LEC 06](../LEC06/README.md)
- 阅读 <http://thesecretlivesofdata.com/raft/>
- 阅读 [Students' Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/)
- 阅读 [Raft Locking Advice](raft-locking.txt)
- 阅读 [Raft Structure Advice](raft-structure.txt)
- 选读 [Paxos Replicated State Machines as the Basis of a High-Performance Data Store](Bolosky.pdf)


## 代码阅读 之 labgob.go

- 该 go 文件定义了一个 encoding/gob 的 wrapper
- wrapper 的目的就是为了保证
  + 由 encoding/gob 编码解码的结构都为大写字母开头
  + 检查传入 RPC 请求的 reply 实例是否带有非默认值的字段

### encoding/gob

- https://golang.org/pkg/encoding/gob/
- https://www.jianshu.com/p/e8166a20935a
- https://www.cnblogs.com/yjf512/archive/2012/08/24/2653697.html

- gob是Golang包自带的一个数据结构序列化的编码/解码工具
- 结构体中缺省的字段将不会被发送
- 在接收端，并不需要所有的字段都要有对应的结构属性对应
- 接收端只接受和发送数据“相似”的数据结构。允许模拟相似，但是不允许矛盾
- Struct中的属性应该是public的，即应该是大写字母开头

### labgob 作为编码解码器所需要实现的接口

##### 全局变量

var mu sync.Mutex
var errorCount int
var checked map[reflect.Type]bool

##### LabEncoder & LabDecoder 结构体

非常简单，其实就是一个结构体，里面封装了 gob.Encoder & gob.Decoder 的指针

##### NewEncoder & NewDecoder 接口函数

调用 gob.NewEncoder/NewDecoder 来初始化 LabEncoder/LabDecoder 的实例并返回

##### Encode(e interface{})

在调用 gob.Encode(e) 之前，先检查一下 checkValue(e)

##### EncodeValue(value reflect.Value) 

在调用 gob.EncodeValue(value) 之前，先检查一下 checkValue(value.Interface())

##### Decode(e interface{})

- 在调用 gob.Decode(e) 之前，先检查 checkValue(e) 和 checkDefault(e)
- 注意：这里 check 是在 Decode 之前进行的，就是说，check 的其实是传到 RPC reply 中的原值
- 调用者可能初始化一个 reply 实例，或者重用之前的 reply 实例，放到 RPC 请求中，返回值会覆盖该实例

##### Register(e interface{})

在调用 gob.Register(e) 之前，先检查一下 checkValue(e)

##### RegisterName(name string, value interface{})

在调用 gob.RegisterName(name, value) 之前，先检查一下 checkValue(value)


##### 核心函数 checkValue(value interface{})

直接返回 checkType(reflect.TypeOf(value))

##### 核心函数 checkType(t reflect.Type)

- 如果没有初始化 checked 变量，那么就初始化之
- checked[t] 存在，说明检查过了，直接返回
- 对 k = t.Kind() 进行分类讨论
  + reflect.Slice, reflect.Array, reflect.Ptr：递归检查其 checkType(t.Elem())
  + reflect.Map：递归检查其 checkType(t.Elem()) & checkType(t.Key())
  + 如果是 reflect.Struct，这个是重点检查目标，看其字段是否大写开头
    * 遍历 f := t.Field(i)，  i in [0, t.NumField())
    * 使用 utf8.DecodeRuneInString(f.Name) 获得字段名的第一个字符
	* 如果不是大写 unicode.IsUpper(rune) == false，则报警，errorCount ++1
	* 最后，还要递归调用检查其字段类型 checkType(f.Type)

我们看到，其实所谓的检查，只是记录有多少个错误，并且输出报警信息而已

##### 核心函数 checkDefault(value interface{})

- 对于 Decode 函数，除了 checkValue 之外，还要求 checkDefault，检测缺省值的部分
- 如果 RPC 的 reply 中包含默认值，那么这个默认值不会覆盖掉之前的非默认值，这是个 bug
- bug 场景：
  + 在多个 RPC 调用中重用一个 reply 的实例，在某个请求中，该实例的一些字段被设置了非默认值
  + 接下来的 RPC 调用中，该实例再次被使用，然后请求的回复中，把该实例的对应字段设置了默认值
  + bug 就是：返回的默认值无法覆盖之前请求中已经被设置了的非默认值
- 在 Decode 函数中被调用，用于检查传入 RPC 请求的实例是否有非默认值，如有，就有无法覆盖的风险
- 直接调用 checkDefault1(reflect.ValueOf(value), 1, "")

##### 核心函数 checkDefault1(value reflect.Value, depth int, name string)

- depth 3 层之上，直接返回
- 获取 value 的 t := value.Type() 和 k := t.Kind()，并对 k 分情况讨论
- k == reflect.Ptr 指针类型
  + 空指针直接返回
  + 递归调用 checkDefault1(value.Elem(), depth+1, name)
- k == reflect.Struct 结构体类型
  + 遍历 vv := value.Field(i)，name1 := t.Field(i).Name   for i in [0, t.NumField())
  + 如果参数中的 name 不等于 ""，那么 name1 = name + "." + name1
  + 然后递归调用 checkDefault1(vv, depth+1, name1)
- k == reflect.Bool / reflect.Int / ... / reflect.String 等这些基础数据类型
  + 调用 reflect.DeepEqual(reflect.Zero(t).Interface(), value.Interface())
  + 如果 false，说明传入请求的 value 不是默认值 reflect.Zero(t)，那么有风险，报警 errorCount++1


## 代码阅读 之 labrpc.go

- 实现了基于 channel 的模拟 RPC 实现，并没有真正走网络，而是通过 channel 发送请求和获取结果
- adapted from Go net/rpc/server.go 
- 使用了很多反射的方法

### 请求和回复的结构体

type reqMsg struct {
	endname  interface{}     // name of sending ClientEnd
	svcMeth  string          // e.g. "Raft.AppendEntries"
	argsType reflect.Type
	args     []byte          // 请求发送的数据，看到是 byte 数组，由 labgob Encode 之后的结果
	replyCh  chan replyMsg   // 回复返回的数据发送到这个 channel 中
}

type replyMsg struct {
	ok    bool               // 回复是否成功
	reply []byte             // 回复的数据，同样是 byte 数组，同样也是由 labgob Encode 的结果
}


### 发送请求的客户端

type ClientEnd struct {
	endname interface{}      // this end-point's name
	ch      chan reqMsg      // 其实是 Network.endCh channel 的拷贝，用于向服务端 (Network) 发送请求
	done    chan struct{}    // 如果该 channel 接收到消息，说明服务端已经关闭
}

ClientEnd.Call(svcMeth string, args interface{}, reply interface{}) bool 发送请求并接受结果的函数
- 首先构建一个 reqMsg 结构体，简单的初始化 endname / svcMeth / argsType 字段，并 make 新的 replyCh
- 使用 labgob 来把 args Encode 为 []byte
- 发送请求，方法是
```
	select {
	case e.ch <- req:
	case <-e.done:
		return false
	}
```
- 这两个 case 都是阻塞操作，要么 req 成功的发送到 ClientEnd.ch 请求 channel 中，要么发现服务端已经关闭
- 如果服务器关闭，那么就直接返回 false，说明请求失败
- 阻塞的方式等待回复 channel 中的结果 rep := <-req.replyCh
- 如果 rep.ok == false，那么返回 false，说明请求失败
- 反之，说明服务端成功处理请求，那么使用 labgob 把 rep.reply 解码到传入的 reply 变量中返回

那么，ClientEnd.ch 怎么来的？把 reqMsg 发送到 ch 后，又是如何被服务器端处理的呢？下面开始介绍服务端


### 最底层的结构体 Service，本质就是对一个提供 RPC 接口的 object 的进一步抽象

type Service struct {
	name    string
	rcvr    reflect.Value
	typ     reflect.Type
	methods map[string]reflect.Method
}

##### 利用反射，由 object 来生成一个 Service 的函数：MakeService(rcvr interface{}) *Service

- 初始化一个空的 Service 指针 svc := &Service{}
- svc.typ = reflect.TypeOf(rcvr)
- svc.rcvr = reflect.ValueOf(rcvr)
- svc.name = reflect.Indirect(svc.rcvr).Type().Name()
- 初始化 svc.methods = map[string]reflect.Method{}，并开始遍历该 object rcvr 的 methods
  + method := svc.typ.Method(i)
  + mtype := method.Type
  + mname := method.Name
  + 满足 method.PkgPath != "" 或通过 mtype 发现该 method 的参数签名不符合 golang RPC 接口规范，则放弃
  + 否则，method 符合作为 golang RPC 接口的规范，那么 svc.methods[mname] = method

##### Service.dispatch(methname string, req reqMsg) replyMsg 函数，真正对请求进行处理的最底层逻辑

- 在 svc.methods[methname] 中查看是否有 methname 对应的接口，如果没有，报错，返回 replyMsg{false, nil}
- 通过 req.argsType 初始化一个空的参数 args := reflect.New(req.argsType)
- 使用 labgob 把 []byte 类型的 req.args 解码到 args 中
- 通过 svc.methods[methname].Type.In(2) 来初始化回复类型的实例 replyv
- 调用接口执行请求 svc.methods[methname].Func.Call([]reflect.Value{svc.rcvr, args.Elem(), replyv})
- 利用 labgob 再把 replyv 编码为 []byte ，并返回 replyMsg{true, 编码后的[]byte 结果}

我们可以看到，目前其实整个数据流已经流通了：
1. 客户端把请求编码，发送到 channel 中
2. 服务器端通过什么方法从 channel 中取出请求字符串，送到 Service.dispatch 中处理
3. Service.dispatch 解码出请求变量，然后调用服务端的接口函数，然后把处理结果编码为字符串，返回
4. 后续服务器端又通过什么方法把结果字符串返回给客户端，客户端解码得到最终的结果
显然，上述流程中，执行的代码流还有一些地方不够清楚，我们需要继续看服务器端的处理


### Server - 一个服务同时响应多个 Services 的请求

type Server struct {
	mu       sync.Mutex
	services map[string]*Service     // 所支持的 Services 字典
	count    int                     // 统计 incoming RPCs 次数
}

##### 辅助函数 MakeServer() & Server.AddService(svc *Service) & Server.GetCount()

- MakeServer 函数用于初始化一个 Server 实例并返回，该实例的 services 列表为空
- Server.AddService(svc *Service) 添加 services 列表，简单的加锁并调用 rs.services[svc.name] = svc 即可
- Server.GetCount() 函数简单的加锁并返回 count 即可

##### Server.dispatch(req reqMsg) replyMsg 函数把 req 请求转给对应的 service 来处理

- 计数 rs.count += 1
- 由 req.svcMeth (比如 Raft.AppendEntries) 拆分出 serviceName 和 methodName 两部分
- 从 rs.services 中找到对应的  service, ok := rs.services[serviceName]
- 如果找不到，直接返回失败 replyMsg{false, nil}
- 如果找到了，直接调用 service.dispatch(methodName, req) 并返回结果即可


### 最顶层的结构体，包装 Server 和 ClientEnd，实现整个控制流

type Network struct {
	mu             sync.Mutex
	reliable       bool                        // Network 是否可靠
	longDelays     bool                        // pause a long time on send on disabled connection
	longReordering bool                        // 通过随机 sleep 使不同请求的处理顺序和到达顺序不一致
	ends           map[interface{}]*ClientEnd  // endname => ClientEnd
	enabled        map[interface{}]bool        // endname => enabled (true or false)
	servers        map[interface{}]*Server     // servername => server
	connections    map[interface{}]interface{} // endname => servername，把 ClientEnd 和 Server 绑定
	endCh          chan reqMsg                 // 读取 ClientEnd 的请求，所有 ClientEnd 共享这个 channel
	done           chan struct{}               // Network 关闭时，其工作协程以及所有 ClientEnd 调用返回
	count          int32                       // total RPC count, 统计计数
	bytes          int64                       // total bytes send, 统计计数
}

##### MakeNetwork() *Network 函数

- 用于构建一个空的 Network 结构体实例，默认 reliable = True，并构建 endCh & done 两个 channels
- 启动一个协程，进行对请求的处理
```
		for {                    // 无限循环，直到 Network 关闭
			select {
			case xreq := <-rn.endCh:               // 如果 rn.endCh 收到请求
				atomic.AddInt32(&rn.count, 1)      // 增加 rn.count、计算并增加 rn.bytes 计数
				atomic.AddInt64(&rn.bytes, int64(len(xreq.args)))
				go rn.processReq(xreq)             // 启动协程，处理请求，详见后续说明；然后下一轮循环
			case <-rn.done:
				return           // 如果 rn.done channel 收到数据，说明 Network 关闭，退出循环结束协程
			}
		}
```

##### (rn *Network) MakeEnd(endname interface{}) *ClientEnd 函数

- 用于在 Network rn 中创建一个 ClientEnd 结构体实例，创建之前首先确保 rn.ends[endname] 并不存在
- 初始化一个 ClientEnd 实例 e，名字为 e.endname = endname
- e.ch = rn.endCh，这样 e 就可以通过 rn.endCh channel 向 Network 的工作协程发送请求了
  + 同时，显然 Network 如果维护多个 ClientEnd，那么这些 ends 都可以通过同一个 channel 发送请求
  + 更进一步，我们看到 reqMsg 结构体中有 endname 字段，就是为了区分 channel 中请求到底由哪个 end 发送
- e.done = rn.done，这样 rn.done 被关闭时，不仅 Network 的工作协程会退出，也会让所有 ClientEnd.Call 返回
- rn.ends[endname] = e 维护该 ClientEnd
- rn.enabled[endname] = false & rn.connections[endname] = nil 默认该 ClientEnd 还未开启正常状态

##### 一些简单的辅助函数

- func (rn *Network) Cleanup() 直接调用 close(rn.done)，这会给 rn.done 发送一个关闭信息，使得上面协程退出

- (rn *Network) Reliable(yes bool) 函数加锁并设置 rn.reliable = yes
- (rn *Network) LongReordering(yes bool) 函数加锁并设置 rn.longReordering = yes
- (rn *Network) LongDelays(yes bool) 函数加锁并设置 rn.longDelays = yes
- (rn *Network) Enable(endname interface{}, enabled bool) 函数加锁并设置 rn.enabled[endname] = enabled

- (rn *Network) AddServer(servername interface{}, rs *Server) 函数加锁并添加 rn.servers[servername] = rs
- (rn *Network) DeleteServer(servername interface{})  函数加锁并删除 rn.servers[servername] = nil
- (rn *Network) Connect(endname interface{}, servername interface{}) 加锁并绑定 ClientEnd & Server
  + rn.connections[endname] = servername

- (rn *Network) GetCount(servername interface{}) 加锁并返回 rn.servers[servername].GetCount()
- (rn *Network) GetTotalCount() Atomically 返回 int(atomic.LoadInt32(&rn.count))
- (rn *Network) GetTotalBytes() Atomically 返回 atomic.LoadInt64(&rn.bytes)

- (rn *Network) isServerDead(endname interface{}, servername interface{}, server *Server) 函数
  + rn.enabled[endname] == false 说明 dead，返回 true
  + rn.servers[servername] != server 说明 dead，返回 true  (满足该条件，说明调用过 DeleteServer)
  + 其他情况，说明 Server 还活着，返回 false
  + 通过上述逻辑，如果一个 ClientEnd connect 了一个 Server，则可以由 rn.enabled[endname] 找到 Server

- (rn *Network) readEndnameInfo(endname interface{}) 函数会根据 endname 返回一系列能得到的信息
  + enabled = rn.enabled[endname]            // 当前状态
  + servername = rn.connections[endname]     // 所 connect 的 Server name
  + server = rn.servers[servername]          // 所 connect 的 Server
  + reliable = rn.reliable                   // 可靠性
  + longreordering = rn.longReordering       // 顺序是否一致

##### 核心函数 (rn *Network) processReq(req reqMsg) 也就是 Network 如何处理请求
  
- 先获取相关数据 enabled, servername, server, reliable, longreordering := rn.readEndnameInfo(req.endname)
- 首先看异常情况：enabled == false 或者 servername/server 其一是 nil 也就是说没有绑定 Server
  + if rn.longDelays： ms = (rand.Int() % 7000)
  + else：ms = (rand.Int() % 100)
  + time.AfterFunc(time.Duration(ms)*time.Millisecond, func() { req.replyCh <- replyMsg{false, nil} })
  + 就是说延迟一定时间之后，直接向请求的返回结果 channel 中加入失败结果 replyMsg{false, nil}
- 回头看正常情况
  + if reliable == false，则 time.Sleep 休眠 (rand.Int() % 27) ms
  + if reliable == false && (rand.Int()%1000) < 100，直接返回失败结果 req.replyCh <- replyMsg{false, nil}
  + 创建协程处理请求，得到结果后放到一个新建的 channel 中
```
ech := make(chan replyMsg)
go func() {
	r := server.dispatch(req)
	ech <- r
}()
```
  + 主线程等待 ech 这个新建 channel 的结果，并同时检查是否服务已经挂掉
```
		replyOK := false
		serverDead := false
		for replyOK == false && serverDead == false {    // 直到收到结果或者服务挂掉，就会推出循环
			select {
			case reply = <-ech:                          // 收到请求调用的结果了
				replyOK = true                           // 设置 flag 变量，下一步会退出循环
			case <-time.After(100 * time.Millisecond):
				serverDead = rn.isServerDead(req.endname, servername, server)   // 检查服务是否挂掉
				if serverDead {            // 如果挂掉了，会启动协程
					go func() {            // 协程的目的就是收一下 ech 中的结果，避免干活儿的协程死锁住
						<-ech // drain channel to let the goroutine created earlier terminate
					}()
				}
			}
		}
```
  + 跳出了循环，仍然再一次检查是否服务挂掉 serverDead = rn.isServerDead(req.endname, servername, server)
  + 最后，根据变量和结果的情况，进行最终返回处理
    * if replyOK == false || serverDead == true： req.replyCh <- replyMsg{false, nil}
	* if reliable == false && (rand.Int()%1000) < 100：req.replyCh <- replyMsg{false, nil}
	* if longreordering == true && rand.Intn(900) < 600：
	  ms := 200 + rand.Intn(1+rand.Intn(2000))
	  time.AfterFunc(time.Duration(ms)*time.Millisecond, func() {
			atomic.AddInt64(&rn.bytes, int64(len(reply.reply)))    // rn.bytes 中把回复的长度也加上
			req.replyCh <- reply
	  })
    * 其他情况，直接 atomic.AddInt64(&rn.bytes, int64(len(reply.reply))) & req.replyCh <- reply
  

### 最后，串起来整个的调用顺序和控制流

- 一个 Network 实例和多个 ClientEnd 实例共享两条 channels
  + Network.endCh == ClientEnd.ch，用于 ClientEnd 发送请求给 Network
  + Network.done == ClientEnd.done，用于关闭 Network，并结束 Network 的处理协程和 ClientEnd 的阻塞等待

流程，主要是正常的流程：
1. Network 的处理主协程一直在等待 endCh 和 done，一旦接收到 done 的数据，则主协程结束
2. ClientEnd 构建 reqMsg 并发送到 ch 中，然后同时等待 reqMsg.replyCh 和 done 
3. Network 的处理主协程从 endCh 中取出请求，直接开启新的协程调用 processReq 函数处理该请求
   3a) 根据 reqMsg.endname 取出对应的 servername/server/enabled ... 等等数据
   3b) 创建一个新的 channel ech，启动新的协程，协程中调用 server.dispatch 处理请求并把结果发送到 ech 中
   3c) processReq 的协程阻塞等待 ech 中的结果，同时还会用 select 的方式检查 isServerDead
       这就是为什么没有直接调用 server.dispatch 而是采用 ech channel 等待结果的方式的原因
   3d) server 通过 reqMsg.svcMeth 字段抽取出 serviceName 和 method，进而调用 service.dispatch 得到处理结果
   3e) 最终，processReq 的协程把请求处理的结果发送到 reqMsg.replyCh 中，完整结束本次 ClientEnd 的请求


  
  
