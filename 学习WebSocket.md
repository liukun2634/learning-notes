# 学习 WebSocket

## 基本概念
> The WebSocket Protocol enables two-way communication between a client running untrusted code in a controlled environment to a remote host that has opted-in to communications from that code. The protocol consists of an opening handshake followed by basic message framing, layered over TCP.  
> The goal of this technology is to provide a mechanism for browser-based applications that need two-way communication with servers that does not rely on opening multiple HTTP connections (e.g., using XMLHttpRequest or `<iframe>`s and long polling).

WebSocket 目的是为了给网页应用提供双向通信，使用 XMLHttpRequest和长连接的polling 保持连接，而不是需要开启多个 HTTP Connection。

1. WebSocket 与 TCP，HTTP 的关系
   - WebSocket 是基于 TCP 上面的独立应用层协议，提供了双向通信。

   - WebSocket 与 HTTP 的唯一关系就是 handshake 时候向 HTTP server 发送 Upgrade request 从而升级为 WebSocket 协议([HTTP 升级机制](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Protocol_upgrade_mechanism))。因而 WebSocket 只利用了HTTP 来建立一个HTTP Connection，后面则转换为 WebSocket 真正传输数据。
  
2. WebSocket 使用端口
   - 默认使用 80 端口
   - 可以通过 Upgrade 升级到 Transport Layer Security(TLS) 使用 443 端口

3. 相关 RFC （Request For Comments）协议
   - [The WebSocket Protocol (RFC6455)](https://datatracker.ietf.org/doc/html/rfc6455)
   - [Hypertext Transfer Protocol -- HTTP/1.1 (RFC2616)](https://datatracker.ietf.org/doc/html/rfc2616)
   - [HTTP Over TLS (RFC2818)](https://datatracker.ietf.org/doc/html/rfc2818)

4. Upgrade Request 实例
   ```
   GET /chat HTTP/1.1
   Host: server.example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key:dGhlIHNhbXBsZSBub25jZQ==
   Origin: http://example.com
   Sec-WebSocket-Version: 13
   ```