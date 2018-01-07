---
title: RabbitMQ基础概念及操作
tags: RabbitMQ
grammar_cjkRuby: true
---


## 基础概念
### Queue
Queue（队列）是RabbitMQ的内部对象，用于存储消息
消息只能存储在Queue中，生产者生产消息并最终投递到Queue中，消费者可以从Queue中获取消息并消费。
多个消费中可以订阅同一个Queue，这时Queue中的消息会被**平均分摊**给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理。
#### Message acknowledgment
在实际的应用中，可能会发生消费者收到Queue中的消息，但还没有处理完就宕机的情况，这种情况下可能会导致消息丢失。为了避免这种情况，我们可以要求消费者在消费完后发送一个回执给RabbtMQ，RabbitMQ收到消息回执（Messageservice acknowledgment）后才将消息从Queue中移除；如果RabbitMQ没有收到回执并且检查到消费者的RabbitMQ链接断开，则RabbitMQ会将改消息发送给其他消费者进行处理。这里**不存在timeout**的概念，一个消费者处理消息时间再长也不会导致该消息被发送给其他消费者，除非它与RabbitMQ连接断开。
如果完成业务后，忘记发送回执给RabbitMQ，就会导致严重的bug，Queue中堆积的消息会越来越多，消费者重启后会重复消费这些消息并且重复执行业务逻辑。
pub message是没有ack的 ？？？
#### Message durability
如果我们希望在RabbitMQ服务重启的情况下，消息也不会丢失，我们可以将Queue与Message都设置为durable，这样可以保证绝大部分情况下消息不会丢失。但比如RabbitMQ服务器已经收到生产者消息，但还没来得及持久化就断电了，要解决这种小概率的问题，就要用到事务。
#### Prefetch count
如果多个消费者同时订阅同一个Queue中的消息，消息会被平摊到它们。如果每个消息的处理时间不同，就可能会导致某些消费者一直在忙，另外一些很快处理完手头工作并且一致空闲。可以通过设置prefetchCount来限制Queue每次发送给每个消费者的消息数。例如设置prefetchCount=1，则Queue每次给每个消费者发送一条消息，消费者处理完这条消息后Queue会再给该消费者发送一条消息。
### Exchange
生产者将消息发送给Exchange，有Exchange将消息路由到一个或多个Queue中（或者丢弃）。
#### routing key
生产者在将消息发送给Exchange时候，一般会指定一个routing key，来指定消息的路由规则。routing key需要与Exchange type及binding key联合使用才能最终生效。
在Exchange Type和binding key固定的情况下，生产者可以在发送消息给Exchange时，通过routing key来决定消息流向哪里。
RabbitMQ routing 可以的长度限制为255bytes
#### Binding
binding将Exchange与Queue关联起来，这样就知道如何正确的将消息路由到指定的queue了。
#### Binding key
在Binding Exchange与Queue的同时，一般会指定binding key；生产者将消息发送给Exchange时，一般会指定routing key，**当binding key与routing key相匹配时**，消息将会被路由到对应的Queue中。
绑定多个Queue到同一个Exchange时，Binding允许使用相同的biding key。
### Exchange Types
常用四种：fanout，direct，topic，headers
#### fanout
会把所有发送到该Exchange的消息路由到与它绑定的Queue中。
![enter description here][1]
#### direct
将消息路由到binding key与routing key完全匹配的Queue中。
![enter description here][2]
我们以routingKye="error"发送消息给Exchange，这消息会路由到Queue1和Queue2，如果以routingKey="info"或routingKey="warning"来发送消息，则消息只会到Queue2。如果以其他routingKye发送消息，则消息不会到这两个Queue中。
#### topic
direct是**严格完全匹配**binding key与routing key。topic在匹配规则上进行了扩展，它与direct相似，将消息路由到binding key与routing key匹配的Queue中，但这里的匹配规则有些不同，它约定：
* routing key为“.”分隔的字符串，如“stock.usd.nyse”、“abc.def”
* binding key格式同routing key
* binding key可以存在两种特殊字符“*”和“#”用于模糊匹配。星�用于匹配一个单词，井号用于匹配多个单词（可以是0个）

![enter description here][3]
routingKey="quick.orange.rabbit"会同时路由到Q1，Q2，"lazy.orange.fox"会路由到Q1，"lazy.brown.fox"会路由到Q2，"lazy.pink.rabbit"会路由到Q2（只会投递给Q2**一次**，虽然routingKey与Q2的两个binding key都匹配）；"qucik.brown.fox","orange"，"quick.orange.male.rabbit"都会被丢弃。

#### headers
headers类型的Exchange不依赖routing key和binding key的匹配规则来路由，它是根据消息内容中的headers属性进行匹配的。
在绑定Queue与Exchange时指定一组键值对，当消息发送到Exchange是，RabbitMQ会渠道消息的headers（也是一种键值对的形式），对比其中的键值对是否完全匹配Queue与Exchange绑定时指定的键值对，如果完全匹配则消息路由到该Queue。

### RPC
RabbitMQ本身是基于异步消息处理，生产者将消息发送到RabbitMQ后不知道消费者处理成功或失败（甚至消费者有没有处理该消息都不知道）。
在实际场景中，有需要同步处理，需要同步等待服务端将消息处理完后再进行下一步处理，这相当于RPC（Remote Procedure Call）。
![enter description here][4]
RabbitMQ的RPC机制是：
* 客户端发送请求（消息）时，在消息的属性（MessageProperties，在AMQP协议中定义了14种properties，这些属性会随消息一起发送）中设置两个值replyTo（一个Queue名称，告诉服务器处理完成后将通知我的消息发送到这个Queue中）和correlationId（此次请求的标志号，服务器处理完成后将此属性返还，客户端根据这个id了解哪条请求被成功执行了或执行失败）
* 服务器收到消息并处理
* 服务器处理完消息后，将生成一条应答消息到replyTo指定的Queue，同时带上correlationId属性
* 客户端之前已订阅replyTo指定的Queue，从中收到服务器应答消息后，根据其中的correlationId属性分析哪条请求被执行了，根据执行结果进行后续业务处理


  [1]: ./assets/2014-2-21-9-54-26.png
  [2]: ./assets/2014-2-21-9-55-20.png
  [3]: ./assets/2014-2-21-9-57-37.png
  [4]: ./assets/2014-2-21-9-59-04.png