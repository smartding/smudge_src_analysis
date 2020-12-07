# smudge架构分析
## 启动流程
1. 解析flag,获取如下flag值：  
	1. `node`: node address
	1. `port`: listen port  
		从`port` flag、环境变量`SMUDGE_LISTEN_PORT`读取，如都没有则采用默认值9999
	1. `hbf`: heart beat frequency (milli seconds)  
		从`hfb` flag，环境变量`SMUDGE_HEARTBEAT_MILLIS`读取读取，如都没有则采用默认值500毫秒
1. 获取首个非loopback ip地址，优先选择ipv6，不存在ipv6地址的情况下选择ipv4
1. 将前面读到的listen port, heart beat frequency, 和ip地址设置到一些全局变量中(properties.go)
1. 如果前面读到的node address不为空，则根据node address创建node（这里居然没有else，node可有可无？）
1. 调用smudge包里的`Begin`函数

## 主要模块
