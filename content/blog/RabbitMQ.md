+++
title = "RabbitMQ常见知识点"
date = 2023-03-03
[taxonomies]
  tags = ["Java"]
+++

# AMQP

AMQP定义了规范，RabbitMQ是规范实现之一。

# Docker 安装 RabbitMQ

1. 下载

```shell
docker pull rabbitmq:3.8.2-management
```

2. 运行

```shell
docker run -d --name rabbitmq3.8.2 -p 5672:5672 -p 15672:15672 -v /Users/felix/share/data:/var/lib/rabbitmq -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=afei123 rabbitmq:3.8.2-management
```

说明：
-d 后台运行容器；
--name 指定容器名；
-p 指定服务运行的端口（5672：应用访问端口；15672：控制台Web端口号）；
-v 映射目录；
-e 指定环境变量；（RABBITMQ_DEFAULT_VHOST：默认虚拟机名；RABBITMQ_DEFAULT_USER：默认的用户名；RABBITMQ_DEFAULT_PASS：默认用户名的密码）

3. 查看在容器中查看运行中的RabbitMQ

docker exec -it 645e674c1151 rabbitmqctl list_exchanges


# Consumer

多个消费者消费消息时，默认Round-robin轮询分发，若有的消费者处理得快，想让能者多得，多劳多得，则可以配置消费者 quality of service。

```
void basicQos(int prefetchCount);
```

## 消费者ack机制

保证消息一定被消费

# Exchanger Type

## fanout 

所有绑定的队列都收到消息

![img](/rabbitmq/python-three-overall.png)

## direct

消息由指定的的 routing key 路由，是特殊的fanout。subscribe only to a subset of the messages

![img](/rabbitmq/python-four.png)

## topic

消息由指定的routing key路由，可以支持通配符。

![img](/rabbitmq/python-five.png)

# Producer

## Reliable publishing

Use publisher confirms to make sure published messages have safely reached the broker.

1. 同步 confirms

```shell
waitForConfirmsOrDie();
```

2. 异步confirms

异步确认有一个序列号，用于异步绑定消息，参考：```com.rabbitmq.client.Channel#getNextPublishSeqNo```

。异步处理的时候添加一个异步回掉接口：

```java
com.rabbitmq.client.Channel#addConfirmListener(com.rabbitmq.client.ConfirmCallback, com.rabbitmq.client.ConfirmCallback)
```

## 事务

事务可参考：

```java
com.rabbitmq.client.Channel#txCommit();
com.rabbitmq.client.Channel#txRollback();
```



# 应用

## 采用RabbitMQ搞定超时订单

使用RabbitMQ来实现延迟任务必须先了解RabbitMQ的两个概念：消息的TTL和死信Exchange，通过这两者的组合来实现上述需求。

- **消息的TTL（Time To Live）**

消息的`TTL`就是消息的存活时间。RabbitMQ 可以对队列和消息分别设置`TTL`。对队列设置就是队列没有消费者连着的保留时间，也可以对每一个单独的消息做单独的设置。超过了这个时间，我们认为这个消息就死了，称之为死信。

那么，如何设置这个TTL值呢？有两种方式，第一种是在创建队列的时候设置队列的`"x-message-ttl"`属性，如下：

```java
Map<String, Object> args = new HashMap<String, Object>();
args.put("x-message-ttl", 6000);
channel.queueDeclare(queueName, durable, exclusive, autoDelete, args);
```

这样所有被投递到该队列的消息都最多不会存活超过6s。

另一种方式便是针对每条消息设置TTL，代码如下：

```java
AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
builder.expiration("6000");
AMQP.BasicProperties properties = builder.build();
channel.basicPublish(exchangeName, routingKey, mandatory, properties, "msg body".getBytes());
```

这样这条消息的过期时间也被设置成了6s。

> 但这两种方式是有区别的，如果设置了队列的TTL属性，那么一旦消息过期，就会被队列丢弃，而第二种方式，消息即使过期，也不一定会被马上丢弃，因为消息是否过期是在即将投递到消费者之前判定的，如果当前队列有严重的消息积压情况，则已过期的消息也许还能存活较长时间。 另外，还需要注意的一点是，如果不设置TTL，表示消息永远不会过期，如果将TTL设置为0，则表示除非此时可以直接投递该消息到消费者，否则该消息将会被丢弃。

单靠死信还不能实现延迟任务，还要靠`Dead Letter Exchange`。

- **Dead Letter Exchanges**

一个消息在满足如下条件下，会进死信路由，记住这里是路由而不是队列，一个路由可以对应很多队列。

1. 一个消息被`Consumer`拒收了，并且`reject`方法的参数里`requeue`是`false`。也就是说不会被再次放在队列里，被其他消费者使用。
2. 上面的消息的`TTL`到了，消息就过期了。
3. 队列的长度限制满了。排在前面的消息会被丢弃或者扔到死信路由上。

`Dead Letter Exchange`其实就是一种普通的`exchange`，和创建其他`exchange`没有两样。只是在某一个设置`Dead Letter Exchange`的队列中有消息过期了，会自动触发消息的转发，发送到`Dead Letter Exchange`中去。

