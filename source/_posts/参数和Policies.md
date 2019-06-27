---
title: 参数和Policies
date: 2019-06-10 15:38:49
tags: RabbitMQ
categories: 中间件

---

## 前言

RabbitMQ中的队列和交换器，除了一些必须属性 *(eg：durable、exclusive等)* 外，还有一些可选的参数。 这些参数的形式都是 `x-argument`，可以在队列或者交换器定义的时候通过Map类型的参数指定，这些参数可以赋予队列或者交换器一些其他的特征

通过定义队列或交换器的时候指定具体的参数，这可能在某些情况下是个不错的选择。 但是，这种控制方式有一些缺点：

- 不能灵活的对参数进行增删改查操作
- 一次只能为一个队列或者交换器指定参数，不能一次性为多个队列或交换器设置参数

在这种情况下，RabbitMQ引入了 `Policies`

## 介绍 Policies
 
`Policies` 的主要的属性：

- name：名称，可以是除基于ASCII的名称以外的任何名称，不建议使用空格
- pattern：正则表达式，匹配队列或者交换器
- definition：定义一系列的键值对，也就是参数，会注入到对应的队列和交换器中
- priority：优先级

`Policies` 会自动的通过其 `pattern` 属性匹配到对应的队列或交换器，并且将其 `definition` 属性定义的参数注入到匹配到的队列或交换器中。 `Policies` 可以仅仅匹配队列、也可以仅仅匹配交换器，甚至可以两者同时匹配，这要取决于其 `apply-to` 指定的类型(`queues`、`exchanges`、`all`) ，默认的值是`all`。 每个队列或者交换器最多只能匹配有一个 `Policies`。 当更新 `Policies` 中相关的参数时，相关的队列或者交换器也会立即更新，对于比较繁忙的队列可能需要花费一点事时间。 当队列或者交换器每次创建的时候都会进行 `Policies` 的匹配和使用，不只是当 `Policies` 创建的时候

## 创建Policies

- 使用 `rabbitmqctl` 创建 Policies

		rabbitmqctl set_policy policies_name "^amq\." '{"federation-upstream-set":"all"}' --priority 1 --apply-to exchanges

- 使用 **WEB UI** 创建 Policies：Admin > Policies > Add/update a policy

![create_policies](/../img/201905/create_policies.png)


## Operator Policies

有时我们可能会希望可以强制控制一个 `policies` 的某些参数，但同时不影响其他的参数的设置，这个时候就可以同时使用 Operator Policies。 一个队列可以同时匹配一个普通的 `policies` 和一个 `Operator Policies`。 但是并不是每个参数都可以强制设定，`Operator Policies` 只允许下列参数设置：

- `expires`：设置队列超时时间
- `message-ttl`：设置消息超时时间
- `max-length`：队列中未分发的最大消息数目
- `max-length-bytes`
- `overflow`

如果一个队列同时匹配了一个普通的 `policies` 和一个 `Operator Policies`，并且存在上述几个参数的设置冲突，则会选用两者中较小的那个值，因此，`Operator Policies` 不是覆盖普通 `policies` 的对应参数，而是在强制的同时尽量的不影响普通的 `policies`

## 定义 Operator Policies

- 使用 `rabbitmqctl` 创建

		rabbitmqctl set_operator_policy transient-queue-ttl "^amq\." '{"expires":1800000}' --priority 1 --apply-to queues

- 使用 **WEB UI** 页面创建： admin > Polocies > Add / update an operator policy

![create_operator_policies](/../img/201905/create_operator_policies.png)