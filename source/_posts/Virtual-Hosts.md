---
title: Virtual Hosts
date: 2019-06-10 09:17:11
tags: RabbitMQ
categories: 中间件

---

## 介绍vhosts

在RabbitMQ中，**virtual hosts** 提供了逻辑上的资源隔离，这些资源包括连接、交换器、队列、 绑定、用户权限、策略等等都属于vhosts。 可以这样认为，用户在使用RabbitMQ的时候，所有的操作都是在vhosts下完成的，如果没有指定vhost则使用默认的vhost: `/`

创建virtual host可以使用 `rabbitmqctl` 或者 `HTTP API`。 当创建了一个vhost后如果用户想要使用这个vhost，则需要授予该用户相应的权限，默认新用户是没有任何权限的。

## 创建vhost

> 使用命令行创建。 创建一个名为 `vhost_name` 的vhost

	 	rabbitmqctl add_vhost [vhost_name]


> 使用 [HTTP API](https://www.rabbitmq.com/management.html)。 调用 `PUT /api/vhosts/{name}` 接口添加vhost

		curl -u userename:pa$sw0rD -X PUT http://rabbitmq.local:15672/api/vhosts/vh1
	
## 设置vhost的最大连接数和最大创建队列数

> 设置vhost的最大连接数

- 设置为正整数，控制最大连接数目

		rabbitmqctl set_vhost_limits -p vhost_name '{"max-connections": 256}'

- 设置最大连接数为 `0`，禁止连接到该vhost

		rabbitmqctl set_vhost_limits -p vhost_name '{"max-connections": 0}'

- 设置最大值为负数，不限制最大连接数

		rabbitmqctl set_vhost_limits -p vhost_name '{"max-connections": -1}'

> 设置vhost的最大队列数目

		rabbitmqctl set_vhost_limits -p vhost_name '{"max-queues": 1024}'

		rabbitmqctl set_vhost_limits -p vhost_name '{"max-queues": -1}'

## 通过 **WEB UI** 页面操作vhost

1. 创建vhost：【admin】 -> 【Virtual Hosts】

![crate vhost](/../img/201905/create_vhost.png)

2. 授权用户操作vhost的权限：【admin】 -> 【Virtual Hosts】 -> 【点击某个vhost】

![crate vhost](/../img/201905/vhost_grant.png)

3. 设置vhost的最大连接数和最大队列数目：【admin】 -> 【Limits】

![crate vhost](/../img/201905/vhost_limit.png)

## 代码中vhost的使用

代码中可以通过设置 `connectionFactory` 的virtual host来指定vhost，如果没有设置默认使用默认的vhsot： `/`

	public static Connection getConnection() throws Exception {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost(HostConstant.HOST_NAME);
        connectionFactory.setUsername(HostConstant.USERNAME);
        connectionFactory.setPassword(HostConstant.PASSWORD);
        connectionFactory.setVirtualHost("vhost_name");
        return connectionFactory.newConnection();
    }