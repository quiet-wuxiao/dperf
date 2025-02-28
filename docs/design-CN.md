# dperf设计原理

## 背景
dperf最早设计来给四层负载均衡器(L4LB)做性能测试。目标是用一个通用物理服务器，装上一些合适的网卡，dperf就可以产生相当大的压力，既方便研发人员开展测试，也方便在客户环境演示我们L4LB的性能。

L4LB的性能通常很高。主流L4LB是基于IP报文转发实现的，通过会话表来跟踪连接状态，然后修改、转发IP报文; 并且L4LB不分析应用层数据； 所以L4LB可以达到数百万的每秒新建连接数，几十亿的并发，数百万Gbps的吞吐。

L4LB的性能高，高到对它进行性能测试是一个非常难事情。本文说明dperf是如何解决这个问题的。由于UDP比较简单，本文只从TCP的角度去说明。

## 核心需求
- 极高性能的TCP协议栈, 各方面的性能要超过L4LB；
- 支持HTTP协议，方便利用各种工具对协议栈的兼容性、正确性进行测试;
- 能够在一台物理服务器上模拟出HTTP客户端与HTTP服务器，用最少的资源，完成测试。

## 一些合理的假设
- 压力测试在内部网络进行，IP地址可整段使用;
- dperf可以接入三层交换机，无需支持路由功能;
- 可以测试小包、大包即可，无需支持长HTTP消息（如16K的HTTP请求, 1MB的HTTP响应等), HTTP消息长度限定在1个MTU内;
- 不追求单个连接的传输速度，通过大并发、长连接等机制来增强整体测试压力;
- 在性能测试中，HTTP的报文格式是合法的，dperf可以不对HTTP报文做合法性做完整的校验;
- 测试过程中dperf发出的HTTP消息内容长度固定、内容不变。

## 设计要点
### 多线程与网卡分流（FDIR）
dperf是一个多线程程序，每个线程只使用1个RX和1个TX。
dperf使用网卡的分流特性（FDIR)。
每个dperf server线程绑定一个监听IP，dperf server上使用报文的目的IP分流。
每个dperf client线程只请求一个目的IP，dperf client上使用报文的源IP分流。
经过分流后，线程之间没有共享资源，没有锁竞争，理论上性能可以线性扩展，实际上会因为共享CPU执行单元、Cache、总线、网卡等物理资源，难免会相互干扰, 做不到百分百线性。

### 极小socket
1个socket（1条连接）只占用64字节，可以放在一个cache line中，10亿连接只需要6.2GB内存，所以dperf很容易达到几十亿并发连接数。

而通用TCP协议栈一条连接大概会占用16K资源，四层负载均衡的一条会话记录的大小在几百个字节, 所以用dperf去压测四层负载均衡的并发连接数是绰绰有余的。

### 地址与socket查找
dperf要求使用连续的客户端地址、连续的监听地址、连续的监听端口。
dperf使用到的源地址、目的地址必须全部写在配置文件中，dperf在启动时根据地址、端口的组合，把socket一次性申请好，组成一个3维数组查找表。
为了减少内存消耗，客户端地址使用最低的两个字节来索引，服务器地址不用索引，每个线程关联一个服务器IP，监听端口用序号索引。
dperf不用哈希算法查找socket，只需要访问数组，花费几条指令而已。

### 定制的TCP协议栈
我原计划使用FreeBSD协议栈, 但由于一些原因，我需要在2周之完成Demo。于是我需要降低工作量与难度，不得不放弃FreeBSD协议栈，需要重新设计一个TCP协议栈，那么这个问题就变成：对四层负载均衡器的压力测试仪来说，什么样的TCP协议栈是可以接收的？

这是我思考的结果：
- 协议栈只用来做四层负载均衡器的性能测试;
- 不需要支持拥塞控制、SACK；只需实现停等协议，超时重传。客户端发送1个请求报文，等待服务器响应的报文，响应仍然是1个报文。如果超时到了，还没收到对方的确认，就重传, 多次重传不成功就认为连接失败;
- 不需要支持事件机制，收到报文后立刻进行应用层处理，socekt与应用层处理深度融合进TCP协议栈;
- 不需要支持TCP keepalive定时器，不存在很长时间的空闲连接，短连接不需要TCP keepalive定时器，长连接会定时发送请求，也不需要TCP keepalive定时器;
- 不需要timewait定时器。已经关闭的连接允许快速回收使用;
- 由于dperf一次只发送1个报文，接收1个报文，我们认为dperf的发送窗口、接收窗口是足够的，不会发生、也不支持报文被部分确认的情况。

### 定时器
dperf曾经使用了通用的时间轮定时器，一个时间轮定时器消耗32字节，一个socket使用了两个定时器，消耗了64个字节，这对支持几十亿并发连接数的测试仪来说内存消耗有点高，另外频繁的函数回调开销也不可接受。
我去掉了时间轮定时器，借鉴了FreeBSD的timewait定时器，把所有相同的任务放在一个按时间排序的链表中。

### 零拷贝，无接收/发送缓存，零校验和
- 零拷贝。除了启动阶段，dperf不拷贝任何数据, 包括以太网头部/IP头部/传输层头部/payload，因为这些内容几乎是固定不变的；
- 无接收缓存。dperf不关心payload内容，这些内容只不过是报文的填充物而已, dper只关心收到了多少字节；
- 无发送缓存。普通协议栈需要发送缓存，只有收到对方的确认，才能释放这些缓存，否则重传；另外还需要考虑到对方的确认的序列号可能是在缓冲区的任意位置，不一定是确认一个报文的结尾；发送缓存的管理是一个比较复杂的工作。发送的数据都是一样的，所以dperf不需要socket级别的发送缓存，有一个全局的报文池就可以了；
- 零校验和。通常我们会利用网卡功，卸载报文的校验和，但是我们要计算伪头部; 对同一个连接来说，报文类型是固定，dperf已经算好了尾部头校验和，整个过程没有校验和计算。

### HTTP协议实现
dperf服务器非常傻，它收到任何一个数据包（第一个字符是G, GET的开始），就认为是完整的请求，就把固定的响应发送出去。

dperf客户端也非常傻，它收到任何一个数据包，如果第10个字符是'2'（假设来自"HTTP/1.1 200 OK")，就认为是成功的响应。

### 其他优化
- dperf大量使用inline来避免函数调用;
- socket的内存从大页分配，避免页表缺失;
- 报文批量释放;
- 发送带缓存，一次发送不成功，下次再发送，不会立刻丢包。

## 其他用途
根据dperf的特点，我们发现它除了可以对L4LB进行测试外，还可：
- 对其他基于四层转发的网关进行测试，如链路负载均衡、防火墙等;
- 对云上虚拟机的网络性能进行测试;
- 对网卡性能、CPU的网络报文处理能力进行测试;
- 作为高性能的HTTP客户端或HTTP服务器用于压测场景。
