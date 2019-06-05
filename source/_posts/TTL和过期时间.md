---
title: TTL和过期时间
date: 2019-06-04 10:37:41
tags: RabbitMQ
categories: 中间件

---

# 1 TTL介绍

TTL全称Time-To-Live，即存活时间。在RabbitMQ中有两种TTL，分别是 `Message TTL` 和 `Queue TTL`。TTL可以通过参数设置，也可以通过 `policies` 策略设置 `message-ttl` 参数，推荐使用 `policies` 可以通过正则表达式为一个或者多个 `queue` 或者 `exchange` 定义TTL

# 2 Message TTL -- TTL类型消息

如果一条消息在队列中的存放时间超过了设置的TTL超时时间 *（即在TTL设置的时间内没有消费者消费该消息）*，则该消息就会被认为已经"死了"，broker不会将"死了"的消息再推送给消费者，并且将会移除这些消息。消息超时时间必须是一个**非负整数(>=0)**，单位是：**毫秒**

定义消息的TTL分为两种方式： **在队列中定义**以及**在消息中定义**

> 在队列中定义，设置的TTL值只对当前队列的消费者有效

通过 `channel.queueDeclare` 方法，其最后一个参数 `arguments` 中添加 `x-message-ttl` 字段，其值设置为具体的超时时间即可。例如，设置消息超时时间为3s，则部分代码如下：

	Map<String, Object> arguments = new HashMap<>();
    arguments.put("x-message-ttl", 3000);
    channel.queueDeclare(queueName, true, false, false, arguments);

使用 `policies` 设置时，会选择 `apply-to=queues`，因此也属于该种方式

> 在消息中定义，设置的TTL值对该消息的所有的消费者都有效

也可以通过生产者调用 `basic.publish` 方法，设置 `AMQP.BasicProperties` 参数的 `expiration` 属性。例如：

	AMQP.BasicProperties basicProperties = new AMQP.BasicProperties()
                    .builder()
                    .expiration("3000")
                    .build();
    channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY, basicProperties, message.getBytes());

> **使用TTL时，你可能有些地方需要注意**

- 发送设置了TTL的消息到队列中时，如果此时队列中已经有了其他设置了TTL的消息，有可能会出现队列中已经过期的消息在未过期消息的后面。这种情况下该已过期的消息不会被移除，只有当其前面的消息过期或者被消费者消费掉，该过期消息到达队列头部时才会被丢弃*(或者放入"死信"队列)*。并且，该已过期的消息在未被移除之前，其占用的系统资源并不会释放，该消息仍然会被计入到一些统计数据中*(eg: 队列中总消息数目)*。
- 设置TTL的消息会存在一种情况：已经被broker写入socket向消费者传送，但是在到达消费者之前，过期了（这种情况下，笔者也不知消息将会怎样？）
- 当使用 `policy` 设置TTL时，建议有消费者实时在线消费消息，以确保过期的消息能够尽快的被丢弃
- 考虑到上述行为，当需要删除消息释放资源的时候，设置了TTL的队列**queue TTL**应该考虑使用

# 3 Queue TTL -- TTL类型队列

TTL不仅能设置到队列中的消息上，还能设置到队列上。设置到队列上表示的意义是：**在该队列自动删除之前还能存留的时间，单位是：毫秒，必须是正整数(>0)**, ，设置了TTL的队列，当没有消费者后，等待超时时间后就会被删除，跟队列的 `auto-delete` 属性无关。

> 使用队列属性参数定义。

如下，定义一个TTL为5s的队列：

	Map<String, Object> arguments = new HashMap<>();
    arguments.put("x-expires", 5000); // 设置队列过期时间
    channel.queueDeclare(QUEUE_NAME, true, false, false, arguments);

当该队列没有消费者后，等待**5s**就会被删除，即使 `autoDelte` 参数为false

> 使用 `policy` 定义

设置 `policy` 策略的 `expires` 参数，可以达到和 `x-expires` 一样的效果，更推荐使用 `policy` 定义。如：

	rabbitmqctl set_policy expiry ".*" '{"expires":5000}' --apply-to queues 

语法格式：rabbitmqctl set_policy <policy名称> <JSON格式参数> --apply-to [queues|exchanges]。除了使用命令行，也可以在WEB UI上直接定义设置 `policy`




