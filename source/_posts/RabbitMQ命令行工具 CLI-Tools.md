---
title: RabbitMQ命令行工具 CLI Tools
date: 2019-06-21 17:40:10
tags: RabbitMQ
categories: 中间件

---

# RabbitMQ命令行工具 CLI Tools

RabbitMQ主要有四个命令行工具 Commend Line Tools，不同的命令行适用不用的场景

1. `rabbitmqctl`：负责服务管理和进行操作
2. `rabbitmq-diagnostics`：负责系统诊断和健康检查
3. `rabbitmq-plugins`：负责插件管理
4. `rabbitmqadmin`：用来操作 *HTTP API*

## 系统环境要求

- CLI Tools要求当前系统必须安装Erlang环境，具体的兼容版本可以参照： [RabbitMQ的Erlang版本要求](https://www.rabbitmq.com/which-erlang.html)
- CLI Tools要求当前系统编码设置是UTF-8(例如：`en_GB.UTF-8` or `en_US.UTF-8`)，否则将会出现警告信息

## rabbitmqctl命令

`rabbitmqctl` 使用服务节点间共享的机密认证，该命令主要的功能包括以下几点：

- 停止节点运行
- 获取节点状态、有效配置、健康检查
- Virtual Hosts管理
- 用户和权限管理
- Policy管理
- 查看queues、connections、channels、exchanges和consumers列表信息
- 集群会员身份管理

## rabbitmq-plugins命令

`rabbitmq-plugins` 是一个管理多种插件的命令，包括获取插件列表以及启用/停用，是跟随RabbitMQ一起安装的。支持在线和离线模式，离线模式下做的改动会在节点重新启动时生效。`rabbitmq-plugins` 使用服务节点间共享的机密认证

## 关于节点的名称 Node Name

RabbitMQ集群中的节点标识和互相通信都使用该节点的**唯一标识-该节点名称**，节点名称有前缀(通常是`rabbit`)和主机名构成，例如：`rabbit@node1.messaging.svc.local`。

如果在一台主机上同时启动多个节点则节点前缀部分一定不能相同，例如：`rabbit1@hostname` 和 `rabbit2@hostname`。CLI tools识别节点地址就是通过该节点的名称，大部分的CLI命令都会通过 `--noe`或者 `-n` 参数指定目标节点。

当一个节点启动的时候，会自动校验是否被分配了名称，该节点名称可以通过环境变量 `RABBITMQ_NODENAME` 来明确指定，如果没有设置，则默认会取 `rabbit+本机的hostname`。如果系统用的是全限定域名(FQDN)，有两种方案选择：

1. 节点必须配置环境变量 `RABBITMQ_USE_LONGNAME` 为 `true`
2. 在命令最后添加 `--longnames` 选项 

## CLI Tools认证和节点间通信认证

集群中每个节点之间是否可以进行通信，还依赖于这两个节点上保存的 `Erlang Cookie` 是否相同，一个集群中的所有节点的Erlang Cookie 应该是相同的。这个值是一个由字母和数字组成的字符串，最大字符数是***255***，保存在节点所在的主机文件中。

如果没有指定每个节点启动的时候Erlang VM会随机生成一个Erlang Cookie，并且自动的创建一个文件用来存放，这种方式每次生成的Erlang Cookie都不相同，不适合生产环境。unix和windows系统上该文件的存放位置不同(注：需要确保该文件的使用者有足够的权限)

> Cookie文件的地址

UNIX系统存放位置：

- 节点之前通信使用：/var/lib/rabbitmq/.erlang.cookie
- CLI Tools使用：$HOME/.erlang.cookie

WINDOWS系统存放位置：不同的Erlang版本存放位置不同，具体可以参考：[Cookie文件说明](https://www.rabbitmq.com/clustering.html#erlang-cookie)

> 最不安全的设置Cookie方式

可以通过设置节点的环境变量 `RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS` 为 `-setcookie <cookie-value>` 来设置cookie值。但是这种方式是最不安全的方式，不推荐使用

## rabbitmqadmin命令

`rabbitmqadmin` 命令是构建在 **HTTP API** 的基础上，该工具要求 `Python 2.7.9` 或者以上版本。 `rabbitmqadmin` 使用 **HTTP API** 的认证机制，基于HTTP认证，该命令默认并没有集成到RabbitMQ安装包中，需要额外的进行安装才能使用

## 命令别名

`rabbitmqctl`, `rabbitmq-diagnostics` 和 `rabbitmq-plugins` 都支持命令别名设置，命令别名提供一种更加简短的命令版本以及命令参数。例如，命令 `rabbitmqctl environment` 可以通过命令别名设置为 `rabbitmqctl env`来达到同样的目的。 命令别名通过环境变量 `RABBITMQ_CLI_ALIASES_FILE` 指定的文件地址来配置。

设置环境变量 `RABBITMQ_CLI_ALIASES_FILE` 的值：
    
    export RABBITMQ_CLI_ALIASES_FILE=/path/to/cli_aliases.conf

`cli_aliases.conf` 文件内容如下：

	env = environment
	st  = status --quiet
	
	lp  = list_parameters --quiet
	lq  = list_queues --quiet
	lu  = list_users --quiet
	
	cs  = cipher_suites --openssl-format --quiet

像上述这样设置后

`rabbitmqctl env` 相当于 `rabbitmqctl environment`
`rabbitmqctl lq` 相当于 `rabbitmqctl list_queues --quiet`
`rabbitmq-diagnostics cs` 相当于 `rabbitmq-diagnostics cipher_suites --openssl-format --quiet`
