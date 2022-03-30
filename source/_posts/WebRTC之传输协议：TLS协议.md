title: WebRTC之传输协议初探：TLS协议
date: 2020-11-3 23:10:07
mathjax: true
tags:
- 传输协议
categories:
- 编程
- WebRTC
keywords:
- TLS

---

### 简述

TLS(Transport Layer Security)协议，其前身为SSL(Secure Socket Layer)，是1994年由Netscape公司设计的一套协议，并于1995年发布了3.0版本，而TLS是IETF基于SSL3.0设计的协议，相当于SSL的后续版本。

TLS建立在传输层之上，服务于应用层，旨在于为通信双方提供一条安全通道，主要提供一下三重保障：

- 身份认证：认证通信双方的身份，防止第三方冒充身份参与通信。
- 数据安全：加密通道数据且只有通信双方可以解密，以防窃听。
- 数据完整：提供数据签名和校验机制，一旦数据被篡改，通信双方可立刻发现。

<!-- more -->

### 实现细节

TLS协议分为两层，分别是握手协议层和记录层，且Handshake协议层位于Record协议层之上：

- 握手协议层（Handshake Protocol Layer）
    
    主要用于通信双方的身份认证，以及使用非对称的加密算法协商Record协议中使用的对称密钥。
    
    握手协议层有三个协议，分别是：
    
    - 握手协议（Handshake Protocol）
    - 更换加密规约协议（Change Cipher Spec Protocol）
    - 告警协议（Alert Protocol）
    
    鉴于非对称加密算法的优缺点：
    
    - 优势：在于加解密过程需要分别依赖公钥和私钥，使用公钥加密的数据只有私钥才能解密，所以即使公钥公开或泄漏也能保证加密数据的安全性。
    - 劣势：加解密耗时长、速度慢、效率低，只适合对少量数据进行加密。
    
    非对称加密算法用于握手协议实现对称密钥的分发。
    
- 记录层（Record Layer）
    
    通过握手协议获得对称密钥，使用对称加密算法对应用数据进行加密，并提供数据签名和校验机制，以实现数据的安全传输。
    
    鉴于对称加密算法的特点：
    
    - 优势：加解密耗时短、速度快、效率高，适合对大量数据进行加密。
    - 劣势：加密和解密使用同一个秘钥，秘钥的管理和分发非常困难，难以保障安全。
    
    对称加密算法用于记录层实现应用数据的加解密。
    

### 握手过程

该过程的实现基于Handshake协议，共分两个阶段，分别是明文通信阶段、非对称加密通信阶段。

以下为TLS1.2版本的握手过程。

1. 客户端发送Hello消息：
    - `Client Hello`
        
        客户端的Hello消息，包括以下参数：
        
        - client_version： 客户端支持的TLS版本，客户端会从高到低去尝试填入自己支持的SSL版本。
        - random：客户端生成的随机数A， 用于后续的密钥协商。
        - session_id：本次会话ID。可用于恢复会话，为空则表示没有会话ID。
        - cipher_suites：客户端支持的加密套件列表，其中的加密套接字使用IANA中注册的名称，根据优先级从高到低排列，优先级越高表示客户端越倾向选择该加密套件。
            
            关于加密套接字：使用的是IANA中注册的名称，可在[https://ciphersuite.info/cs/](https://ciphersuite.info/cs/)中查询，IANA 名称由 Protocol，Key Exchange Algorithm，Authentication Algorithm，Encryption Algorithm ，Hash Algorithm 的描述组成。例如，TLS\_ECDHE\_RSA\_WITH\_AES\_128\_GCM\_SHA256 的含义如下：
            
            - Protocol: 安全传输协议，即TLS协议。
            - Key Exchange: 密钥交换算法，即Elliptic Curve Diffie-Hellman Ephemeral (ECDHE)
            - Authentication: 非对称加密算法，即Rivest Shamir Adleman algorithm (RSA)
            - Encryption: 对称加密算法，即Advanced Encryption Standard with 128bit key in Galois/Counter mode (AES 128 GCM)
            - Hash: 摘要算法，即Secure Hash Algorithm 256 (SHA256)
        - compression_methods：客户端支持的压缩算法列表，根据优先级从高到低排列，优先级越高表示客户端越倾向选择该压缩算法。可为空，表示不支持压缩。在TLS1.3版本中已弃用。
        - extensions：客户端所支持的扩展协议簇，如果服务端未回应某个扩展协议表示不使用该扩展协议。
2. 服务端响应客户端的Hello消息（以下消息是通过一次传输传递给客户端）：
    - `Server Hello`
        
        服务端的Hello消息，包括以下参数：
        
        - server_version：服务端支持的TLS版本。
        - random：服务端生成的随机数B，用于后续的密钥协商。
        - session_id：本次会话ID。如果服务端同意客户端重用上次会话则返回与客户端相同的ID，否则返回新的会话ID。
        - cipher_suite：服务端支持的加密套件，且必须选于客户端支持的加密套件列表。
        - compression_method：服务端支持的加密算法，且必须选于客户端支持的压缩算法列表。在TLS1.3版本中已弃用。
        - extensions：服务端支持的扩展协议簇，且必须选于客户端支持的扩展协议簇。
    - `Server Certificate`
        
        服务端证书，一般是X.509证书，用于客户端验证服务端身份和交换密钥，其中的密钥交换算法与ServerHello消息中所选加密算法一致。
        
    - `Server Key Exchange`
        
        该消息用于将服务端的公钥发送给客户端，常用的密钥协商算法有：
        
        - RSA算法：如果采用该算法，可以不发送该消息，因为RSA算法使用的公钥以包含在Server Certificate中。
        - EC Diffie-Hellman算法：可根据对方的公钥和自己的私钥计算共享密钥。
    - `Certificate Request`
        
        在某些安全性要求高的场景，例如银行支付等，不仅需要验证服务端的身份，还需要验证客户端的身份，这时候服务端就会要求客户端提供客户端的身份证书。
        
    - `Server Hello Done`
        
        服务端Hello阶段结束信息。
        
3. 客户端校验服务端身份：
    
    客户端会校验服务端发过来的证书的合法性，包括：
    
    - 证书链是否可信
    - 证书是否被吊销
    - 证书是否处于有效期
    - 证书的域名是否和当前访问的域名匹配
    
    如果发现服务端的证书不合法，客户端可以向服务端发起告警信息。
    
4. 客户端回应服务端：
    - `Certificate`
        
        可选。如果服务端发送了“Certificate Request”，要求校验客户端身份，那么客户端需要回应自己的证书，一般是X.509证书。如果客户端没有合适的证书，直接抛出告警信息让服务端处理（服务端的处理方式通常就是断开TCP连接）。其中的密钥交换算法与ServerHello消息中所选加密算法一致，用于服务端验证客户端身份。
        
    - `Client Key Exchange`
        
        该消息用于将客户端的公钥发送给服务端，其中的信息用于生成最终的对称加密密钥。常用的密钥协商算法有：
        
        - RSA算法
        - EC Diffie-Hellman算法
    - `Certificate Verify`
        
        可选。如果服务端要求客户端要求客户端提供证书，那么在客户端在发送ClientKeyExchange消息后紧接着发送该消息。消息内容是使用客户端的私钥加密的一段基于已经协商的通信信息，服务端可以用客户端的公钥解密验证。
        
    - `Change Cipher Spec`
        
        用于提示服务端在随后的连接中都使用当前已协商好的加密方式和主密钥进行通信。
        
    - `Client Handshake Finished`
        
        表示客户端的握手流程结束。当所有的操作完成后，客户端发送Finish消息。该消息包含了Handshake信息和证书信息的哈希值，用于验证身份校验和密钥交换过程都完成。
        
        Finish消息不要求服务端回复，发送该消息后可立刻对应用数据的加密并传输。
        
5. 服务端回应客户端：
    - `Server Change Cipher Spec`
        
        用于提示客户端在随后的连接中都使用当前当前已协商好的加密方式和主密钥进行通信。
        
    - `Server Handshake Finished`
        
        同客户端的Server Handshake Finished消息类似，表示服务端的握手流程结束。
        

### Record层传输过程

该过程为应用数据传输过程，基于Record协议实现，采样握手过程中已协商好的对称加密算法和对称密钥对数据进行加密和传输。具体过程有待深入研究。

### 密钥协商

握手过程最重要的任务之一就是密钥协商，下面详述密钥协商的过程。

1. 计算Pre-Master Secret
    
    不同的密钥交换算法生成pre-master secret的方式也不同：
    
    - RSA算法：
        
        当客户端在验证服务端的身份证书后，取出包含在其中的服务端公钥，然后产生一个随机数C作为pre-master secret，并使用服务端的公钥对其进行加密，最后通过该消息发送给服务端。当服务端收到该消息后，使用私钥解密出其中的pre-master secret。
        
    - EC-DH 算法：
        
        服务端和客户端通过KeyExchange获得对方用于DH算法的公钥，然后双方使用对方的公钥和自己的私钥，根据特殊的数学特性，计算出一个相同的结果作为共享密钥，即pre-master secret。
        
        在KeyExchange中包含EC-DH具体采用的椭圆曲线算法，以secp256r1为例：
        
        椭圆曲线算法需要一下几个参数：
        
        - 素数p，用于确定有限域的范围
        - 椭圆曲线方程中的a，b参数
        - 用于生成子群的基点G
        - 子群的阶n
        - 子群的辅助因子h
        
        可定义为一个六元组（p,a,b,G,n,h），在[https://www.secg.org/sec2-v2.pdf](https://www.secg.org/sec2-v2.pdf)中可查到secp256r1使用的参数。
        
        假设私钥是D，那么公钥 H = DG，G即子群的基点G
        
        服务端和客户端使用椭圆曲线算法分别计算出自己的密钥对（公钥+私钥），假设私钥分别是：{Ds, Hs} 和  {Dc, Hc}，且满足下面的等式：
        
        ```cpp
        S = Ds * Hc = Ds(DcG) = Dc(DsG) = Dc * Hs
        ```
        
2.  计算 Master Secret
    
    此时，客户端和服务端都拥有相同三个随机数：random_A，random_B和pre-master secret，使用伪随机函数PRF（pseudo random function）生成一串随机数，截取48位作为主密钥Master secret。比如：
    
    ```jsx
    master_secret = PRF(pre_master_secret, "master secret", Random_A + Random_B)[0..47]
    ```
    
    注：TLS中PRF本质上的一个扩展后的Hash函数，具体使用的Hash算法取决于协商的密钥套件和TLS版本，对于关系如下：
    
    - prf\_tls10：TLS 1.0 和 TLS 1.1 协议，PRF 算法是结合 MD5 和 SHA\_1 算法
    - prf\_tls12\_sha256：TLS 1.2 协议，默认是 SHA_\256 算法(这是能满足最低安全的算法)
    - prf\_tls12\_sha384：TLS 1.2 协议，如果加密套件指定的 HMAC 算法安全级别高于 SHA\_256，则采用SHA_384 算法
3. 导出Record层的对称密钥
    
    根据握手过程中协商好的加密套件，从中可知具体的对称加密算法和校验算法，使用对应的PRF和Master Secret导出对应的密钥。
    
    以TLS\_ECDHE\_RSA\_WITH\_AES\128\_GCM\_SHA256为例：
    
    - 对称加密算法：AES-128-GCM
    - 校验算法：SHA256
    
    使用PRF和Master Secret生成密钥块key block：
    
    ```jsx
    key_block = PRF(master_secret, "EXTRACTOR", client_random + server_random)[length]
    ```
    
    截取出6个参数作为通信密钥，分别是：
    
    - 客户端写MAC密钥
    - 服务端写MAC密钥
    - 客户端写密钥
    - 服务端写密钥
    - 客户端IV
    - 服务端IV
    
    其中，客户端写、服务端读用客户端写密钥；服务端写、客户端读用服务端写密钥。另外对称加密算法是AES-GCM（AES算法除ECB模式外都需要IV），所以需要客户端IV和服务端IV。
    

### 结语

由于TLS协议主要是提供加密功能以保障通道数据的安全性，因此要求底层协议提供一个可靠且有序的通信过程，这也是TLS常与TCP协议组合使用的原因。如果需要为UDP协议提供安全保障，则可以使用DTLS协议。

### 参考资料

[rfc8446](https://datatracker.ietf.org/doc/html/rfc8446)

[https://datatracker.ietf.org/doc/html/rfc5705#section-4](https://datatracker.ietf.org/doc/html/rfc5705#section-4)

[SSL/TLS发展历史和SSLv3.0协议详解 | 文章 | BEWINDOWEB](http://www.bewindoweb.com/271.html)

[ECC椭圆曲线加密算法：ECDH 和 ECDSA](https://zhuanlan.zhihu.com/p/66794410)

[TLS1.2 PreMasterSecret And MasterSecret | 老青菜](https://laoqingcai.com/tls1.2-premasterkey/)

[HTTPS 温故知新（五） -- TLS 中的密钥计算](https://halfrost.com/https-key-cipher/)