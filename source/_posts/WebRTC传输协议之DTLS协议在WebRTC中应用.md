title: WebRTC之传输协议初探：DTLS协议在WebRTC中应用
date: 2020-11-16 23:10:07
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

DTLS在WebRTC中主要提供两个功能：

1. 协商SRTP加密密钥
    
    虽然DTLS协议提供了解决丢包和乱序的机制，但是具体实现方案比较简单，对于音视频这类对丢包和乱序更为敏感的数据包有点力不从心。因此WebRTC中RTP（Real-time Transport Protocol）协议作为音视频包的传输协议。而SRTP（Secure Real-time Transport Protocol）在RTP协议之上，为RTP包提供了加密、消息认证和完整性以及重放攻击等保护。
    
    SRTP作为安全协议，采用的是对称加密算法，而密钥协商则是通过DTLS协议，具体协商过程留待下文。
    
2. 为SCTP协议提供加密通道
    
    SCTP是一种在网络连接两端之间同时传输多个数据流的协议，提供的服务与UDP和TCP类似，同样SCTP本身不提供加密功能。因此在WebRTC中使用DTLS作为SCTP的底层协议，为SCTP中的消息提供加密等一系列安全保护。
    
<!-- more -->

### SCTP密钥协商过程

由于SCTP密钥协议依赖于DTLS协议，因此其过程包含了DTLS的握手过程。

DTLS同TLS一样，握手过程中有client和server的两个角色。但是在WebRTC中没有这样的角色划分，因此在使用DTLS之前需要先进行角色协商。

此外，DTLS的握手过程还可能需要进行身份验证，因此在还需要进行证书交换。

注：角色协商和证书交换的可靠性在于SPD是通过安全的信令通道完成交换的，比如信令通道是是基于Https协议。

1. 角色协商
    
    在WebRTC中DTLS协商角色是通过SDP（Session Description Protocol）协议实现的，在SDP的描述如下：
    
    ```cpp
    a=setup:actpass
    ```
    
    setup属性有三个值，分别是：
    
    - setup:active，表示 client，主动发起协商
    - setup:passive, 表示 server，等待发起协商
    - setup:actpass, 表示既可以是 client也可以是server
    
    详见 [rfc4145](https://datatracker.ietf.org/doc/html/rfc4145#section-4.1)
    
2. 证书交换
    
    在WebRTC中，通信的双方通常无法获得由知名根证书颁发机构（CA）签名的身份验证证书，因此自签名证书通常的唯一的选择。WebRTC采用机制是：将自签名证书的哈希值，即证书指纹，通过SDP传输给对方。因为在角色协商之前任一方都可能是sever，所以双方都必须获得对方的证书指纹，用于后续DTLS的身份验证。如果DTLS中对方提供的证书与SDP中的指纹匹配，则可以表示对方是可信任的。
    
3. DTLS协议握手
    
    DTLS协议握手过程不再赘述，详见 [DTLS协议初探](/2020/11/10/WebRTC之传输协议：DTLS协议) 
    
    但是，由于WebRTC中DTLS需要支持SRTP密钥的协商，因此在ClientHello和ServerHello中加入了use_srtp扩展，用于协商SRTP使用的加密套件。
    
4. 导出STP密钥
    1. 使用DTLS协商后导出的master\_secret，client\_random，server\_random生成key block:
        
        ```cpp
         key_block = PRF(master_secret, "EXTRACTOR-dtls_srtp", client_random + server_random)[length]
        ```
        
    2. 计算需要从SRTP的master\_secret的字节数：
        
        在[rfc5764](https://datatracker.ietf.org/doc/html/rfc5764#section-4.2)中描述了计算公式：
        
        ```cpp
        // 2 *（主密钥长度 + 主密钥盐值长度）/ 8 
        srtp_master_secret_bytes = 2 * (SRTPSecurityParams.master_key_len + 
        														    SRTPSecurityParams.master_salt_len) bytes of data 
        ```
        
        use_srtp扩展可知SRTP协商使用的加密套件，常用的SRTP加密套件及其描述如下:
        
        详见：[rfc5764](https://datatracker.ietf.org/doc/html/rfc5764#section-4.1.2)
        
        ```cpp
        SRTP_AES128_CM_HMAC_SHA1_80
                 cipher: AES_128_CM
                 cipher_key_length: 128
                 cipher_salt_length: 112
                 maximum_lifetime: 2^31
                 auth_function: HMAC-SHA1
                 auth_key_length: 160
                 auth_tag_length: 80
        SRTP_AES128_CM_HMAC_SHA1_32
                 cipher: AES_128_CM
                 cipher_key_length: 128
                 cipher_salt_length: 112
                 maximum_lifetime: 2^31
                 auth_function: HMAC-SHA1
                 auth_key_length: 160
                 auth_tag_length: 32
                 RTCP auth_tag_length: 80
        SRTP_NULL_HMAC_SHA1_80
                 cipher: NULL
                 cipher_key_length: 0
                 cipher_salt_length: 0
                 maximum_lifetime: 2^31
                 auth_function: HMAC-SHA1
                 auth_key_length: 160
                 auth_tag_length: 80
        SRTP_NULL_HMAC_SHA1_32
                 cipher: NULL
                 cipher_key_length: 0
                 cipher_salt_length: 0
                 maximum_lifetime: 2^31
                 auth_function: HMAC-SHA1
                 auth_key_length: 160
                 auth_tag_length: 32
                 RTCP auth_tag_length: 80
        ```
        
        假设协商的加密套件是SRTP\_AES128\_CM\_HMAC\_SHA1\_80，那么master secret的字节数为：
        
        ```cpp
        srtp_master_secret_bytes = 2 * (cipher_key_length + cipher_salt_length) / 8
        											   = 2 * (128 + 112) / 8
        											   = 240 / 4 = 60 // 字节
        ```
        
    3. 导出SRTP密钥
        
        获得master secret后，按顺序分别逐段截取出：客户端master\_key，服务端master\_key，客户端master\_salt，服务端master\_salt，详见：[rfc5764](https://datatracker.ietf.org/doc/html/rfc5764#section-4.1.2)
        
        ```cpp
        // 客户端master_key，长度为16字节
        client_write_SRTP_master_key[SRTPSecurityParams.master_key_len];
        // 服务端master_key，长度为16字节
        server_write_SRTP_master_key[SRTPSecurityParams.master_key_len];
        // 客户端master_salt，长度为14字节
        client_write_SRTP_master_salt[SRTPSecurityParams.master_salt_len];
        // 客户端master_salt，长度为14字节
        server_write_SRTP_master_salt[SRTPSecurityParams.master_salt_len];
        ```
        
        随后，使用master\_key和master\_salt作为SRTP或SRTCP的KDF（Key Derivation Function）的参数分别生成用于加密和签名密钥。如下图所示：
        
        ```cpp
        TLS master
             secret   label
              |         |
              v         v
           +---------------+
           | TLS extractor |
           +---------------+
                  |                                         +------+   SRTP
                  +-> client_write_SRTP_master_key ----+--->| SRTP |-> client
                  |                                    | +->| KDF  |   write
                  |                                    | |  +------+   keys
                  |                                    | |
                  +-> server_write_SRTP_master_key --  | |  +------+   SRTCP
                  |                                  \ \--->|SRTCP |-> client
                  |                                   \  +->| KDF  |   write
                  |                                    | |  +------+   keys
                  +-> client_write_SRTP_master_salt ---|-+
                  |                                    |
                  |                                    |    +------+   SRTP
                  |                                    +--->| SRTP |-> server
                  +-> server_write_SRTP_master_salt -+-|--->| KDF  |   write
                                                     | |    +------+   keys
                                                     | |
                                                     | |    +------+   SRTCP
                                                     | +--->|SRTCP |-> server
                                                     +----->| KDF  |   write
                                                            +------+   keys
        ```
        
        至此，完成基于TLS协商的master\_secret导出SRTP密钥的全过程。
        
### 参考资料
[详解 WebRTC 传输安全机制：一文读懂 DTLS 协议](https://xie.infoq.cn/article/182f59ec77450fde2f87e2354)

[RFC 5764 - Datagram Transport Layer Security (DTLS) Extension to Establish Keys for the Secure Real-time Transport Protocol (SRTP)](https://datatracker.ietf.org/doc/html/rfc5764)