---
layout: post
title:  "NSQ消息队列的几个特性与场景理解"
date:   2016-06-28 12:32:28 +0800
categories: my article
---

NSQ的[官方描述文档](http://nsq.io/overview/features_and_guarantees.html)中明确的声明了NSQ的几个特性：

### 1. 消息默认是不持久化的
nsq主要还是一个内存存储的消息队列，本身消息的缓冲区是在内存中的，只有超过一定的值之后才会持久化硬盘。如果希望每一条消息都持久化，可以通过--mem-queue-size=0参数来实现。

### 2. 没有内置的副本机制（built in replication）
如果想要保证HA，需要使用者自己考虑部署的拓扑结构，可以采用的技术方式没有限制，比如设置一个slave节点，或者把mem-queue-size设置为0保证消息不丢失等。

不要把支持集群与副本机制混为一谈，集群里的每个机器之间不一定是有主备关系的。

### 3. 消息的送达保证语义是至少送达一次（at least once），但前提是nsqd没有宕机

这里要两点要注意：

- 消费者要保证执行的逻辑是幂等的。 
- 这个与上面第二点有关，因为没有内置的备份机制，所以如果使用者既没有设计合理的拓扑结构保证nsqd的高可用性，又没有设置把所有的消息保存到硬盘上，那么万一nsqd挂了，那就是真悲剧了，消息会丢。

### 4. 消息的送达是没有时序保证的
因为nsqd之间是没有消息共享的，所以想保证集群全局范围内的时序性是不可能的。但是有一个很简单的策略可以给其加上一个弱时序的约束（虽然不能保证全局时序，但是能保证此消费者接受的时序）：

```
在消费者处理消息前，先经过一个『时延窗口』的排序，在这个窗口里对接收到的消息进行一个排序，然后再
按顺序交给消费者处理。当然，对于在时间窗口外的消息，是要做丢弃处理的。
```

----

### 总结
- nsq说自己可以很好的避免的单点故障，我认为指的是单点出问题了之后，后续整体业务功能不受影响，但是对之前的业务（尤其是出问题的单点上的丢失的消息）还是会有一定的影响的。所以一定要靠自己做好HA拓扑，比如官网上就很明确的有这样一段关于[消息可靠性的描述](http://nsq.io/overview/design.html)：

```
Message Delivery Guarantees

NSQ guarantees that a message will be delivered at least once, though duplicate 
messages are possible. Consumers should expect this and de-dupe or perform 
idempotent operations.

This ensures that the only edge case that would result in message loss is an 
unclean shutdown of an nsqd process. In that case, any messages that were in 
memory (or any buffered writes not flushed to disk) would be lost.

If preventing message loss is of the utmost importance, even this edge case can be 
mitigated. One solution is to stand up redundant nsqd pairs (on separate hosts) 
that receive copies of the same portion of messages. Because you’ve written your 
consumers to be idempotent, doing double-time on these messages has no downstream 
impact and allows the system to endure any single node failure without losing 
messages
```

也就是充分利用cosuner幂等操作的特性，做一个冗余的nsqd，每条消息都处理两遍，虽然有点费事，但是对于不允许数据丢失的场合，这样的冗余还是可以接受的。

![image](http://o9iu90isb.bkt.clouddn.com/nsqd_to_consumer.png)

## 其他问题

### 1. nsq的消息是被推送到消费者？还是由消费者自己去轮询（kafka的策略）？
	
	[引用一段官网上话](http://nsq.io/overview/design.html#efficiency)：
	
```
For the data protocol, we made a key design decision that maximizes performance 
and throughput by pushing data to the client instead of waiting for it to pull. 
This concept, which we call RDY state, is essentially a form of client-side flow.

When a client connects to nsqd and subscribes to a channel it is placed in a RDY 
state of 0. This means that no messages will be sent to the client. When a client
is ready to receive messages it sends a command that updates its RDY state to 
some # it is prepared to handle, say 100. Without any additional commands, 100 
messages will be pushed to the client as they are available (each time 
decrementing the server-side RDY count for that client).
```
	
![image](http://o9iu90isb.bkt.clouddn.com/nsq-comuser-recvmsg-process.png)
	
上流程图的步骤如下：

	1. consumer通过某种方式（nsqlookupd查询，或者直接配置nsqd地址）获取到nsqd地址后，通过tcp连接上
	2. consumer向nsqd上报自己订阅的topic
	3. consumer
	4. 接受消息
	5. 返回处理的结果是成功还是失败
	
首先明确一个前提，就是一个consumer可以同时连接多个producer（甚至说所有的producer）。producer只会向已经更行了RDY状态大于0的consumer发送消息，所以果从这个角度来看，nsq中的消息是被推送到consumer的。但是，consumer在汇报ready状态的时候，是轮询每一个nsqd的。

### 2. 既然不同的nsqd之间没有交流，如果一个topic在两个以上的nsqd中都存在，何保证一个topic中的某个消息只被一个channel接受？
这个问题本身就是错的。一个topic中的消息会被所有的channel接收。一个channel中的消息才会只被一个cousumer接收。

首先这个问题的担忧是建立在这样一个假设上的：若topic分布在多个nsqd上，那么每个producer发出的消息都应该在这些nsqd上都有一个拷贝。只有在这种情况下，才需要担心会被发送到多个channel中的情况。
但实际上，一个producer只会把一个消息发送给一个nsqd，并且这个nsqd与集群内的其他nsqd是没任何交流的，也就不存在这个消息有多个拷贝的情况。
如果某个cousumer订阅了这个topic下的某个channel，此时这个channel才会被创建。

假设，某个topic叫『跑男』被分布在两个nsqd上，分别叫做A,B，并且都有producer往里面放内容（因为Topics are created on first use by publishing to the named topic or by subscribing to a channel on the named topic.）。一个cousumer叫小强订阅了这个topic下的channel叫『浙江卫视』，此时A和B上都有『跑男』+『浙江卫视』的channel。
如果某个procuder往B发布了新一集的跑男，A上是没有的。小强在A，B上都注册过RDY > 0；所以B会推送给小强新一集的跑男消息。这就是一个完整的过程。一切工作正常的原因是A，B都有完整的channel订阅信息，谁获得到信息谁负责推送，所以不会有问题。

这时候有强迫症的同学肯定又担心了，如果某个nsqd的注册信息不完整呢？比如突然多了一个C，消息也发送到了C上，但是小强还没有通过nsqlookupd发现这个C，那消息会丢吗？当然不会，如果消息没有被任何消费者接受，这个消息是会存贮在nsqd中一直等到有人把它领走的。如果有人把消息领走了，即使不是小强也没有关系，因为channel不是广播，就是单点的，只要consumer处理掉就行了，管他是不是小强呢。

如果非要找一个临界的情况，倒是有一个，那就是当有多了一个消费者杰克，订阅了topic『跑男』下的另一个channel『爱奇艺』，刚刚到A那里注册完，但是还没有来得及到B那里注册这个topic，这时候B刚巧又收到了一条新的跑男消息，那么B会推送给订阅了『跑男』+『浙江卫视』的小强，之后这条消息就处理完毕了，杰克会丢掉这条消息。其实这也是合理的，因为这个topic下的channel信息没在整个集群中同步完，也就是没有达到一致。所以这也启发我们要注意一点

	1. 确保consumer与所有包含它订阅的topic的nsqd保持连接是极其重要的，如果不能保证会发生严重
	的消息丢失
	
	2. 如果新上线一个consumer，并且在已有的topic中订阅了一个新的channel，在它没有连接到所有
	nsqd之前的这段时间，所有发到它没有连接到的nsqd上的消息都会丢失。