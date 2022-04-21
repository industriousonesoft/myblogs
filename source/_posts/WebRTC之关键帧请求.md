title: WebRTC之关键帧请求
date: 2021-4-22 22:10:07
mathjax: false
tags:
- 拥塞控制
categories:
- 编程
- WebRTC
keywords:
- PLI
- FIR

---

### 简述

所谓关键帧，就是不需要参考其他视频帧，可单独解码的帧。比如H264中的I帧。在WebRTC中有两种关键帧请求方式，分别是：

- PLI：Picture Loss Indication
- FIR：Full Intra Request

二者的主要区别在于报文结构和使用场景不同，但实际上在发送端处理两种请求的逻辑很可能是一样的，比如，WebRTC中的实现就是如此。

<!-- more -->

### 报文结构

在WebRTC中，PLI和FIR均被封装在RTCP反馈消息包中，详解[RFC4585](https://datatracker.ietf.org/doc/html/rfc4585#section-6.1)，其报文格式如下：

```cpp
  // RFC 4585: Feedback format.
  // Common packet format:

   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |V=2|P|   FMT   |       PT      |          length               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                  SSRC of packet sender                        |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |             SSRC of media source (unused) = 0                 |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  :            Feedback Control Information (FCI)                 :
  :                                                               :

  // FMT: Feedback message type, 5 bits
  // PT: Payload type, 8 bits
```

RTCP反馈消息包根据PT可分为两类：

- RTPFB：Transport layer FB message, PT=205
- PSFB：Payload-specific FB message, PT=206

PLI和FIR均属于PSFB，且根据FMT字段进行区分：

- PLI：FMT=1
    
    在PLI报文中，FCI部分为空，故其报文结构就是RTCP反馈消息的报文。
    
- FIR：FMT=4
    
    在FIR报文中，可能会包含多个FCI，且FCI包括两个字段，其结构如下：
    
    ```cpp
       // FCI:
       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                              SSRC                             |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      | Seq nr.       |    Reserved = 0                               |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    
    // SSRC：指定发送端响应该关键帧请求的媒体流
    // Seq nr：当前关键帧请求对应的序列号
    ```
    

NOTE：由PLI和FIR报文结构的差异可知，通过PLI发送的关键帧请求会作用于发送端所有的媒体流，而通过FIR发送的关键帧请求可指定发送端响应该请求的媒体流。

### 使用场景

PLI和FIR设计初衷就是为了应对不同的使用场景，尽管二者发送的关键帧请求在发送端有可能被一视同仁。

- 丢包场景
    
    PLI，Picture Loss Indication，即丢包标识，顾名思义就是用于丢包场景请求关键帧。常见的丢包场景如下：
    
    - 接收端检测到丢包过多
        
        当丢包率过高时，重传会导致延时过大，此时可通过关键帧及时刷新画面。由于关键字可单独解码，故解码时不会出现花屏，但由于之前的视频帧都被丢弃，如果该关键帧前后的画面变化过大，渲染时会出现卡顿或跳帧的情况。
        
    - 发送端无丢失包的缓存
        
        发送端会以滑动窗口的方式缓存近期发送的包，如果某个丢失包过旧，即不在滑动窗口范围内，会导致发送端无法重传该包。
        
    - H264解码时无SPS和PPS信息
        
        当H264解码端收到的IDR帧包未包含SPS或PPS信息，会导致解码失败，此时可发送PLI请求。
        
    - 解码器获取帧数超时
    - 解码器请求关键帧或解码失败
- 非丢包场景
    
    FIR请求常用于非丢包的场景，编码端在收到FIR的关键帧请求时，通常会发送一个IDR帧用于解码端的及时刷新，故FIR请求又被称为“即时解码刷新请求（Instantaneous Decoder Refresh Request）”或“视频快速刷新请求（Video Fast Update Request）”。
    
    FIR请求常见的使用场景如下：
    
    - 视频流切换
        
        当解码器需要切换到其它的视频流时，可通过FIR请求从编码端获取一个IDR帧，及时刷新解码器。
        
    - 新参与者入会
        
        在视频会议中，当某个新用户加入时，接收端收到的不一定是关键帧，这会导致新用户不能及时解码。此时可通过发送FIR请求从编码端获取一个IDR帧，及时刷新解码器。
        

### 参考资料

[Media Communication](https://webrtcforthecurious.com/docs/06-media-communication/#full-intra-frame-request-fir-and-picture-loss-indication-pli)

[RFC 5104 - Codec Control Messages in the RTP Audio-Visual Profile with Feedback (AVPF)](https://datatracker.ietf.org/doc/html/rfc5104#section-3.5.1)

[RFC 4585 - Extended RTP Profile for Real-time Transport Control Protocol (RTCP)-Based Feedback (RTP/AVPF)](https://datatracker.ietf.org/doc/html/rfc4585#section-6.3.1)