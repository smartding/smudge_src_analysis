# smudge架构分析
## 启动流程
1. 解析flag,获取如下flag值：  
	1. `node`: node address  
	格式为ip:port（port可选），node flag信息会覆盖下面的port flag
	1. `port`: listen port  
	从`port` flag、环境变量`SMUDGE_LISTEN_PORT`读取，如都没有则采用默认值9999
	1. `hbf`: heart beat frequency (milli seconds)  
	从`hfb` flag，环境变量`SMUDGE_HEARTBEAT_MILLIS`读取读取，如都没有则采用默认值500毫秒
1. 获取首个非loopback ip地址，优先选择ipv6，不存在ipv6地址的情况下选择ipv4
1. 将前面读到的listen port, heart beat frequency, 和ip地址设置到一些全局变量中(properties.go)
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
1. 调用smudge包里的`Begin`函数，监听UDP端口，开始heartbeat  
	1. 初始化全局host环境  
		1. 配置全局变量`thisHost`，将之初始化为如下node对象：  
			1. ip：如果之前ip没有获得，则设置为`SMUDGE_LISTEN_IP`者认值（127.0.0.1）
			1. port：如果之前的port没有得到，则设置为`SMUDGE_LISTEN_PORT`或默认值（9999）
			1. timestamp为当前时间
			1. pingMillis为`PingNoData`
		1. 配置全局变量`thisHostAddress`为`thisHost`的ip:port
	1. 将`thisHost`的状态设置为`StatusAlive`，`heartbeat`为0
	1. 调用`AddNode`将thisHost也加入进去
	1. 启动goroutine,执行`listenUDP`函数。该方法除了执行`net.ListenUDP`之外，还执行死循环从`UDPConn`读取消息，每次收到一条消息就启动goroutine执行`receiveMessageUDP(addr, buf)`。其中`addr`是方的udp地址，buf是消息
	1. 用`AddNode`函数将所有initial host入到`updatedNodes`和`knownNodes`中。可以通过环境变量`SMUDGE_INITIAL_HOSTS`为当前主机指定集群中已知的host。所以在smudge中的member的粒度是host
## 主要逻辑
### UDP消息处理
