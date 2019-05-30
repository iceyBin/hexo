---
title: 3, Direct类型交换器-生产者confirm模式
date: 2019-05-26 17:29:29
tags: RabbitMQ
categories: 中间件

---
# Direct类型交换器-生产者confirm模式

上篇文章简单入门了Direct交换器，如有需要请查看 [Direct类型交换器-基础使用](https://iceybin.github.io/2019/05/26/2-Direct%E7%B1%BB%E5%9E%8B%E4%BA%A4%E6%8D%A2%E5%99%A8-%E5%9F%BA%E7%A1%80%E4%BD%BF%E7%94%A8/#more)

## 1 prefetch属性的设置

问题：分布式环境中，每个消费者会有多个实例同时存在，这时对应的生产者推送的消息，该消费者如何消费呢？是均匀的分发，还是根据消费能力，能者多劳呢？

答案是：**均匀分发消费**。如果一个消费者有多个实例，那么到达这个消费者的消息会被broker进行均匀的分发消费，这是broker默认的机制。当然，实际应用中，会有很多的条件导致该消费者的多个实例的消费能力不同 *(比如机器配置不同、网络带宽影响等等)*，这时我们理想的情况是"能者多劳"。这时，我们只需要调用 `basicQos` 方法这时 `prefetch = 1` 即可 ，即可达到理想效果 *(这也是springBoot集成rabbitMQ的默认配置)*。这样设置的情况下，如果某个消费者实例消费能力比较弱，在收到 `ack` 确认之前，broker不会再分发消息到该实例

	int prefetchCount = 1;
	channel.basicQos(prefetchCount);
	
## 2 生产者confirm模式

上篇文章说到，消费者可以采用手动 `ack` 确认的模式尽量的防止消息的丢失。但是，这样真的就达到目的了吗？当然不是，试想，如果生产者的消息根本就没有到达broker呢？要解决这个问题，就需要用到**生产者confirm模式**

### 2.1 设置channel为confirm模式

	// 将信道设置为confirm模式
    channel.confirmSelect();

broker如果接收到消息会进行回调，生产者有两种方式可以进行消息发送成功以及失败的后续处理，分别是**手动逐条确认**和**异步监听确认**

### 2.2 手动逐条确认

每次发送消息后进行确认

	if (channel.waitForConfirms()) {
    	System.out.println(String.format("推送消息[%s]成功", message));
    } else {
        System.out.println(String.format("推送消息[%s]失败，消息丢失", message));
    }

### 2.3 异步监听确认

还可以通过添加监听进行异步确认。改方式的优点是：**异步、可以批量确认**

	channel.addConfirmListener(new ConfirmListener() {
        @Override
        public void handleAck(long deliveryTag, boolean multiple) throws IOException {
            System.out.println(String.format("MQ接收消息[%s:%b]成功", deliveryTag, multiple));
        }

        @Override
        public void handleNack(long deliveryTag, boolean multiple) throws IOException {
            System.out.println(String.format("MQ接收消息[%s:%b]失败，消息丢失", deliveryTag, multiple));
        }
    });

我们来看下异步确认程序的执行结果：

	MQ接收消息[2:true]成功
	MQ接收消息[3:false]成功
	MQ接收消息[4:false]成功
	MQ接收消息[5:false]成功
	MQ接收消息[6:false]成功

从执行结果中可以看出，是否批量确认在于返回的multiple的参数，此参数为bool值。有时 `multiple` 返回的是 **true**，这说明批量确认了  响应的 `deliveryTag` 标识的该条消息之前的所有消息，返回 `false` 表示单条确认对应的 `deliveryTag` 这条消息。可以看到使用这种模式 `deliveryTag` 会从1依次递增标识消息。

# 3 总结

1. 一般情况下消费者会使用 `channel.basicQos(int prefetch)` 方法设置prefetch为1，达到同个消费者多个实例之间"能者多劳"的目的
2. 为了尽量的保证避免消息的丢失，应该同时采用**消费者手动ack**和**生产者confirm模式**
3. 虽然rabbitMQ还有事务机制保证生产者发送的消息一定会到达broker，但是本身性能受损比较严重，不建议使用，有兴趣的可以另行查找资料