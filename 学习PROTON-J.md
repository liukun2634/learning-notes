# Qpid Proton-J

## 背景

### Apache Qpid
[Apache Qpid](https://qpid.apache.org/index.html) - Messaging built on AMQP


**Build AMQP applications**
- Qpid Proton - The AMQP messaging toolkit
- Qpid JMS - JMS with the strength of AMQP
  
**Deploy AMQP networks**
- Broker-J - A pure-Java AMQP message broker
- C++ broker - A native-code AMQP message broker
- Dispatch router - A lightweight AMQP message router

### Qpid Proton

[Qpid Proton](https://qpid.apache.org/proton/index.html) 是Apache 下面的用于处理 AMQP message toolkit，实现了AMQP 1.0 version 协议，因而可以与实现AMQP 1.0 的系统进行集成。

  - Qpid Proton (C,C++, go, ruby, python)
  - Qpid Proton-J (Java)
  - Qpid ProtonJ2 (Java)


## Qpid Proton-J 项目

### Reactor

ReactorChild
- Endpoint
- Selectable (用于存储selectableChannel，并用于判断是否可读可写)
- Acceptor (Connection 建立之前的)

### Endpoint

EndpointImpl implement ProntonJEndpoint

- Connection
- Transport
- Session
- Link
  - Sender
  - Receiver

#### Transport

用于从buffer中读取和写入数据，交给socketChannel 发送和接收。


#### Connection

session(ArrayList)
- sender(HashMap) -- Link
- receiver(HashMap) -- Link
- collector(list of events, head, tail)

### Selectable

Selectable 存储于 Selector 之中, 用 hashset 保存多个。


## Log

Frame trace 
- Enable: set environment varibale `PN_TRACE_FRM=true`


