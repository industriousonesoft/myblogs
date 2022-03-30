title: WebRTC之传输协议初探：DTLS协议
date: 2020-11-10 23:10:07
mathjax: true
tags:
- 传输协议
categories:
- 编程
- WebRTC
keywords:
- DTLS

---

### 简述

由于TLS协议主要是提供加密功能以保障通道数据的安全性，因此要求底层协议提供一个可靠且有序的通信过程，一旦数据包出现丢包或乱序的情况则会断开连接，所以UDP协议是不能和TLS协议组合使用。

现实情况是UDP协议常用于媒体数据传输，因此为了保障媒体传输的安全性，引入了DTLS协议来对UDP通信过程进行加密。DTLS是在TLS的基础为UDP定制和改进的安全传输协议，在设计上尽可能复用TLS现有的结构。

DTLS和TLS在版本上的对应关系：

- DTLS1.0 → TLS1.1
- DTLS1.2 → TLS1.2
- DTLS1.3 → TLS1.3

<!-- more -->

### 握手过程

由于DTLS和TLS的握手过程大体上是一致的，详见 [TLS协议详解](/2020/11/03/WebRTC之传输协议：TLS协议)，此处只列出DTLS协议改进的地方：

1. 结构上，在Handshake和ClientHello消息中新增字段和新增HelloVerifyRequest消息：
    - Handshake消息中新添了三个字段：message\_seq、fragment\_offset、fragment\_length，结构所示：
        
        ```cpp
        struct {
             HandshakeType msg_type;
             uint24 length;
             uint16 message_seq;                               // New field
             uint24 fragment_offset;                           // New field
             uint24 fragment_length;                           // New field
             select (HandshakeType) {
               case hello_request: HelloRequest;
               case client_hello:  ClientHello;
               case hello_verify_request: HelloVerifyRequest;  // New type
               case server_hello:  ServerHello;
               case certificate:Certificate;
               case server_key_exchange: ServerKeyExchange;
               case certificate_request: CertificateRequest;
               case server_hello_done:ServerHelloDone;
               case certificate_verify:  CertificateVerify;
               case client_key_exchange: ClientKeyExchange;
               case finished: Finished;
             } body;
         } Handshake;
        ```
        
    - ClientHello消息新添了Cookie字段，结构如下：
        
        ```cpp
        struct {
             ProtocolVersion client_version;
             Random random;
             SessionID session_id;
             opaque cookie<0..2^8-1>;                             // New field
             CipherSuite cipher_suites<2..2^16-1>;
             CompressionMethod compression_methods<1..2^8-1>;
         } ClientHello;
        ```
        
    - 新增HelloVerifyRequest消息，结构如下：
        
        ```cpp
        struct {
             ProtocolVersion server_version;
             opaque cookie<0..2^8-1>;
        } HelloVerifyRequest;
        ```
        
2. 功能上，增加了一些防护机制，解决重放、丢包和乱序问题:
    - 重传机制解决丢包问题
        
        当客户端和服务端每次发送消息后会启动一个定时器等待远端相应的消息，一旦等待超时就理解为发送的消息或等待的消息丢失，然后重发一次消息。当重传次数超过上限时仍未收到回复消息则断开连接。
        
    - 序列号+分片机制解决乱序问题
        
        TCP是面向字节流，且内置数据包的分拆和组装功能，而UDP则是面向报文，本身不支持分片和重组。所以一旦UDP报文大于MTU，则由IP层进行分片和重组。但是一旦出现丢包则重组失败，进而导致整个UDP报文丢失。
        
        DTLS新增了分片和重组机制，因此不再会触发IP层的分片和重组机制。具体做法是在Handshake消息中新增了fragment\_offset和fragment\_length，分别表示当前分片在报文中的偏移量和报文的总长度。此外。DTLS还支持丢包重传，所以即便出现丢包也能进行恢复，提高重组的成功率。
        
        当报文重组完成后会进入客户端或服务端的缓存队列，该队列用于缓存提前到达的消息，然后根据序列号对缓存消息进行排序，以解决报文乱序问题。
        
    - 消息重放检测+Cookie机制
        
        TCP中通过SYN Cookie机制来实现消息重放检测，以防范DoS攻击，TLS在1.2及之前的版本都没有相应的防范机制，直到TLS1.3才通过新增HelloRetryRequest和Cookie来解决DoS攻击。而DTLS在1.0版本就新增了HelloVerifyRequest和Cookie，用于服务端的二次校验，以防范DoS攻击。具体实现方式如下：
        
        - 当客户端首次给服务端发送 Client Hello 时，服务端只会生成一个 Cookie 并通过 HelloVerifyRequest 发送给客户端，不会执行分配缓冲区等操作，直到收到带上相同 Cookie 的 Client Hello 才会继续握手，可以使得伪造 IP 的攻击难以实现。这也只是一种时间和空间上的权衡。如果攻击者发送大量的ACK包过来，那么服务端需要花费大量的CPU时间用于在计算Cookie，最终导致正常的逻辑无法被执行。
        - HelloVerifyRequest 足够小，即使服务端被攻击者当枪使来攻击其他机器，也不会造成大量数据发送。

### 结语

DTLS协议在TLS协议的基础上提供了可靠性和有序性保障，可实现基于UDP协议的安全传输。鉴于DTLS协议中防护机制的实现相对比较简单，不适合用于传输音视频数据，所以一般用于传输文本数据，比如可作为SCTP协议的底层传输协议。

### 参考资料

[RFC 6347 - Datagram Transport Layer Security Version 1.2](https://datatracker.ietf.org/doc/html/rfc6347)

[一文读懂 DTLS 协议_mb6004f7ec10a08的技术博客_51CTO博客_DTLS协议详解](https://blog.51cto.com/u_15087084/2598254)

[详解 WebRTC 传输安全机制：一文读懂 DTLS 协议](https://xie.infoq.cn/article/182f59ec77450fde2f87e2354)

[SYN Cookie的原理和实现_zhangskd的专栏-CSDN博客_syncookie](https://blog.csdn.net/zhangskd/article/details/16986931)

[DoS攻击手法与解决办法](https://zhuanlan.zhihu.com/p/462994526)