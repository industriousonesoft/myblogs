title: WebRTC之抖动估计
date: 2021-4-5 23:10:07
mathjax: true
tags:
- 拥塞控制
categories:
- 编程
- WebRTC
keywords:
- 抖动
- jitter

---

### 时延和抖动

时延和抖动是两个不同但又相互关联的概念。时延是网络通信中的一个重要指标，用于衡量数据包从一个端点传送到另外一个端点所需的时间。

在网络通信中，连续发送的数据包即便使用相同的路径也会产生不同的时延。原因在于，路由器一次只能处理一个数据包。当数据包抵达的速度大于路由器处理数据包的速度，会被暂时放入路由器的网络缓冲区，等待路由器的处理和转发，从而增加了数据包的传输时延。

抖动就是用于表示数据包之间传输时延的不一致性，即用来衡量数据包传输时延的变化程度。

导致抖动产生的原因有很多，比如网路拥塞，网路错误，丢包等。

<!-- more -->

### 抖动缓冲区

抖动的幅度过大和过于频繁会影响用户体验，比如抖动会使接收端接受到的帧率变低，进而导致后续的解码和播放出现卡顿。因此，常用的作法是在接收端引入抖动缓冲区，用于缓存并处理一段时间内的数据包，然后再把累积的数据包以均匀的间隔传送给后续操作。

使用抖动缓冲区会引入一个新的问题，即增加播放时延。所谓播放时延是指数据包抵达接收端的时间到最终播放之间的时延。播放时延过长会影响用户体验，因此，抖动缓冲区设计和采用的缓存策略很重要。在WebRTC中，分别使用NetEQ和JitterBuffer处理音频和视频的抖动，以消减抖动造成的影响，尽可能地降低播放时延。

### 抖动估计

*RFC3550* 中给出的计算公式：

首先，使用$S_i,R_i$分别表示第$i$个数据包发送和接收的时间，使用$S_i,R_i$分别表示第$j$个数据包发送和接收的时间，那么这两个数据包传输时延时延的差异值为：

<div>
$$
\begin{equation}
D(i,j)=(R_j-Ri)-(Sj-Si)=(R_j-S_j)-(R_i-S_i)
\end{equation}
$$
</div>

然后，使用一次指数平滑法来消除噪音和过滤掉突发数据的影响，最终得到一个较为合理的估计值：

<div>
$$
\begin{equation}
\hat{J}_i=(1-\sigma)\hat{J}_{i-1}+\sigma{J_i}=\hat{J}_{i-1}+\sigma(J_i-\hat{J}_{i-1})
\end{equation}
$$
</div>

其中，$\hat{J}\_{i}$表示当前的估计值， $\hat{J}\_{i-1}$表示上一次的估计值，$J\_{i}$表示当前观测值，即 $D(i,j)$，$\sigma$表示加权系数，即增益参数，WebRTC中$\sigma=\frac{1}{16}$。

在WebRTC中有两种抖动的估计算法，分别用于以下两处:

- 抖动缓冲区
    
    WebRTC中NetEQ和JitterBuffer使用的抖动估计算法比较复杂：除了在计算$D(i,j)$时考虑了时间回绕，还将计算结果作为观测值传递给卡尔曼滤波器，最终获得一个较为精确的jitter估计值，用于后续的解码和同步播放。详细内容留待后续。
    
- 统计参数
    
    本文介绍的是用作统计参数的抖动估计算法。由于这个抖动估计值只是作为统计参数以提供感观参考，所以使用的估计算法相对比较简单：使用一次指数平滑过滤噪音。
    
    <div align="center"><img src="/imgs/webrtc/statistics_jitter.png"></div>
    
    这个抖动估计值对应于RTCP Report block包的interarrival jitter字段：
    
    ```cpp
       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | fraction lost |       cumulative number of packets lost       |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |           extended highest sequence number received           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                      interarrival jitter                      |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         last SR (LSR)                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                   delay since last SR (DLSR)                  |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    ```
    

### 代码导读

WebRTC中统计参数jitter的估算代码位于文件/src/modules/rtp_rtcp/source/receive_statistics_impl.cc中，函数调用栈为：

1. StreamStatisticianImpl::UpdateCounter
    
    该函数用于统计Rtcp report block中相关的参数，在接收端收到RTP包时被调用。
    
    ```cpp
    void StreamStatisticianImpl::UpdateCounters(const RtpPacketReceived& packet) {
      RTC_DCHECK_EQ(ssrc_, packet.Ssrc());
      int64_t now_ms = clock_->TimeInMilliseconds();
    
      // 接受端码率统计
      incoming_bitrate_.Update(packet.size(), now_ms);
      // 最近一个RTP包的抵达时间
      receive_counters_.last_packet_received_timestamp_ms = now_ms;
      // 统计所有接收到的RTP包，包括重发包
      receive_counters_.transmitted.AddPacket(packet);
      --cumulative_loss_;
    
      // 包序号解回绕
      int64_t sequence_number =
          seq_unwrapper_.UnwrapWithoutUpdate(packet.SequenceNumber());
    
      // 接收到的第一个包
      if (!ReceivedRtpPacket()) {
        received_seq_first_ = sequence_number;
        last_report_seq_max_ = sequence_number - 1;
        received_seq_max_ = sequence_number - 1;
        receive_counters_.first_packet_time_ms = now_ms;
      // 过滤掉乱序包
      } else if (UpdateOutOfOrder(packet, sequence_number, now_ms)) {
        return;
      }
    
      // 正常包
      // In order packet.
      cumulative_loss_ += sequence_number - received_seq_max_;
      received_seq_max_ = sequence_number;
      // 更新当前包序号以便下一次计算
      seq_unwrapper_.UpdateLast(sequence_number);
    
      // 更新抖动需同时满足的两个条件：
      // 1、新接收的包与上一次接收的包源于不同帧，因为同一帧的数据包发送的时间戳是相同的。换言之，抖动估计的最小粒度为帧而非单个数据包。
      // 2、收到的正常包的个数比乱序包至少多1个以上，以确保抖动估计的参考价值，此处存疑？？
      // If new time stamp and more than one in-order packet received, calculate
      // new jitter statistics.
      if (packet.Timestamp() != last_received_timestamp_ &&
          (receive_counters_.transmitted.packets -
           receive_counters_.retransmitted.packets) > 1) {
        // 计算抖动估计值
        UpdateJitter(packet, now_ms);
      }
      last_received_timestamp_ = packet.Timestamp();
      last_receive_time_ms_ = now_ms;
    }
    ```
    
2. StreamStatisticianImpl::UpdateJitter
    
    该函数用于计算抖动估计值，且计算结果采用Q4格式表示。
    
    ```cpp
    void StreamStatisticianImpl::UpdateJitter(const RtpPacketReceived& packet,
                                              int64_t receive_time_ms) {
      // 接收端时延，单位为毫秒
      int64_t receive_diff_ms = receive_time_ms - last_receive_time_ms_;
      RTC_DCHECK_GE(receive_diff_ms, 0);
    	// 计算接收端时延，并将单位转换成采样时间戳单位
      uint32_t receive_diff_rtp = static_cast<uint32_t>(
          (receive_diff_ms * packet.payload_type_frequency()) / 1000);
    	// 计算发送端时延，此处未考虑时间回绕问题，我认为原因有两个：
    	// 1、简化计算，毕竟只是一个统计参数
    	// 2、出现时间回绕时，|send_diff_rtp|会是一个负数，使用uint32_t表示则会是极大值。
    	//    导致接下来计算的|time_diff_samples|也会是极大值，在更新时被视为突发数据而被过滤掉。
    	uint32_t send_diff_rtp = packet.Timestamp() - last_received_timestamp_;
      // 计算接收时延与发送时延的差值，单位为采样时间戳
      int32_t time_diff_samples = receive_diff_rtp - send_diff_rtp;
    
      // 取时延差值的绝对值以方便计算
      time_diff_samples = std::abs(time_diff_samples);
    
      // 过滤因为突发数据导致的剧烈抖动，对应的时延差值为5秒
      // lib_jingle sometimes deliver crazy jumps in TS for the same stream.
      // If this happens, don't update jitter value. Use 5 secs video frequency
      // as the threshold.
      // 出于优化考虑，默认使用视频的采样率，450000=5s x 90000kbps.
      if (time_diff_samples < 450000) {
        // 使用Q4格式以避免浮点数计算
        // Note we calculate in Q4 to avoid using float.
        // 由于|jitter_q4_|采用的是Q4格式，因此需要将|time_diff_samples|转换成Q4格式再进行计算。
        int32_t jitter_diff_q4 = (time_diff_samples << 4) - jitter_q4_;
        // 一次指数平滑法，参考上述公式（2）：
        //   J(i) = J(i-1) + (|D(i-1,i)| - J(i-1))/16
        //        = jitter_q4_ + jitter_diff_q4 / 16
        //        = jitter_q4_ + (jitter_diff_q4 >> 4)
        jitter_q4_ += ((jitter_diff_q4 + /*四舍五入*/8) >> 4);
      }
    }
    ```
    
    由于抖动估算值采用的是Q4格式，且单位为采样时间戳，因此，在作为统计参数使用时需要进行如下转换：
    
    ```cpp
    // 从Q4格式还原成普通数值
    uint32_t jitter = jitter_q4_ >> 4;
    
    // 将单位从采样时间戳转换成毫秒，clock_rate_hz为音频或视频的采样率
    double jitter_ms = static_cast<double>(jitter * 1000 / clock_rate_hz)
    ```
    

### 参考资料

[RFC 3550 - RTP: A Transport Protocol for Real-Time Applications](https://datatracker.ietf.org/doc/html/rfc3550#section-6.4.1)

[抖动和延迟之间的区别](https://webrtc.org.cn/jitter-latency/)