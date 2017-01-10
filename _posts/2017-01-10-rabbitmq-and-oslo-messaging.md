---
layout:     post
title:      RabbitMQ 和 oslo.messaging
subtitle:
date:       2016-08-23 23:10:00
author:     Liao
catalog:    true
header-img: img/post-bg-rabbitmq-and-oslo-messaging.jpg
permalink:  /rabbitmq-and-oslo-messaging/
tags:
    - OpenStack
---

RabbitMQ 是一个消息队列系统，它实现了 AMQP 协议。在 OpenStack 中，RabbitMQ 被广泛的作为 RPC 中间件使用，在 OpenStack 核心项目如 Nova, Cinder, Neutron 等服务中，内部组件的 RPC 调用都是通过消息队列完成的，而 RabbitMQ 是 OpenStack 场景下使用最为广泛的消息队列组件。

RabbitMQ 本质上是一个消息的代理服务器（broker），它使用 AMQP 协议来完成消息的路由和交换。这里会解释一下 AMQP 中的基本概念，如果想深入学习 RabbitMQ，可以先从官方的文档开始：https://www.rabbitmq.com/tutorials/tutorial-one-python.html

# AMQP 协议
AMQP 是一个开放的消息队列通信标准，要了解 RabbitMQ 的使用，就先要了解 AMQP 协议的基本概念。AMQP 消息路由必须有三部分：**exchange**, **queue** 和 **binding**.

![](../img/in-post/rabbitmq-and-oslo-messaging/pic1.png)

## 消息（message）

使用消息队列时，生产者创建消息，然后发布到代理服务器，这种通信方式是一种「发后即忘」（fire-and-forget） 的单向方式

消息包括两部分内容：

- payload，即要传输的数据，可以是任何内容
- label，用于描述 payload，RabbitMQ 用它来决定谁将获得消息的拷贝，这部分内容不会发送给消费者

消费者连接到代理服务器上，并订阅到队列（queue） 上，当消费者接收到消息时，它只得到消息的一部分：payload.

## 连接（Connection）和 信道（Channel）

AMQP 的连接一般都是长久的连接，通常使用 TCP 连接。应用程序首先需要连接到 Rabbit，才能消费或者发布消息，因此会创建一条 TCP  连接。一旦 TCP 连接打开（通过了认证），应用程序就可以创建一条 AMQP channel，channel 是建立在「真实的」 TCP 连接内的虚拟连接。AMQP 命令都是通过 channel 发送出去的。每个 channel 都会有一个唯一的 ID 来进行标识。

![](../img/in-post/rabbitmq-and-oslo-messaging/channel.png)

使用 channel 的原因在于，频繁的建立和销毁 TCP  会话是非常昂贵的开销。使用 channel 既能满足每个信道的私密性，又能避免给操作系统 TCP  栈造成额外负担。

## 队列（queue）

队列就像邮箱，消息最终会到达队列中等待消费。

- 消费者通过 AMQP 的 basic.sonsume 命令订阅队列。这样将 channel 设置为接收模式，消费者将自动接收消息。
- 也可以通过 basic.get 接收一条消息。

当 queue 上拥有多个消费者时，队列收到的消息将以轮询（round-robin） 的方式发送给消费者。每条消息只会发送给一个订阅的消费者。因此队列天生就可以用来做负载均衡。

### 创建队列

要使用队列必须先声明队列，如果队列不存在那么会创建此队列。消费者和生产者都能使用 AMQP 的 `queue.declare` 命令来声明队列，但是如果消费者在同一个 channel 上订阅了另一个队列的话，就无法再声明队列了。声明的队列如果已经存在，将不会做任何改动。如果声明的队列参数和已存在的队列属性不一致，那么将会返回一个错误码为 406 的异常。

队列可以设置中的参数：

- exlcusive，如果为 true，则队列将变成私有的，此时只有声明此队列的应用程序才能够消费队列消息
- auto-delete，当最后一个消费者取消订阅的时候，队列就会自动移除，通常用于临时队列
durability，重启 broker 之后队列是否还存在

**队列名称**

在声明队列时，可以指定队列名称，也可以由 Rabbit 随机生成一个名称并在 **queue.declare** 命令中返回（对于 RPC 应用来说，使用临时「匿名」队列很有用）。要使用随机的队列名称，在声明队列时使用空字符串作为队列名称参数。以 "amq." 开头的名称是 broker 内部使用的队列。

**持久队列**

持久队列的声明和属性信息会被持久化到硬盘中。因此 broker 重启也这些队列也不会丢失。非持久的队列被成为瞬时（transient）队列。要注意：持久队列仅是队列本身的持久化，**持久队列并不负责消息的持久化**。如果 broker 重启，持久队列会重新声明，但是只能恢复那些持久性的消息。

生产者和消费者都应该尝试去声明队列，这样能够保证队列的存在，以避免出现消息被路由到不存在的队列中，造成消息的丢失。

## Exchange

当发送消息时，消息会被发送给 exchange，然后，根据确定的规则，RabbitMQ 将会决定消息该投递到哪个队列。这些规则被称作 routing key，队列通过 routing key 绑定到 exchange。当消息发送到 MQ 时，消息将拥有一个 routing key（可以是空），MQ 会讲其和 binding 中的 routing key 进行匹配。如果相匹配，那么消息将会投递到该队列。如果路由的消息不匹配任何 binding pattern，消息将被忽略。

exchange 常用的属性有：
 - Name，名字
 - Durability，即 exchange 的持久性，默认情况下 broker 的重启丢失 exchange
 - Auto-delete，当最后一个消费者取消订阅的时候，是否删除队列，通常用于临时队列

AMQP 中总共有四中类型的 exchange：direct、fanout、topic 和 headers，每一种类型实现了不同的路由算法。

#### Default Exchange

Broker 会默认声明一个没有名字（empty string）的 exchange。当声明一个队列时，它会自动绑定到 default exchange，并以队列名称作为 routing key.

#### Direct Exchange

Direct exchange 根据 routing key 的直接匹配来将消息投递到队列中，它通常用于消息的单播。Direct exchange 通常用来在多个 worker 之间来进行任务的分配。消息在队列中通过 round robin 的方式来进行轮询。

#### Fanout Exchange

Fanout exchange 将所有的消息直接发送到所有绑定的队列中，并且忽略 routing key。Fanout 通常用于消息的广播。

#### Topic Exchange

Topic exchange 将消息的 routing key 与队列的 binding pattern 进行匹配，将消息路由到所有匹配消息 routing key 的队列中。它通常用于消息的多播。

#### Headers Exchange

Headers exchange 根据消息的 header 和 binding 中的值来进行匹配，进而完成消息的路由。

## Bindings

Bindings 指的是 exchange 用来将消息路由到队列所用的规则。为了将消息从 exchange 路由到队列上，队列必须绑定到 exchange 上。Binding 有一个 routing key 的属性被用来进行将消息发送到队列时的路由匹配，routing key 的作用类似于 filter。

如果一个消息不能被路由到任何队列上，这个消息可能被丢弃或者返回给发送者，具体取决于发送者设置的消息属性。

## Consumer

应用程序可以有两种方式来消费队列中的消息：

- 通过 "push API" 被动的接收推送的消息
- 通过 "pull API" 主动获取消息

使用 "push API" 时，消费者需要指定从哪个队列接收消息，即订阅一个队列。一个队列可以有多个订阅的消费者，也可以声明一个互斥队列（只能有一个消费者）。每个队列的消费者都有一个标签（tag）作为标识。

### 消息确认
消费者在收到一条消息之后，可能会在处理的过程中崩溃，或者由于网络的原因断开了和 AMQP broker 的连接。队列中的消息被消费者接收之后何时从队列中删除，有两种方式：

- broker 将消息发送给消费者之后就立即删除（使用 `basic.deliver` 或 `basic.get-ok` 方法）
- 消费者返回 ACK 之后才删除（使用 `basic.ack` 方法）

第一种方式被称作自动确认模型（automatic acknowledgement model），第二种称作显示确认模型（explicit acknowlegement model）。使用显示确认模型时，消费者可以自行决定何时发送 ACK。如果消费者没有返回 ACK， broker 会将消息重新发送给其他消费者，如果此时队列没有可用的消费者，broker 会等待直至队列至少存在一个消费者再将消息重新发送到队列。只有确认了消息之后，RabbitMQ 才会给消费者发送下一条消息。

消费者收到的每一条消息都必须进行确认。消费者必须通过 `basic.ack` 命令像 MQ 发送一个确认，或者在订阅队列的时候将 auto_ack 设置为 true。消费者通过确认通知 MQ  它已经正确的接收了消息，同事 RabbitMQ 才能安全的把消息从队列中删除。

### 消息属性和载荷

消息可以带有属性（attribute），有时候属性也被称作 header ，常见的有：

- Content type
- Content encoding
- Routing key
- Delivery mode(persistent or not）
- Message priority
- Message publishing timestamp
- Expiration period
- Publisher appliaction id

消息也包含载荷（payload，即包含的数据），消息也可以只有属性而没有载荷。消息可以作为持久消息发送，broker 将会把这些消息保存到硬盘中。即使服务重启了这些消息也不会丢失。而把消息发送到一个 durable exchange 或者 durable queue 并不会使得这个消息持久化。持久化的消息会比较损耗性能。

# Oslo.messaging

在 OpenStack 的各个项目中都会使用到 RPC 调用，为了避免代码重复，简化开发工作，社区把 RPC 相关的功能作为 OpenStack 的一个基础库独立出来，即 oslo.messaging 项目。oslo.messaging 可以支持多种不同的消息队列，使用 RabbitMQ 时，oslo.messaging 底层可以选择使用 kombu 或者 pika 来完成消息层的处理。

oslo.messaging 对底层消息队列进行了一些高级封装，因此引入了一些新的概念，用以屏蔽掉底层消息队列中开发者不需要了解的细节。这些概念有：

- Transport：指的是 RPC 调用底层使用的消息传输介质，所有的 RPC 调用都是在 Transport 之上完成的
- Target：是一个通用概念，被用来进行 RPC 调用的目标匹配
- RPC Server：即接受 RPC 调用方，需要创建相应的队列，exchange，并以某个 routing key 绑定到 exchange 上
- RPC Client：即 RPC 的调用者，它需要通过 Target 指定 RPC server 的一些信息用来匹配对应的 RPC Server
- RPC Endpoint：指的是 RPC Server 中可被调用的方法集合，在 Endpoint 中可以通过 Target 来对收到的 RPC 进行一些精确匹配

## Transport

Transport 就是 RPC 调用过程中，使用的消息通信介质，如果我们使用 rabbitmq，那么需要指定 rabbitmq 服务器的连接地址，以及用户名，密码等参数。RPC 调用的 client 和 server 端都需要指定一个 transport 作为消息的 broker.

oslo.messaging 中通过 `oslo_messaging.get_transport` 函数返回一个 transport 对象，如：

```python
import oslo_messaging as messaging
from oslo_config import cfg

CONF = cfg.CONF

transport = messaging.get_transport(CONF,
        url='rabbit://rabbit:passwd@localhost:5672/')
```

## Target


Target 对象代表一个调用需要匹配的目标。RPC 客户端需要指定 Target 来进行 RPC 调用，RPC server 段也要指定 Target 来说明接收哪些 RPC 调用。Target 在底层被用来决定 RPC server 需要创建哪些队列，使用哪些 routing key 来绑定到 exchange 上，以及 RPC client 发送消息的 routing key。

Target 类的原型如下：

```python
class oslo_messaging.Target(exchange=None, topic=None, namespace=None, version=None, server=None, fanout=None, legacy_namespaces=None)
```

这里的 exchange, topic, namespace, server， fanout 等参数会被用于完成 exchange 的声明，队列的创建，binding 的创建以及 routing key 的选择等。而 namespace, version 等参数是 oslo.messaging 为了实现更精确的匹配规则创建的概念。

RPC 中的各个组件都需要使用这个 Target 对象，他们在使用时需要指定的参数如下：

- RPC Server: 必须指定 topic 和 server，还可以指定 exchange
- RPC endpoint: 可以指定 namespace 和 version
- RPC client: 必须指定 topic，其他均为可选项
- Notification Server：必须指定 topic，还可以指定 exchange
- Notifier: 必须指定 topic，还可以指定 exchange

## RPC Server

RPC Server 通过 Endpoint，将方法暴露出去，供 Client 端进行调用。一个 RPC Server 可以指定多个 Endpoint 对象。

RPC Server 的构造函数为：

```python
oslo_messaging.get_rpc_server(transport, target, endpoints, executor='blocking', serializer=None, access_policy=None)
```

这里是 endpoints 参数是一个列表，包含所有的 endpoints 对象。以 nova-compute 为例，创建 RPC Server：

```python
LOG.debug("Creating RPC server for service %s", self.topic)

target = messaging.Target(topic=self.topic, server=self.host)

endpoints = [
    self.manager,
    baserpc.BaseRPCAPI(self.manager.service_name, self.backdoor_port)
]
endpoints.extend(self.manager.additional_endpoints)

serializer = objects_base.NovaObjectSerializer()

self.rpcserver = rpc.get_server(target, endpoints, serializer)
self.rpcserver.start()
```

这里 Nova 使用了两个 endpoints 对象，第一个 `self.manager` 是一个 `ComputeManager` 对象，其中实现了对虚拟机操作的所有 RPC 调用，第二个 endpoints 是一个基础的 enpoints 包含几个通用方法。

RPC Server 的创建后，oslo.messaging 在底层会创建相应，并将队列绑定到 exchange 上。创建一个 RPC Server 在 oslo.messaging 中会调用下面的方法创建队列：

```python
def listen(self, target, batch_size, batch_timeout):
    conn = self._get_connection(rpc_common.PURPOSE_LISTEN)

    listener = AMQPListener(self, conn)

    conn.declare_topic_consumer(exchange_name=self._get_exchange(target),
                                topic=target.topic,
                                callback=listener)
    conn.declare_topic_consumer(exchange_name=self._get_exchange(target),
                                topic='%s.%s' % (target.topic,
                                                target.server),
                                callback=listener)
    conn.declare_fanout_consumer(target.topic, listener)
```

这里可以看到创建三个队列，两个 topic 类型，分别以 "topic" 和 "topic.server" 为名称，同时也是队列绑定到 exchange 的 routing key. 还有一个 fanout 类型的队列用作 notification.

## RPC Client

RPC 的调用都要通过 RPC Client 来完成。创建一个 RPC Client 需要指定 Transport 和 Target.

```python
class oslo_messaging.RPCClient(transport, target, timeout=None, version_cap=None, serializer=None, retry=None)
```

RPCClient 的作用就是通过 Target 中设置的参数来找到 RPC 调用需要发送的 exchange 和 routing key。虽然 target 是在创建 RPC Client 的时候指定的，在某些调用中也可以通过 RPCCLient 的 `prepare()` 方法重载 target 中的属性。例如在某些调用中设置一个特殊的 Target namespace 或者 version.

#### call 调用

RPCClient 可以发起 call 调用，此时线程会阻塞直至收到调用的返回结果。call() 调用会在调用时创建一个用于接收返回消息的 direct exchange 和队列，并监听在此队列上。一张图解释整个 call() 调用过程如下：

![](../img/in-post/rabbitmq-and-oslo-messaging/call.png)

call() 方法接收的参数分别为请求的 context dict，需要调用的方法，和方法的参数。由于 call 调用是阻塞的，因此程序中的 call() 是保证按顺序执行的。

#### cast 调用

cast 调用是以非阻塞的方式来进行 RPC 调用（例如 Nova 中的虚拟机重启）。cast 调用可以发送到 fanout exchange 中。由于 cast() 是非阻塞的，因此程序中的 cast 调用不会保证按顺序执行。

了解了这些概念之后，OpenStack 中关于 RPC 调用过程理解起来就很简单了。由于篇幅有限，本文没有介绍 oslo.messaging 中的 notification，serializer 等概念，但是这些概念相对来说比较简单，读者可以自行研究。

