---
title: 'Topic交换器'
date: 2019-06-03 11:00:11
tags: RabbitMQ
categories: 中间件

---

上次文章说到 `Fanout交换器`，有兴趣的可以参考：[Fanout交换器](https://iceybin.github.io/2019/05/27/4-Fanout%E4%BA%A4%E6%8D%A2%E5%99%A8/)

# 1 应用场景

当需要**"分类"推送消息**的时候，就会用到 `topic` 类型的交换器。例如：推送日志信息，需要将系统日志和普通日志分类推送给不同的队列；当然使用前面说到的 `direct` 类型交换器和 `fanout`交换器也可以实现，但是比较麻烦，这种场景下，`topic` 类型交换器将是你最好的选择。

# 2 Topic交换器

![topic交换器](/../img/201905/topic_exchange.png)

`topic` 交换器允许使用带有"通配符"的 `RoutingKey` 将队列和broker绑定在一起，支持的通配符有两种：`*` 和 `#`

- `*` 匹配一个单词，以 `.` 为分割标记。例如： `*.rabbit` 可以代表 `one.rabbit` 、`two.rabbit`等等
- `#` 匹配一个或者多个单词

## 2.1 生产者推送消息

	// 创建交换器：auto-delete=false  durable=false
    channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, true);

    // 发送消息
    String message = "hello topic exchange";
    String routingKey = "TOPIC.RK.001";
    channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes());

生产者推送消息到broker，可以看到使用的 `RoutingKey` 是 `TOPIC.RK.001`

## 2.2 消费者消费消息，这里定义两个消费者 `Recv1` 和 `Recv2`

**Recv1.java** 部分代码如下：

	// 创建交换器
    channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, true);

    // 创建一个私有的临时的队列 exclusive=true auto-delete=true durable=false
    String queueName = channel.queueDeclare().getQueue();

    // 绑定队列和broker
    channel.queueBind(queueName, EXCHANGE_NAME, "TOPIC.*.*");

**Recv2.java** 部分代码如下：

	// 创建交换器
    channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, true);

    // 创建一个私有的临时的队列
    String queueName = channel.queueDeclare().getQueue();

    // 绑定队列和broker
    channel.queueBind(queueName, EXCHANGE_NAME, "TOPIC.#");

以上可以看到，`Recv1.java` 中使用的 `RoutingKey` 是 `TOPIC.*.*`；而 `Recv2.java` 中使用的 `RoutingKey` 是 `TOPIC.#`。此时，当生产者程序启动向broker推送消息，则 `Recv1` 和 `Recv2` 都会接收到消息

- 如果生产者的 `RoutingKey` 换成 `TOPIC.001` 则 只有 `Recv2` 能接收到消息
- 如果生产者的 `RoutingKey` 换成 `RK.001` 则两者都不会接收到消息

# 3 总结

- `topic` 交换器支持通配符匹配，支持的通配符有 `*` 和 `#`
- 当精准匹配的时候相当于 `direcr` 交换器
- 当模糊匹配的时候相当于 `fanout` 交换器