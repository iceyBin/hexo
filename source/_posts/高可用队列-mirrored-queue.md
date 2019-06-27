---
title: 高可用队列 - mirrored queue
date: 2019-06-11 11:38:26
tags: RabbitMQ
categories: 中间件

---

# 1 queue mirror 介绍

默认情况下，在RabbitMQ集群中，所有的信息，状态都会每个节点之间进行复制。但是队列例外，尽管在每个节点都可以看到并且可以访问所有队列，但是每个队列只会将其内容存储一个节点(定义队列的节点)上，如果想要队列内容存储也实现集群，则可以通过设置为mirrored queue实现跨多个节点。 每个mirrored queue都是由一个 ***master*** 节点和一到多个 ***mirror*** 节点， 每个 mirrored queue 都有自己的 ***master*** 节点，该队列的所有的操作都是在 ***master*** 节点上完成的 (*包括生产者推送消息、分发消息到队列、消费者 `ack` 确认等操作*)， ***master*** 节点之后再广播通知所有的 ***mirror*** 节点。消费者总是会连接到其对应 mirrored queue 的 ***master*** 节点上，即使连接的是 ***mirror*** 节点也会被转接到 ***master*** 节点，消费者 `ack` 确认后会通知 ***master*** 节点，然后会通知所有 ***mirror*** 节点，***mirror*** 节点收到通知后会将已经`ack`的消息从队列删。mirrored queue保证了队列的高可用性

# 2 如何设置 mirrored queue

通过设置队列的参数可以保证该队列是 mirrored queue，这些队列参数只能通过 `Policies` 来控制。普通队列可以随时变为 mirrored queue，反之亦然，只需要修改 `Policies` 即可。普通队列和有镜像的队列的不同之处在于：前者缺乏相关的参数，会有较高的吞吐量

> 具体的参数设置

设置 mirrored queue需要两个关键的参数 `ha-mode` 和 `ha-param`，其中 `ha-param` 是可选配置，这两个参数是相互呼应的。`ha-mode` 参数有三个值：`exactly`、`all`、`nodes`，且 `ha-params` 参数具体值为： *count* 和 *node names*

1. 当 `ha-mode` 值为 `exactly` 时，`ha-params` 参数值是 *count* 也就是具体的数字，表示在队列集群中所有节点的数目(master +  mirrors)

	- 如果 *count* 值为1，说明集群中只有一个 ***master*** 节点，如果该节点故障，将会出现不可预测的结果 
	- 如果 *count* 值为2，说明集群中有两个节点，一个 ***master***，一个 ***mirrors***，如果 ***master***故障，则 ***mirrors*** 节点会立刻取代成为master
	- 如果集群中的节点数目比 *count* 的值小，则会使用全部的节点
	- 如果集群中的节点比 *count* 的值大，则如果此时集群中的一个节点下线了，则会立刻加入新的节点进来

2. 当 `ha-mode` 值为 `all` 时，此时无需设置 `ha-params` 参数。表示 mirrired queue 会将所有节点都会用到集群中，如果新增了一个节点也会被添加到集群中。这种方式是比较老旧的方式，会对网络I/O、磁盘I/O以及磁盘空间的利用率带来额外的负担，不应该将所有的节点都纳入集群中，官方推荐的集群中的节点数是 *N/2 + 1*

3. 当 `ha-mode` 值为 `nodes` 时， `ha-params` 参数设置为 *nodes names*，具体的值是 `rabbitmqctl cluster_status` 命令查询出来的nodes的值，形如 *rabbit@hostname*

> 集群中多少节点比较合适

将所有的节点都应用到队列集群中，会对网络I/O、磁盘I/O以及磁盘空间的利用率带来额外的负担。如果节点数量是3个或3个以上，则推荐使用公司 *(N+1)/2* 计算需要放入集群中的节点数目。如果一些数据是瞬时态的或者对时间要求比较高，可以对这些队列降低节点数量，或者甚至不适用 mirrored queue

> 如何查看一个队列是否是mirrored queue

可以通过 **WEB UI** 页面查看相应的队列是否是存在集群，存在集群的队列形式如下：

![mirrored_queue](/../img/201905/mirrored_queue.png)

# 3 mirrored queue的 master、master节点迁移、数据存放


> 集群中的master节点

集群中 ***master*** 节点的选举有是三种策略控制方式：

1. 使用队列参数 `x-queue-master-locator`
2. 通过 ***policy*** 设置 `queue-master-locator`
3. 在配置文件中设置 `queue_master_locator`

上述三种配置方式可以达到相同的效果，配置的值是一样的，有以下几种情况：

- 设置值为 `min-masters`：选举节点中负载最小的为 ***master***
- 设置值为 `client-local`:选举客户端定义队列时指定的节点为 ***master***
- 设置值为 `random`：随机挑选一个节点

> **"nodes" 类型的mirrored queue**

如果一个mirrored queue的 `ha-mode`设置为 `nodes`，则当更新该队列对应的 *policy* 时，如果当前的 ***master*** 节点没有在修改后的 `ha-params`指定的主机中，则该 ***master*** 节点会在当前集群中丢失。为了防止数据的丢失，RabbitMQ会保持 ***master*** 节点在线，直到所有的 ***mirrors*** 节点同步完成，该节点才会消失

> 独立队列 *`exclusive=true`* 集群

如果一个队列是独立性质的，即 `exclusive=true`，则该队列不能设置为 mirrored queue

> 普通队列集群

如果队列集群中的 ***master*** 节点正常运行，那么对于该队列的所有的操作都可以映射到集群中的每个节点上，该队列的操作仍然只会被路由到主节点。如果集群中的 ***master*** 节点故障了，那么该队列是否是持久化队列表现不同。**非持久化队列直接删除**，**持久化队列的所有的操作都不能继续进行**，直到节点恢复正常。这个时候在RabbitMQ的日志文件中可以看到相关报错信息，大致像这样：
	
	operation queue.declare caused a channel exception not_found: home node 'rabbit@hostname' of durable queue 'queue-name' in vhost

