# smudge架构分析
## 启动流程
1. 解析flag,获取如下flag值：  
	1. `node`: node address  
	格式为ip:port（port可选），node flag信息会覆盖下面的port flag
	1. `port`: listen port  
	从`port` flag、环境变量`SMUDGE_LISTEN_PORT`读取，如都没有则采用默认值9999
	1. `hbf`: 周期性ping集群中其他node时，两次ping之间的间隔，单位毫秒  
	从`hbf` flag，环境变量`SMUDGE_HEARTBEAT_MILLIS`读取读取，如都没有则采用默认值500毫秒
1. 获取首个非loopback ip地址，优先选择ipv6，不存在ipv6地址则选择ipv4
1. 将前面读到的listen port, heart beat frequency, 和ip地址设置到`properties.go`中定义的全局变量中
1. 如果前面读到的node address不为空，则根据node address创建node，表示当前节点是否要参加gossip？下面必然会创建一个`thisHost`所以这里的node为可选  
这里仅创建node结构体对象，设置其ip、port、timestamp（置为当前时间）、pingmillis。node的`status`初始化为默认值`StatusUnknown`
1. 如果创建node过程中没有错误，则调用`smudge`包的`AddNode`方法:  
	1. 如果node状态为`StatusUnknown`则将其设置为`StatusAlive`。这里除了更新status之外，还要   
		1. 更新`timestamp`、`statusSource`、`emitCounter`
		1. 将node加入到`updatedNodes`，它是定义在registry.go中类似`knownNodes`的map，表示最近更新过的node, 包括living和dead的node
		1. 如果node状态不为`StatusDead`则要将node从`deadNodeRetries`中删除
	1. 如果node状态为`StatusForwardTo`则直接panic
	1. 更新node的`timestamp`为当前时间
	1. 将node对象加入到`knownNodes`里  
	`knownNodes`是定义在registry.go中的全局变量，是一个带读写同步锁的map: `map[string]*Node`,表示所有已知node，包括living和dead。
	1. 将node状态更新的事件通知所有status listener（调用`StatusListener`的`OnChange`函数）
1. 调用smudge包里的`Begin`函数，完成本节点初始化，启动主要流程处理逻辑  
	1. 初始化本节点host环境  
		1. 配置全局变量`thisHost`，将之初始化为如下node对象：  
			1. ip：从`properties.go`中读取之前保存的ip，如果没有则设置为`SMUDGE_LISTEN_IP`，再没有就设置为127.0.0.1
			1. port：和ip类似逻辑，默认值9999
			1. timestamp为当前时间，单位毫秒
			1. pingMillis为`PingNoData`（int类型const）
		1. 配置全局变量`thisHostAddress`为`thisHost`的ip:port
		1. 调用`updateNodeStatus`将`thisHost`的状态设置为`StatusAlive`，`heartbeat`为0，消息源（statusSource）设置为`thisHost`，`emitCounter`为`lambda * log(node count)`（可见后续本节点的情况也会通过ping发出去），并加入`updatedNodes`
		1. 调用`AddNode`将`thisHost`也加入进去
	1. 加入集群中其他已知节点  
	smudge可以通过环境变量`SMUDGE_INITIAL_HOSTS`将集群中已知的host告诉当前节点。这里用`AddNode`函数将所有initial host入到`updatedNodes`和`knownNodes`中。
	1. 启动主要流程理逻辑，内容将在下节描述

## 主要概念

### host和node
在registry中的`knownNodes`和`updatedNodes`管理的都是node，所以说member管理管的是node。

而在smudge中，使用`SMUDGE_INITIAL_HOSTS`可以指定集群中已有member，所以可以说host是node的同义词。支持这种说法的另外一个证据是`thisHost`在前面被初始化为一个node

### node的定义
```go
// node.go
type Node struct {
	ip           net.IP
	port         uint16
	timestamp    uint32
	address      string
	pingMillis   int
	status       NodeStatus
	emitCounter  int8
	heartbeat    uint32
	statusSource *Node
}
```
说明如下：
1. node的状态（status）定义在`nodeStatus.go`中，包括：
	* Unknown: the default node status of newly-created nodes.
	* Alive: a node is alive and healthy
	* Suspected: a node is suspected of being dead
	* Dead: a node is dead and no longer healthy
	* ForwardTo: a pseudo status used by message to indicate the target of a ping request
1. emitCounter: 当node状态发生变化时，需要通知几个其他node，通知被附加在ping息中。被初始化为lambda * log(node count)，其中lamda是常量2.5，node count是`knownNodes`的大小。
1. timestamp: 最近收到来自该节点的时间
1. heartbeat  
1. statusSource：node状态的信息来源

### 消息格式
smudge使用下面的结构体表示消息
```go
//message.go
type message struct {
	sender          *Node
	senderHeartbeat uint32
	verb            messageVerb
	members         []*messageMember
	broadcast       *Broadcast
}
```
解释如下：
1. `verb`: 表示消息类型，类型可以为:  
	1. `verbPing`: 普通的ping消息
	1. `verbAck`： 作为对ping消息的正常响应
	1. `verbPingRequest`：当之前的ping消息没有受到ack时，发pingreq给其他node，要求其代为转发ping
	1. `verbNonForwardingPing`: 代为转发的ping，如果该消息没有收到ack，则不会再次发起pingreq。smudge中的节点执行操作时没有上下文，不知道某个ping如果没有收到ack是否需要转发，只有单独列出这样的ping类型才能知道
1. `members`: 消息携带的member状态。member状态包括信息来源和node的heartbeat、status。每条消息最多可以携带2^6 - 1 = 63个（含）member的状态信息。关于这个magic number解释在message.go的`addMember`方法中
1. `broadcast`: 消息携带的广播信息,每条消息仅可携带一个广播消息

## 主要流程处理逻辑
`membership`包的`Begin`函数除了初始化本地节点，还启动了smudge的各主要流程处理逻辑，分析如下：

### 侦听UDP端口，处理接收到的UDP消息
启动goroutine,执行`listenUDP`函数。该方法除了执行`net.ListenUDP`来监听启动时获得的UDP端口值之外，还执行死循环从`UDPConn`读取消息。每收到一条消息就启动goroutine执行`receiveMessageUDP(addr, buf)`。其中`addr`类型为`net`包下的`UDPAddr`，是消息发送方的udp地址（封装了一个叫`IP`的byte slice存储ipv4或ipv6地址及端口），`buf`是消息内容。`receiveMessageUDP`处理流程如下：
1. decode UDP消息为`message`对象
1. 更新本地heartbeat todo这个做什么的？如果把heartbeat当成一种内部的transaction id,为什么要更新？一直++不好吗？
1. 更新本地关于消息发送者和消息中携带的member的状态
1. 如果消息包含广播消息，则处理广播消息
1. 根据消息类型（`message.verb`），做后续处理。这里处理的消息类型包括 `verbPing`、 `verbAck`、 `verbPingRequest`、 `verbNonForwardingPing`，具体处理逻辑见后续各小节。

#### `verbPing`消息的处理

调用`transmitVerbGenericUDP`返回ack。注意这里发UDP消息用的`code`或者说发送出去的消息的`senderHeartbeat`就是ping消息中带的`senderHeartbeat`。所以`senderHeartbeat`不可以直观理解为消息发送者在发送消息时的`currentHeartBeat`，可以将heartbeat认为是一种transaction id，比如作为ping的发送者，当你收到ack时，通过ack消息中带的`senderHeartbeat`可以知道这个ack是相应之前发出的哪个ping。heartbeat并不是gossip协议的一部分，只是内部的机制，用来把收到的消息和发出的消息对应起来。

#### `verbAck`消息的处理

根据sender的address拼接消息中包含的`senderHeartbeat`作为key，寻找`pendingAcks`中是否有之前发出的ping消息与之对应。如果找到了，则：
1. 更新本地关于sender的`timestamp`为当前时间。
1. 如果收到的ack为对之前发出的pingreq消息的响应，发ack消息给pingreq消息的发送者。
1. 最后步骤是统计ping的响应时间（当前时间减去ping消息发出的时间），低于最小ping响应时间（这个时间可以用`SMUDGE_MIN_PING_TIME`环境变量指定，默认为150毫秒）的响应时间会被算作最小ping响应时间（这个设计的作用在于prevents the system instability and flapping that can come from consistently small values）。
1. 完成之后将之前的`pendingAck`对象从`pendingAcks`表中删除。

#### `verbPingRequest`消息的处理

#### `verbNonForwardingPing`消息的处理

### 处理ping超时
在单独goroutine里执行`startTimeoutCheckloop`函数，处理3类ping消息的超时

### 周期性ping群中所有node（不包含自身）并在ping的同时附带传播最近更新的node状态和广播
这个逻辑直接写在`Begin`函数内，在一个死循环内执行个个“ping周期”。每个ping周期挑选一部分已知节点ping一轮，逻辑如下：
1. 生成不包含本地node的所有已知node列表（dead & living），node列表通过range map得到，本身每次执行都能得到随机顺序，并再次随机交换node顺序到更为随机的列表
1. ping列表中的每个node：  
如果目标node状态为`StatusDead`，每次ping之后都要跳过后续连续的若干次ping周期，跳过的次数为`retryCountdown`，计算方法为2的retry次方，所以是一种指数退避。一旦retry次数到达硬编码的`maxDeadNodeRetries`（10）次，就将该node从`deadNodeRetries` map里删除，并从`knownNodes`中删除。对于非dead node,则直接ping  
ping某个node的流程为：  
	1. `currentHeartBeat`++
	1. `PingNode`: 以对方node的address拼接`currentHeartBeat`构成的string为key,在`pendingAcks`里存入`pendingAck`对象，表示正在进行的一个ping某个node的任务，后续可能需要处理其超时问题。然后执行`transmitVerbGenericUDP`。该函数发出的ping消息中会带上`lambda * log(node count)`个本地最近有状态更新的成员（即`emitCounter`大于零的成员）。这些成员从`updatedNodes`里按照`emitCounter`从高到低挑出来，不包含本地node和目的地node。每次某个node的状态作为ping的加信息的发出去之后，该node的`emitCounter`就会减一，一旦`emitCounter`变为0,就会从`updatedNodes`中删除。因此某个node状态被更新的事件只会通知`emitCounter`个ping对象，而不是`emitCounter`个ping周期。万一无法从`updatedNodes`中找到符合条件的node,就从`knownNodes`中找相同数量的node作为ping的附加成员状态更新。可见状态不变的node的信息还是有机会同步到别的node中，只是机会少一些。
	1. sleep由`hbf` flag或环境变量指定的毫秒数

如果上述ping的过程中修改了`knownNodes`，则打断当前ping周期，重新生成ping的对象列表，进入下一个ping周期。如果整个ping周期内都没有ping任何node（`pingCounter`=0）,则sleep由`hbf` flag或环境变量指定的毫秒数

### 本地member状态更新

todo 包含那些状态？

### 广播消息处理

todo
