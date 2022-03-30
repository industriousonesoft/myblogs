title: WebRTC之传输协议初探：SRTP协议
date: 2020-11-20 23:10:07
mathjax: true
tags:
- 传输协议
categories:
- 编程
- WebRTC
keywords:
- SRTP

---

### 简述

SRTP协议，全称Secure Real-time Transport Protocol。顾名思义，SRTP就是在RTP协议的基础上提供了安全保障，主要包括数据加密、消息认证和重放保护。

SRTP提供了一套框架用于加密和认证RTP和RTCP数据流，且内置一系列预定义的加密套件。当然也可以使用自定义加密套件。

SRTP协议建立于在RTP协议之上，在RTP/RTCP包的基础上通过增加额外的空间和计算量来实现安全保障。因此SRTP增加的额外开销越小，对RTP传输影响也越小。而SRTP额外开销的大小主要取决于加密套件的选择，SRTP中预定义的加密套件是一个不错的选择，详见 [RFC 3711](https://datatracker.ietf.org/doc/html/rfc3711#section-2)。

<!-- more -->

### 实现细节

由于RTP和RTCP包在结构、功能以及传输上都存在差异，因此SRTP对二者的具体实现也存在差异。以下是SRTP基于预定义的加密套件的实现细节。

#### 密钥上下文

在RTP/RTCP协议中将通信参与者称之为一个RTP session，常见的会话类型有video RTP session、audio RTP session和text RTP session。

每一个RTP session可以包含多个stream，每个stream通过SSRC标识。RTP session对其中的每个stream的处理过程都是对等的，即发送或接收每个stream中的RTP/RTCP包。因此RTP session用使用一个二元组<destination network address, destination transport port number>来标识即可。

SRTP/SRTCP协议中新增了SRTP stream，一个三元组<SSRC, destination network address, destination transport port number>，用于标识RTP session下的每个stream。这样可以给每个SRTP stream提供不同的加密处理，具体实现原理是基于每个stream的SSRC生成不同的初始化向量（IV）：

```cpp
IV = (k_s * 2^16) XOR (SSRC * 2^64) XOR (i * 2^16)
```

- 两种密钥类型
    
    SRTP中有两类密钥，分别是：
    
    - master key
        
        master key是一串随机生产的字符串，一般由密钥协商协议提供，比如TLS。
        
    - session keys
        
        session keys中包含加密密钥、签名密钥和盐值，用于数据加密和消息认证。
        
        SRTP和SRTCP共用一个master key和各自的KDF（Key Derivation Function）生成对应的session keys，如下图所示：
        
        ```cpp
        				  packet index ---+
                                  |
                                  v
        +-----------+ master  +--------+ session encr_key
        | ext       | key     |        |---------->
        | key mgmt  |-------->|  key   | session auth_key
        | (optional |         | deriv  |---------->
        | rekey)    |-------->|        | session salt_key
        |           | master  |        |---------->
        +-----------+ salt    +--------+
        ```
        
        Session keys的导出函数除了master key和master salt之外还需要以下参数：
        
        - key_label
            
            用于标识key的类型，取值范围为0x00~0x05的含义如下：
            
            ```cpp
            enum KeyLabel : int {
            	label_rtp_encryption = 0x00,  // 导出RTP加密密钥
            	label_rtp_msg_auth = 0x01,    // 导出RTP校验密钥
            	label_rtp_salt = 0x02,        // 导出RTP盐值
              label_rtcp_encryption = 0x03, // 导出RTCP加密密钥
              label_rtcp_msg_auth = 0x04,   // 导出RTCP校验密钥
            	label_rtcp_salt = 0x05,       // 导出RTCP盐值
            }
            ```
            
        - packet_index
            
            SRTP/SRTCP的包序号，注意不是RTP/RTCP包的序号。为了防止重放攻击，SRTP和SRTCP都有一套自己的包管理方式，详见下文。
            
        - key_derivation_rate
            
            表示key的导出速率，取值范围是{1,2,4,...,2^24}。除了初始时导出一次，之后的导出条件需要同时满足r > 0，且packet_index mod r = 0，其中r的计算方式如下：
            
            ```cpp
            r = packet_index / key_derivation_rate;
            ```
            
            新导出的key值会替换原有的session keys用于数据加密和签名。
            
        
        Session keys导出函数调用过程的伪代码如下：
        
        ```cpp
        // 计算r值
        let r = packet_index / key_derivation_rate
        // 计算key_id，由lable和r值组成，label位于高字节，r位于低字节
        let key_id = <label> || r
        // 计算x
        let x = key_id XOR master_salt
        // 调用导出函数kDF生成不同的key值
        let key = KDF(master_key, x)
        ```
        
- 预定义加密套件
    
    SRTP协议中预定义了一些加密套件，包括加密和校验算法以及密钥导出函数等，这些加密套件所需的额外空间和计算都比较小，因此推荐使用。此外也可以自定义加密套件。
    
    其中优先预定义的加密套件必须在SRTP协议的实现中强制实现，如下所示：
    
    ```cpp
    													必须实现            可选            默认
    											mandatory-to-impl.   optional       default
    
       encryption            AES-CM, NULL         AES-f8       AES-CM
       message integrity     HMAC-SHA1              -          HMAC-SHA1
       key derivation (PRF)  AES-CM                 -          AES-CM
    ```
    

#### SRTP包

- 包结构

```cpp
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+<+
     |V=2|P|X|  CC   |M|     PT      |       sequence number         | |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
     |                           timestamp                           | |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
     |           synchronization source (SSRC) identifier            | |
     +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+ |
     |            contributing source (CSRC) identifiers             | |
     |                               ....                            | |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
     |                   RTP extension (OPTIONAL)                    | |
   +>+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
   | |                          payload  ...                         | |
   | |                               +-------------------------------+ |
   | |                               | RTP padding   | RTP pad count | |
   +>+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+<+
   | ~                     SRTP MKI (OPTIONAL)                       ~ |
   | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
   | :                 authentication tag (RECOMMENDED)              : |
   | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
   |                                                                   |
   +- Encrypted Portion*                      Authenticated Portion ---+
```

SRTP包主要包括两个部分，分别是：

- 加密部分：Encrypted Portion
    
    该部分在RTP包负载（payload + padding）的基础上新增了MKI字段和authentication tag字段：
    
    - MKI (Master Key Identifier)：可变长度，可选字段
        
        该字段最大长度为32比特。该字段表示一个master key，可用于后续生产加密和认证的密钥。
        
    - Authentication tag：可变长度，推荐使用
        
        该字段用于存储校验值，其最大长度为32比特。
        
    
    通常情况下SRTP只需要对RTP负载进行加密，但是也可以对RTP header extensions进行加密，详见 RFC 6904。
    
- 校验部分：Authenticated Portion
    
    该部分由RTP的header和Encrypted Portion组成，即RTP包和SRTP新增的字段。值得注意的是完整性校验只针对RTP包本身（header、payload和padding），不包括MKI字段。
    
- SRTP包序号管理
    
    RTP包使用一个16bit的字段来表示序号，其最大值为2^16(65535)，能表达范围很有限，一旦超过最大值便会循环使用之前的序号。因此当收到一个与之前的包列号相同的包时，RTP协议没有办法判断其是一个新包还是之前的包重放了一次，这也是重放攻击的依据所在。
    
    鉴于RTP包序号的局限性，SRTP包序号在RTP包序号的基础上引入回绕技术以表示每个SRTP包的绝对下标。所谓回绕就是RTP包的序号超过最大值65535后重置为0继续使用，SRTP使用ROC来记录RTP包序号的回绕次数，发送端和接收端的具体实现有差异，如下所示：
    
    - 发送端
        
        由于发送端不存在乱序和丢包的问题，因此只需维护ROC即可，即当RTP包的序号出现回绕时ROC加1：
        
        ```cpp
        // ROC：32-bit unsigned，初用于表示RTP包的回绕次数，初始值为0
        // SEQ：16-bit unsigned，RTP包的序号，初始值任意
        // i：48-bit unsigned，高32位为ROC，低16位为SEQ，表示SRTP包的序号
        i = 2^16 * ROC + SEQ
        ```
        
    - 接收端
        
        对于接收端，考虑到丢包和乱序的因素，除了维护ROC，还需记录当前已收到的最大的RTP包序号，记为s_l，初始值为第一个RTP包序号。当收到一个新包时，接收端需要推算出当前包所对应的SRTP的序号，具体实现如下：
        
        ```cpp
        // v：表示当前包可能的回绕次数，取值范围{ ROC-1, ROC, ROC+1 }
        i = 2^16 * v + SEQ
        // s：表示当前已收到的最大的SRTP包序号
        s = 2^16*ROC + s_l
        ```
        
        v分别取ROC-1、ROC和ROC+1计算出i值，再与s进行比较，假定最接近s的值为新包的SRTP序号。具体分三种情况：
        
        1. v = ROC - 1，表示乱序包，ROC和s_l都不更新
        2. v = ROC，表示正常包，s_l = max(SEQ, s_l)
        3. v = ROC + 1，表示正常包但出现回绕，ROC += 1，s_l = SEQ
- 防重放攻击
    
    所谓重复攻击是攻击者将截获的SRTP/SRTCP包保存下来，然后再重新发送出去，使接收端就会大量出现重复的包，以达到攻击的效果。而防重发攻击的目的就是用最高效的方式识别出重复包并将其丢弃，只处理为接收过的包。
    
    此外，防重放攻击的有效性是基于包的完整性，即提供SRTP协议提供了消息认证，否则攻击者可以任意修改SRTP包的内容，包括序号，导致接收端无法判断被修改过的包是否为重发包。
    
    具体实现方式：在每个SRTP包的接收端会维护一个重发列表replay list，用于存储已收到并校验的SRTP包序号。出于性能考虑不会将所有已接收的SRTP包序号（48bits）都存储起来，而是引入一个滑动窗口SRTP_WINDOW_SIZE来推断新收到的包是否为重发包。SRTP_WINDOW_SIZE是一个定值，最小值要求为64，用户可根据需求自定义，比如WebRTC中SRTP_WINDOW_SIZE的默认值为1024。
    
    SRTP_WINDOW_SIZE将replay list划分为了三个区域，分别表示重发区，待定区和新包区：
    
    ```cpp
         重发区        |        待定区            |   新包区
     --replay packet--|---check replay failed---|--new packet--
                      |-----SRTP_WINDOW_SIZE----|
    ```
    
    此外，接收端还需要记录当前接收的最大的SRTP包序号，记为last_packet_index。当接收到新的SRTP包时先推算出其序号，记为packet_index，用于确定新包在replay list中的位置，具体过程如下：
    
    1. 计算差值delta
        
        delta = packet_index - last_packet_index
        
    2. 如果delta > 0，表示收到了新包，并更新last_packet_index和replay list。
    3. 如果delta < -SRTP_WINDOW_SIZE，表示收到重发包，丢弃。
    4. 如果-SRTP_WINDOW_SIZE < delta < 0，表示收到重发窗口之内的包，如果能在replay list中找到相同序号的包则表示收到重发包，否则视为乱序包并接收。

#### SRTCP包

- 包结构
    
    ```cpp
    			0                   1                   2                   3
          0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+<+
         |V=2|P|    RC   |   PT=SR or RR   |             length          | |
         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
         |                         SSRC of sender                        | |
       +>+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+ |
       | ~                          sender info                          ~ |
       | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
       | ~                         report block 1                        ~ |
       | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
       | ~                         report block 2                        ~ |
       | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
       | ~                              ...                              ~ |
       | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
       | |V=2|P|    SC   |  PT=SDES=202  |             length            | |
       | +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+ |
       | |                          SSRC/CSRC_1                          | |
       | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
       | ~                           SDES items                          ~ |
       | +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+ |
       | ~                              ...                              ~ |
       +>+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+ |
       | |E|                         SRTCP index                         | |
       | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+<+
       | ~                     SRTCP MKI (OPTIONAL)                      ~ |
       | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
       | :                     authentication tag                        : |
       | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
       |                                                                   |
       +-- Encrypted Portion                    Authenticated Portion -----+
    ```
    
    SRTCP包主要包括两大部分，分别是：
    
    - 加密部分：Encrypted Portion
        
        该部分在RTCP包负载的基础上新增了四个字段，分别是：
        
        - E-flag：1 bits，必需值。
            
            该字段用于标识SRTCP包是否加密。
            
        - SRTCP index：31 bits，必需值。
            
            该字段表示SRTCP包的序号。不同于SRTP包序号使用回绕技术的处理方式，SRTCP包序号只是简单的将RTCP包序号的长度从16比特扩展为32比特，即同样会出现序号回绕的情况，只是间隔时间更长。
            
        - MKI：可变长度，可选值。
            
            该字段的作用同SRTP中的MKI字段一样。
            
        - authentication tag：可变长度，必须值。
            
            该字段的作用同SRTP中的MKI字段一样，只是在SRTCP包要求必须支持完整性校验。原因在于SRTCP包中携带的是控制RTP包发送和接收端反馈等重要信息，决定着整个传输的效果，所以需要确保其数据是完整的。
            
    - 校验部分：Authenticated Portion
        
        该部分由RTCP的header和Encrypted Portion组成，即RTCP包和SRTCP新增的字段。值得注意的是完整性校验只针对RTCP包本身（header、payload），不包括MKI字段。
        
- 防重放攻击
    
    SRTCP防重放攻击的实现方式与SRTP基本相同，不同的地方在于SRTCP使用的滑动窗口大小更小，比如在libsrtp库中的值为128。
    

### 总结

SRTP/SRTCP协议主要是为RTP/RTCP协议提供数据安全保障，即数据加密、数据校验和防重放攻击。从其实现细节可知SRTP/SRTCP并不保证数据的有序性和可靠性，即不提供丢包和乱序包的保障。

### 参考资料

[RFC 3711 - The Secure Real-time Transport Protocol (SRTP)](https://datatracker.ietf.org/doc/html/rfc3711)

[RFC 5764 - Datagram Transport Layer Security (DTLS) Extension to Establish Keys for the Secure Real-time Transport Protocol (SRTP)](https://datatracker.ietf.org/doc/html/rfc5764)

[WebRTC 传输安全机制：深入显出 SRTP 协议](https://segmentfault.com/a/1190000040211375)