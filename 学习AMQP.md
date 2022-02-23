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

3. AMQP 协议结构

   - AMQP version negotiate

     ```
        AMQP
         |
       TCP/IP
     ```
   - After AMQP version negotiate

     ```
        AMQP
         |
      TSL/SASL(optional)
         |
       TCP/IP
     ```
## AMQP Transport
1. 组成部分
   - Container
      - Application Client, Broker
   - Node
      - Responsible for the safe storage and/or delivery of Messages. 
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
   
2. 相互关系
   - 一个 Container 包含了很多 Nodes (1 Container - n Nodes)
   - 两个 Nodes 之间通过多个 Links 来连接 (2 Node - n Links)
   - 两个 Container 之间包含多个 Connections (2 Container - n Connections)
   - 一个 Connection 可以有多个 Session
   - 一个 Session 可以有多个Links，这样可以同时传输多个数据
   - 一个 Session 只能有两个Channel (ICH 和 OCH)
  
3. 总结理解
   - Connection
     - Connection 就是代表的是实际的 TCP 连接。
     - 通过复用多个 Channel 来同时传输不同数据。
   - Channel：
     - Channel 用于真正传输单向数据的。
     - 可以想象实现上用 socketChannel 的单向读写，然后多个Channel 复用构成了一个Connection。
   - Session
     - 由于 Channel 只是单向的，所以两个反向 Channel 组成了一个 Session，从而一个 Connection 通过多个 Session 来同一时刻传输不同的数据。
      - 可以理解 Session 就是真正用来传输双向数据的，两个Channel 组成 一个 Session， 多个Session 复用 构成了 一个Connection。
   - Link 
     - Link 则是 Messge 层面的概念，Connection，Session, Channel 都是 Frame 层面的概念，所以 Link 是位于 Session的上面。
     - 当Source 有 Message 需要传输时，Source 会通过找到 Target 构建之间 Link 来传输 Message。
     - Link 需要 attached 到一个 具体的 Session 来传输Message，Session 会把 Message 拆分成多个 Frame 在Channel 上发送和接收。因为是Link attach 上 Session， 所以 一个 Session 上可能存在有多个 Links 来复用传输信息。
   - Connection，Session，Link 这三种 Communication Endpoint 关系图
  
     ```
        +-------------+
        |    Link     |  Message Transport
        +-------------+  (Node to Node)
        | name        |
        | source      |
        | target      |
        | timeout     |
        +-------------+
             /|\ 0..n
              |
             \|/ 0..1
        +------------+
        |  Session   |  Frame Transport
        +------------+  (Container to Container)
        | name       |  (Include Two Channels)
        +------------+
             /|\ 0..n
              |
             \|/ 1..1
        +------------+   
        | Connection |  Frame Transport
        +------------+  (Container to Container)
        | principal  |  
        +------------+ 
     ```


## Framing

1. 组成
   - frame header 
     - 大小 8 bytes
     - SIZE: header 前4个字节，标明 frame 整个字节数大小, 无符号 32 位整型
     - DOFF：header 第5个字节，用于标志frame body 的位置（位置是DOFF值的大小再乘以4的字节数），无符号 8 位整型
     - TYPE： header 的第6个字节，标明type code, 其中 0x00 表示是 AMQP frame, 0x01 表示是 SASL frame。
     - TYPE-SPECIFIC： header的7和8字节，
       - 对于 AMQP frame, 该位置是CHANNEL NUMBER。
   - extended header 
     - 大小 (DOFF * 4 - 8) bytes，去掉header的8 bytes
     - TYPE-SPECIFIC
       - 对于AMQP frame，直接忽略该段内容
   - frame body
     - TYPE-SPECIFIC
       - 对于AMQP frame，body的开头部分是PERFORMATIVE信息（Open/Begin/Attach/..）

2. AMQP frame
   ```

                 type: 0x00 - AMQP frame

              +0       +1       +2       +3
          +-----------------------------------+ -.
        0 |                SIZE               |  |
          +-----------------------------------+  |---> Frame Header
        4 |  DOFF  |  TYPE  |     CHANNEL     |  |      (8 bytes)
          +-----------------------------------+ -'
          +-----------------------------------+ -.
        8 |                ...                |  |
          .                                   .  |---> Extended Header
          .             <IGNORED>             .  |  (DOFF * 4 - 8) bytes
          |                ...                |  |
          +-----------------------------------+ -'
          +-----------------------------------+ -.
   4*DOFF |           PERFORMATIVE:           |  |
          .      Open / Begin / Attach        .  |
          .   Flow / Transfer / Disposition   .  |
          .      Detach / End / Close         .  |
          |-----------------------------------|  |
          .                                   .  |---> Frame Body
          .                                   .  |  (SIZE - DOFF * 4) bytes
          .             PAYLOAD               .  |
          .                                   .  |
          .                           ________|  |
          |                ...       |           |
          +--------------------------+          -' 
   ```

## Connection

## Session

> Messages transferred on a link are sequentially identified within the session. A session can be viewed as multiplexing link traffic, much like a connection multiplexes session traffic. However, unlike the sessions on a connection, links on a session are not entirely independent since they share a common delivery sequence scoped to the session. This common sequence allows endpoints to efficiently refer to sets of deliveries regardless of the originating link. This is of particular benefit when a single application is receiving messages along a large number of different links. In this case the session provides aggregation of otherwise independent links into a single stream that can be efficiently acknowledged by the receiving application

多个Link 复用同一个Session 来有效的传输数据，这些Link 在Session上并不是完全独立的，因为他们都用了相同的delivery sequence，从而Session 只需要关注传输部分，使得传输更有效率


End Session：
Sessions end automatically when the connection is closed or interrupted. Sessions are explicitly ended when either endpoint chooses to end the session.

## SASL 和 SSL/TSL 协议
1. SSL/TSL 协议
   - SSL (Secure Socket Layer 安全套接层) 是一种间于传输层和应用层的协议。起初是 HTTP 在传输数据时使用的是明文（虽然说POST提交的数据时放在报体里看不到的，但是还是可以通过抓包工具窃取到）是不安全的，因而推出了 SSL 来保证传输时候的数据安全，所以HTTPS是 HTTP + SSL/TCP 的简称。
   - SSL 的基本思想是用非对称加密来建立链接（握手阶段），用对称加密来传输数据（传输阶段）。这样既保证了密钥分发的安全，也保证了通信的效率(因为非对称加密更耗时)。
   - TSL (Transport Layer Security 安全传输层协议) 在SSL更新到3.0时，IETF对SSL3.0进行了标准化，并添加了少数机制(但是几乎和SSL3.0无差异)，标准化后的IETF更名为TLS1.0，可以说TLS就是SSL的新版本3.1。
   
2. SASL 协议
   - SASL (Simple Authentication and Security Layer 简单认证与安全层) 一个在网络协议中用来认证和数据加密的构架。具体实现可以有多种，包括 "PLAIN"，"DIGEST-MD5"，"GSSAPI" 等。

3. SASL 与 SSL/TSL 区别
   - SASL 用于认证
   - SSL/TSL 用于对传输的数据进行加密

Refer：[数据动态安全协议综述](https://silvermissile.github.io/2020/08/16/%E6%95%B0%E6%8D%AE%E5%8A%A8%E6%80%81%E5%AE%89%E5%85%A8%E5%8D%8F%E8%AE%AE%E7%BB%BC%E8%BF%B0/)


