---
title: "Websocket 协议分析"
date: 2022-03-27T20:10:23+08:00
draft: false

description: Websocket 协议分析
author: realzhangm
tags: ["网络协议"]
---

# 简述

# Websocket 协议建立
Websocket 协议的建立是客户端与服务端通过 HTTP 的 [Update 机制](https://developer.mozilla.org/en-US/docs/Web/HTTP/Protocol_upgrade_mechanism)完成的。即将当前的 HTTP 协议升级为 websockt 协议，这样一来，websockt 可以复用 HTTP 的连接。

总结如下：
```
1. 协议握手，HTTP update 机制
2. 协议建立，可以通过 websocket 双向数据传输了
```
## 握手流程

- 客户端请求协议升级
- 服务端响应升级状态

具的报文如下：
```
   The handshake from the client looks as follows:

        GET /chat HTTP/1.1
        Host: server.example.com
        Upgrade: websocket
        Connection: Upgrade
        Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
        Origin: http://example.com
        Sec-WebSocket-Protocol: chat, superchat
        Sec-WebSocket-Version: 13

   The handshake from the server looks as follows:

        HTTP/1.1 101 Switching Protocols
        Upgrade: websocket
        Connection: Upgrade
        Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
        Sec-WebSocket-Protocol: chat

```
**其中客户端报文携带的 header 字段:**
`Sec-WebSocket-Key` ：必传， 由客户端随机生成的 16 字节值, 然后做 base64 编码, 客户端需要保证该值是足够随机, 不可被预测的 

`Sec-WebSocket-Version` ：  必传， 指示 WebSocket 协议的版本, [RFC 6455](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc6455) 的协议版本为 13, 在 [RFC 6455](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc6455) 的 Draft 阶段已经有针对相应的 WebSocket 实现, 它们当时使用更低的版本号, 若客户端同时支持多个 WebSocket 协议版本, 可以在该字段中以逗号分隔传递支持的版本列表 (按期望使用的程序降序排列), 服务端可从中选取一个支持的协议版本。

`Sec-WebSocket-Protoco`： 可选, 客户端发起握手的时候可以在头部设置该字段, 该字段的值是一系列客户端希望在于服务端交互时使用的子协议 (subprotocol), 多个子协议之间用逗号分隔, 按客户端期望的顺序降序排列, 服务端可以根据客户端提供的子协议列表选择一个或多个子协议
`Sec-WebSocket-Extensions` ： 可选, 客户端在 WebSocket 握手阶段可以在头部设置该字段指示自己希望使用的 WebSocket 协议拓展。

**服务端若支持 WebSocket 协议, 并同意与客户端握手, 则应返回 101 的 HTTP 状态码, 表示同意协议升级, 同时应设置 Upgrade 字段并将值设置为 websocket, 并将 Connection 字段的值设置为 Upgrade, 这些都是与标准 HTTP Upgrade 机制完全相同的, 除了这些以外, 服务端还应设置与 WebSocket 相关的头部字段:**
` Sec-WebSocket-Accept` ： 必传, 客户端发起握手时通过 | Sec-WebSocket-Key | 字段传递了一个将随机生成的 16 字节做 base64 编码后的字符串，服务端若接收握手，则应将该值与 WebSocket 魔数 (Magic Number) "258EAFA5-E914-47DA- 95CA-C5AB0DC85B11" 进行字符串连接, 将得到的字符串做 SHA-1 哈希, 将得到的哈希值再做 base64 编码， 最终的值便是该字段
的值。当客户端收到服务端的握手响应后，会做同样的运算来校验该值是否符合预期。

`Sec-WebSocket-Protocol` ：即对应的服务端支持的子协议

客户端参考代码 ：[nhooyr/websocket/dial.go](https://github.dev/nhooyr/websocket/blob/8dee580a7f74cf1713400307b4eee514b927870f/dial.go#L144)
# Websocket 协议帧
```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

- FIN, 长度为 1 比特， 该标志位用于指示当前的 frame 是消息的最后一个分段，因为 WebSocket 支持将长消息切分为若干个 frame 发送，切分以后，除了最后一个 frame，前面的 frame 的 FIN 字段都为 0， 最后一个 frame 的 FIN 字段为 1，当然， 若消息没有分段，那么一个 frame 便包含了完成的消息，此时其 FIN 字段值为 1。

- RSV 1 ~ 3， 这三个字段为保留字段，只有在 WebSocket 扩展时用，若不启用扩展，则该三个字段应置为 1，若接收方收到 RSV 1 ~ 3 不全为 0 的 frame， 并且双方没有协商使用 WebSocket 协议扩展， 则接收方应立即终止 WebSocket 连接。

- Opcode， 长度为 4 比特， 该字段将指示 frame 的类型，[RFC 6455](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc6455) 定义的 Opcode 共有如下几种:
   - 0x0， 代表当前是一个 continuation frame
   - 0x1，代表当前是一个 text frame
   - 0x2，代表当前是一个 binary frame
   - 0x3 ~ 7，目前保留, 以后将用作更多的非控制类 frame
   - 0x8，代表当前是一个 connection close, 用于关闭 WebSocket 连接
   - 0x9，代表当前是一个 ping frame
   - 0xA， 代表当前是一个 pong frame
   - 0xB ~ F， 目前保留, 以后将用作更多的控制类 frame

- `Mask`，长度为 1 比特, 该字段是一个标志位, 用于指示 frame 的数据 (Payload) 是否使用掩码掩盖, [RFC 6455](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc6455) 规定当且仅当由客户端向服务端发送的 frame, 需要使用掩码覆盖, 掩码覆盖主要为了解决代理缓存污染攻击 (更多细节见 [RFC 6455 Section 10.3](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc6455%23section-10.3))

- `Payload Len`, 以字节为单位指示 frame Payload 的长度, 该字段的长度可变。当 Payload 的实际长度在 [0, 125] 时，则 Payload Len 字段的长度为 7 比特，它的值直接代表了 Payload 的实际长度；当 Payload 的实际长度为 126 时；则 Payload Len 后跟随的 16 位将被解释为 16-bit 的无符号整数， 该整数的值指示 Payload 的实际长度；当 Payload 的实际长度为 127 时，其后的 64 比特将被解释为 64-bit 的无符号整数, 该整数的值指示 Payload 的实际长度。

- `Masking-key`，该字段为可选字段，当 Mask 标志位为 1 时， 代表这是一个掩码覆盖的 frame，此时 Masking-key 字段存在, 其长度为 32 位，[RFC 6455](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc6455) 规定所有由客户端发往服务端的 frame 都必须使用掩码覆盖, 即对于所有由客户端发往服务端的 frame，该字段都必须存在, 该字段的值是由客户端使用熵值足够大的随机数发生器生成。

- `Payload`，该字段的长度是任意的，该字段即为 frame 的数据部分，若通信双方协商使用了 WebSocket 扩展， 则该扩展数据 (Extension data) 也将存放在此处， 扩展数据 + 应用数据, 它们的长度和便为 Payload Len 字段指示的值。

[nhooyr/websocket/frame.go](https://github.dev/nhooyr/websocket/blob/8dee580a7f74cf1713400307b4eee514b927870f/frame.go#L92)
# WebSocket 掩码算法
具体参考：
[https://datatracker.ietf.org/doc/html/rfc6455#section-5.3](https://datatracker.ietf.org/doc/html/rfc6455#section-5.3)
简单说就是 playload 和 masking-key 做异或，masking-key 的长度是 4 字节，所有 playload 的 index 与 4 取模，得到使用 masking-key 的哪个字节和 playload[i] 做异或
 playload[i] ^=  masking-key[i%4]
```
j                   = i MOD 4
transformed-octet-i = original-octet-i XOR masking-key-octet-j
```

# 参考
- [dial.go#L144](https://github.dev/nhooyr/websocket/blob/8dee580a7f74cf1713400307b4eee514b927870f/dial.go#L144)
- [https://zhuanlan.zhihu.com/p/407711596](https://zhuanlan.zhihu.com/p/407711596)
- [https://datatracker.ietf.org/doc/html/rfc6455](https://datatracker.ietf.org/doc/html/rfc6455)


