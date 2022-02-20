# AMQP 学习笔记

## 概念理解

1. 消息传输的相关协议有哪些？
   - AMQP
     - 实现有RabbitMQ（Erlang，v0.9.1）, Apache ActiveMQ（Java）, Apache Qpid
     - 分布式，消息持久化，高性能高可靠
   - MQTT
     - 适合物联网
     - 低带宽，不稳定
   - Kafka
     - 基于TCP/IP 的二进制协议
     - 结构简单，效率高，无事务支持，有持久化
   - OpenMessage
     - 实现有RocketMQ
     - Cloud Native
  
2. 为什么不直接使用HTTP协议？
   - HTTP请求报头和响应报头都比较复杂，包含cookie，数据加密解密，状态码，响应码等附加功能。但对于一个消息，并不需要复杂的报头，只需要负责数据的传输，存储，分发。
   - 大部分HTTP是短连接，响应容易中断，造成请求丢失。而消息中间件传输是需要一个长期获取，同时传输可靠的场景。

3. AMQP基本概念
   - 组成部分
     - Container
       - Application Client, Broker
     - Node
       - responsible for the safe storage and/or delivery of Messages. 
       - Producers, Consumers, and Queues
     - Connection 
       - Real TCP connection  
       - Container to Container, Frame Transport Layer
     - Session
       - Command Dispatcher
       - Container to Container, Frame Transport Layer
     - Channel
       - Virtual connection attached to one Connection
       - Unidirectional, Frame Transport Layer
     - Link
       - Unidirectional, Source to Target
       - Node to Node, Message Transport Layer    
   - 相互关系
     - 一个 Container 包含了很多 Nodes (1 Container - n Nodes)
     - 两个 Nodes 之间通过多个 Links 来连接 (2 Node - n Links)
     - 两个 Container 之间包含多个 Connections (2 Container - n Connections)
     - 一个 Connection 可以有多个 Session
     - 一个 Session 可以有多个Links，这样可以同时传输多个数据
     - 一个 Session 只能有两个Channel (ICH 和 OCH)
  
   - 总结
     - Connection 是代表的是实际的 TCP 连接，通过复用多个 Channel 来同时传输不同数据。由于Channel只是单向的，所以两个反向 Channel 组成了一个 Session，从而一个 Connection 可以使用多个 Session 来同一时刻传输不同的数据。
     - Connection，Session, Channel 都是 Frame 层面的概念，而 Link 则是 Messge 层面的概念，Link 需要 attached 到一个 Connection/Session 来传输，所以一个Connection/Session 上可能存在有多个Links来复用传输信息 。
  

  
## Proton-J

Reactor


Selector 

Event
- type
- 

Delivery