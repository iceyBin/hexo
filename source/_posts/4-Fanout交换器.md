---
title: 4，Fanout交换器
date: 2019-05-27 17:41:47
tags: RabbitMQ
categories: 中间件

---
# 1 知识回顾

通过之前 `direct` 交换器的学习，我们已经对rabbitMQ消息传递机制有了一定的了解，可以总结为以下几点：

1. 生产者是生产并发送消息的应用程序
2. 队列是消息的缓冲区，用来存放消息
3. 消费者是接收并消费消息的应用程序

我们知道，broker是生产者和消费者之前的"桥梁"，接收生产者消息并将消息发送给消费者。那么，应该发给那些消费者，是指定的消费者还是全部的消费者呢？这就取决于 `exchange` 的类型，一共四种类型：`direct`，`headers`，`fanout`，`topic`，其中 `headers` 跟 `direct` 很类似，基本不再使用。 这里我们将会说到 `fanout` 交换器

# 2 应用场景

生产者发送的消息到broker，所有绑定到改broker的队列对应的消费者都可以接收到消息，类似于"广播"通知模式。<br>
例如注册操作。 一般情况下，用户注册后都会进行一系列的业务操作，比如站内信通知，发送积分，甚至活动期间发送奖励等。这时，就需要当用户注册后，所有关注 "注册" 的业务都需要收到 "用户注册" 这个消息，并进行相应的逻辑处理，这时就可以用 `fanout` 类型交换器实现消息的推送通知。

# 3 进入正题

![fanout交换器](/../img/201905/fanout.png)

## 3.1 生产者发送消息

	// 定义Fanout类型的交换器
	channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);

	// 发送消息
    String message = "hello world";
    channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes("UTF-8"));
    System.out.println(String.format("推送消息[%s]成功", message));

我们可以看到 `basicPublish` 方法的第二个参数 `routinhKey` 为 `""`。这里无论是否使用空字符串，效果都是一样的，因此直接使用 `""` 即可

## 3.2 消费者接收消息

	// 创建交换器
	channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);
	
	// 创建一个私有的临时的队列 exclusive=true auto-delete=true durable=false
	String queueName = channel.queueDeclare().getQueue();
	
	// 绑定队列到交换器
	channel.queueBind(queueName, EXCHANGE_NAME, "");

	// 消费消息
    channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {});

当然，你完全可以像之前那样定义一个持久性有固定名称的队列。但是，如果有需要 *（比如每次连接到broker都需要一个全新的队列）*，这时就可以使用方法 `channel.queueDeclare()` 创建一个随机的队列，该队列性质：`exclusive=true`，`auto-delete=true`，`durable=false`<br>

这里还要说下 `queueBind()` 方法，可以看到该方法的第三个参数 `routingKey` 也是 `""`。 就像我们之前说的那样，这里也是一样，无论是否使用空字符串效果都是一样的，因为 `fanout` 类型的交换器会把到达broker的消息发送给所有绑定到该交换器上的 `queue` 中 *(无论使用的routingKey是否是"")*