---
title: 2，Direct类型交换器-基础使用
date: 2019-05-26 16:13:49
tags: RabbitMQ
categories: 中间件

---
# Direct交换器
## 1 建立连接，获取信道

**MQUtil.java**

	// 建立到代理服务器的连接
	public static Connection getConnection(String vhost) throws Exception {
		ConnectionFactory connectionFactory = new ConnectionFactory();
		factory.setHost("localhost");
		factory.setUsername("admin");
		factory.setPassword("admin");
        if (!StringUtils.isEmpty(vhost)) {
            connectionFactory.setVirtualHost(vhost);
        }
        return connectionFactory.newConnection();
    }

后续的动作都是用channel来完成，使用try-resource包装，使得connection和channel可以自动关闭

	try (Connection connection = MQUtil.getConnection(); Channel channel = connection.createChannel()) {...}

## 2 生产者推送消息

生产者推送消息到MQ只需要**RoutingKey**和**Exchange**即可：

	// 定义交换器. durable标识是否持久化
	channel.exchangeDeclare(exchange, BuiltinExchangeType.DIRECT, durable);

	// 推送消息
	channel.basicPublish(exchange, routingKey, props, body);

方法参数意义：

- exchange: 使用的交换器名称，如果使用默认的交换器，此处使用空字符串 `""`
- routingKey: 消息指定的RoutingKey，MQ会通过该RoutingKey将消息分发到通过该值绑定的队列中
- props: 该参数的类型是 `BasicProperties` 主要用来设置消息的一些信息，如contentType、headers等。 `MessageProperties` 是获取 `BasicProperties` 对象的一个工具类，只要用来获取不同类型的信息
- body: 消息本身内容，类型是 `byte[]`

## 3 消费者消费消息

### 3.1 定义交换器和队列
消费消息时不需要将 `Connection` 和 `Channel` 进行 `try-resource` 封装，因为消费者需要循环的监控接收消息，因此不能关闭信道和连接

	// 定义交换器	
	channel.exchangeDeclare(exchange, type, durable);
	// 定义队列
    channel.queueDeclare(quueName, true, false, false, null);
	// 将队列通过RoutingKey绑定到交换器上
    channel.queueBind(queue, exchange, routingKey);

这里主要看下 `queueDeclare` 方法，该方法的定义如下：

	Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,Map<String, Object> arguments) throws IOException;

- queue: 队列名称
- durable: 是否持久化
- exclusive: 是否排他，如果设置为true，则表示只能用于创建其的连接中，不能用于另外的连接*(例如：如果有两个客户端同时都是新创建连接，然后定义相同的一个 `exclusive` 属性为**true**的队列，则只有其中的一个能成功启动，另外一个启动会报错)*。 并且，当连接关闭或者断开的时候，该队列会被自动删除 *(即使定义的队列是持久化的)*
- autoDelete: 是否自动删除。 如果设置为**true**，则当没有消费者使用这个队列的时候，这个队列会被自动删除

### 3.2 消费消息

	// 消费者成功接收到消息，消费回调函数
	DeliverCallback deliverCallback = (consumerTag, message) -> {
        String receiveMsg = new String(message.getBody());
		// 手动确认消息的接收
        channel.basicAck(message.getEnvelope().getDeliveryTag(), false);
    };
	// 消费消息
    channel.basicConsume(QUEUE_NAME, deliverCallback, (consumerTag) -> {
        System.err.println(String.format("系统异常，消费者[%s]不能正常消费消息", consumerTag));
    });

消费消息的函数是 `basicConsume` 该函数定义如下：

	String basicConsume(String queue, DeliverCallback deliverCallback, CancelCallback cancelCallback) throws IOException; 

- queue：队列名称。 指定消费者从哪个队列中消费消息
- deliverCallback: 消费者成功接收MQ推送的消息的回调函数，主要在该函数中进行消息的处理以及手动ack。该函数有个重载，多了一个 `autoAck` 参数，标识是否自动确认消息的接收。如果设置为 **true** 则当消费者正常接收到消息就会向MQ自动确认，MQ就会将该消息从消息队列中删除*(即使消息后续没有被成功消费)*， 默认false，一般情况下都会使用默认false。 如果设置为 `false` 就需要在 `deliverCallback` 回调函数中进行手动确认。
- cancelCallback: 当MQ服务出现问题，导致不能正常消费消息 *(比如：队列被删除等)*，则会调用该回调函数。此外，消费者也可以调用 `channel.basicAck` 手动取消该消费者 

**如果想要消息的持久化，必须保证交换器持久化、队列持久化**