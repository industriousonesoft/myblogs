title: WebRTC之RTT的两种计算方式
date: 2021-4-10 23:10:07
mathjax: true
tags:
- 拥塞控制
categories:
- 编程
- WebRTC
keywords:
- RTT

---

### 简述

RTT（round-trip time），往返时延，表示在通信过程中，发送端发送的数据包或信号抵达接收端后，再返回发送端的整个往返过程所需时长。

RTT由三个部分决定，分别是

- 链路的传输时长
- 路由器的排队和处理时长
- 接收端的处理时长

由于接收端的处理时长只与接收端的处理逻辑相关，不涉及网络传输，所以在RTT计算过程中一般不包括接收端的处理时长。即：

```cpp
RTT = 链路的传输时长 + 路由器的排队和处理时长
```

其中，路由器的排队和处理时长会随着整个网络的拥塞程度的变化而变化。
简言之，网络拥塞程度越严重，RTT的值也就越大。因此，RTT的变化在一定程度上反应了整个网络的拥塞程度的变化。这也是RTT常被当做重要指标用于分析网络性能的原因。

<!-- more -->

### RTT的计算方式

在WebRTC中根据RTT计算的发起端的不同，可分为以下两种的计算方式：

- 基于发送端
    
    这是WebRTC中默认开启的RTT计算方式，其流程图如下所示：

	<div align="center"><img src="/imgs/webrtc/rtt_sr.png"></div>
    
    计算公式：
    
    <div>
    $$
    \begin{equation}
    \begin{aligned}
    RTT &= T_t - T_0 - delay\_since\_last\_sr \\ &= (T_t - T_0) - (t_1-t_0)
    \end{aligned}
    \end{equation}
    $$
    </div>
    
    与之相关的RTCP报文有：SR（Sender Report）、RR（Receiver Report）和Report Block。
    
    - 相关报文的格式
        - Send Report
            
            ```cpp
            // Sender report (SR) (RFC 3550).
                    0                   1                   2                   3
                    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            header |V=2|P|    RC   |   PT=SR=200   |             length            |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                         SSRC of sender                        |
                   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
            sender |              NTP timestamp, most significant word             |
            info   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |             NTP timestamp, least significant word             |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                         RTP timestamp                         |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                     sender's packet count                     |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                      sender's octet count                     |
                   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
                   |                         report block(s)                       | 
                   |                               ...                             |
            ```
            
            字段解析：
            
            - NTP timestamp：64 bits
                
                表示该SR包发送时的NTP时间戳，之所以使用NTP时间，是因为NTP可以同步不同计算机之间的时钟，降低时间误差。
                
                关于NTP时间的介绍，可参考文章： [计算机中几个与时间相关的概念](/2020/03/10/计算机中几个与时间相关的概念) 
                
            - RTP timestamp：32 bits
                
                表示该SR包发送时的RTP时间戳，单位是采样时间戳，一般与NTP timestamp指向同一时刻，但也有可能会引入一个随机的偏移值。
                
            - sender's packet count ：32 bits
                
                表示在生成该SR包时已经发送的RTP包的总个数。
                
            - sender's octet count：32 bits
                
                表示在生成该SR包时已经发送的RTP包的总字节数。
                
            
            SR包除了自身信息之外，还可以作为Report Block的载体。
            
        - Receiver Report
            
            ```cpp
            // RTCP receiver report (RFC 3550).
                    0                   1                   2                   3
                    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            header |V=2|P|    RC   |   PT=RR=201   |             length            |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                     SSRC of packet sender                     |
                   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
                   |                         report block(s)                       | 
                   |                               ...                             |
            ```
            
            RR包只是Report Block的载体，此外除了RTCP头，无其它字段。
            
        - Report Block
            
            ```cpp
                   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
            report |                     SSRC of media source                      |
            block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
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
            
            字段解析：
            
            - SSRC of media source：32 bits
                
                表示该Report Block包所属的数据流，与发送SR包的数据源所对应的SSRC一致。
                
            - fraction lost：8 bits
                
                表示从发送上一个Report Block包到发送当前Report Block包的这段时间内的丢包率。
                
                采用Q8格式，即用0~255映射0.0~1.0范围。
                
            - cumulative number of packets lost：24 bits
                
                表示SSRC对应的数据流从开始到现在所累积的丢包个数，
                
            - extended highest sequence number received：32 bits
                
                表示接收端当前所接收到的最大的RTP包序号。
                
                该序号是一个解回绕后的值，高16 bits表示序号回绕的次数，低16位表示RTP序号。
                
                NOTE：关于序号回绕技术的介绍留待后续。
                
            - interarrival jitter：32 bits
                
                表示接收端先后接收到的两个RTP包之间的抵达间隔抖动。关于抖动的计算，可参考文章： [WebRTC之抖动估计](/2021/04/05/WebRTC之抖动估计) 
                
            - last SR：32 bits
                
                该字段对应接收端接收到的最新SR包中所携带的NTP时间戳。
                
                NOTE：该字段是一个紧凑型的NTP时间戳，即分别取NTP时间中秒部分的低16 位和小数秒部分的高16位，然后拼凑成一个32位的值。采用紧凑型的NTP时间戳是一种以精度换空间的优化。
                
                NTP时间转换成紧凑型NTP时间戳的方式：
                
                ```cpp
                struct NTPTime {
                		uint32_t seconds;     // 秒
                		uint32_t fractions;   // 小数秒
                };
                
                // 接收端接收到的最新SR包中的NTP字段
                NTPTime last_received_ntp;
                // Compacted NTP timestamp
                // 紧凑型的NTP时间戳
                uint32_t last_sr_timestamp = ((last_received_ntp.seconds & 0x0000ffff) << 16) +
                			                       ((last_received_ntp.fractions & 0xffff0000) >> 16);
                ```
                
            - delay since last SR：32 bits
                
                表示从接收端接收到的最新SR包到发送当前Report Block包之间的时间间隔，即接收端处理SR包的时间。如果截止目前没有收到SR包，则该值为0。
                
                NOTE：该字段也是一个紧凑型的NTP时间戳。
                
    - 代码导读
        1. 发送端发送SR包
            
            有两种情况会触发SR包的发送，分别是：
            
            1.  在每次发送RTP包之前会检查是否可以发送一个SR包
                
                ```cpp
                bool ModuleRtpRtcpImpl2::OnSendingRtpFrame(uint32_t timestamp,
                                                           int64_t capture_time_ms,
                                                           int payload_type,
                                                           bool force_sender_report) {
                  // 如果当前端不是发送端，则直接返回
                  if (!Sending())
                    return false;
                
                  ...
                
                  // 确保SR包在关键帧之前发送，因为SR包的RTP timestamp字段会用于抖动估计和音视频同步
                  // Make sure an RTCP report isn't queued behind a key frame.
                  // 检查是否可以发送新的SR包
                  if (rtcp_sender_.TimeToSendRTCPReport(force_sender_report)) 
                    // 通过RTCPSender来发送SR包
                    rtcp_sender_.SendRTCP(GetFeedbackState(), kRtcpReport);
                
                  return true;
                }
                ```
                
            2. 在RTCPSender发送Report Block包时，如果是当前端是发送端则会使用SR作为载体
                
                ```cpp
                // RTCP的Report包：Report Block、XR中扩展报告、SDES包等
                void RTCPSender::PrepareReport(const FeedbackState& feedback_state) {
                  bool generate_report;
                  // 允许发送RTCP Report的两类情况：
                  if (IsFlagPresent(kRtcpSr) || IsFlagPresent(kRtcpRr)) {
                    // 情况一：在每次发送SR包或RR包时携带Report Block，因为Report Block只能以SR包或RR包作为载体发送
                    // Report type already explicitly set, don't automatically populate.
                    generate_report = true;
                    RTC_DCHECK(ConsumeFlag(kRtcpReport) == false);
                  } else {
                	  // 情况二：当前发送的RTCP包类型中不包括SR或RR，则需要满足以下任一条件：
                    // a. 当前RTCP模式是复合模式（kCompound）
                    //    这种模式下，发送任意的RTCP包都会触发Report Block的发送。
                    // b. 当前RTCP模式是简化模式（kReducedSize）
                    //    只有当前发送的RTCP包类型中包含了Report Block类型，即添加了kRtcpReport标识，才会触发Report Block的发送。
                    generate_report =
                        (ConsumeFlag(kRtcpReport) && method_ == RtcpMode::kReducedSize) ||
                        method_ == RtcpMode::kCompound;
                    // 如果是发送端则使用SR包作为Report Block的载体，否则使用RR包作为载体。
                    if (generate_report)
                      SetFlag(sending_ ? kRtcpSr : kRtcpRr, true);
                  }
                
                  // 检测是否需要发送SDES包
                  if (IsFlagPresent(kRtcpSr) || (IsFlagPresent(kRtcpRr) && !cname_.empty()))
                    SetFlag(kRtcpSdes, true);
                
                	// 允许发送RTCP Report。
                  if (generate_report) {
                    // 检测是否需要发送扩展报告包，载体是XR包。
                    // 与基于接收端的RTT计算方式相关的两个RTCP包：Receiver Reference Time Report Block 
                    // 和DLRR Report Block**，**以及视频分配码率的报告都是通过XR包作为载体发送。
                    if ((!sending_ && xr_send_receiver_reference_time_enabled_) ||
                        !feedback_state.last_xr_rtis.empty() ||
                        send_video_bitrate_allocation_) {
                      SetFlag(kRtcpAnyExtendedReports, true);
                    }
                
                    // Report Block的发送是周期性的
                    // generate next time to send an RTCP report
                    TimeDelta min_interval = report_interval_;
                
                    if (!audio_ && sending_) {
                      // 音频的RTCP报告使用固定的周期，视频的RTCP报告周期则与当前的码率相关：优先取当前码率的36%用于发送RTCP报文。
                      // Calculate bandwidth for video; 360 / send bandwidth in kbit/s.
                      int send_bitrate_kbit = feedback_state.send_bitrate / 1000;
                      if (send_bitrate_kbit != 0) {
                        min_interval = std::min(TimeDelta::Millis(360000 / send_bitrate_kbit),
                                                report_interval_);
                      }
                    }
                
                    // 部署下一次发送Report Block的间隔时间，是一个任意值。
                    // 取范围是下一次发送时刻的前后0.5毫秒内：[interval_ms - 0.5，interval_ms + 0.5]
                    // The interval between RTCP packets is varied randomly over the
                    // range [1/2,3/2] times the calculated interval.
                    int min_interval_int = rtc::dchecked_cast<int>(min_interval.ms());
                    TimeDelta time_to_next = TimeDelta::Millis(
                        random_.Rand(min_interval_int * 1 / 2, min_interval_int * 3 / 2));
                
                    // 部署下一次Report Block的发送
                    RTC_DCHECK(!time_to_next.IsZero());
                    SetNextRtcpSendEvaluationDuration(time_to_next);
                
                    // RtcpSender expected to be used for sending either just sender reports
                    // or just receiver reports.
                    RTC_DCHECK(!(IsFlagPresent(kRtcpSr) && IsFlagPresent(kRtcpRr)));
                  }
                }
                ```
                
        2. 接收端收到SR包之后，返回一个RR包或SR包
            
            如果当前端本身既是发送端又是接收端，则在收到SR包后会在下次发送SR包时将Report Block一并返回。
            
            如果当前端只是接受端，那么则为以RR为载体将Report Block返回。
            
            同上，相关代码见函数RTCPSender::PrepareReport。
            
        3. 发送端在收到RR包或SR包后，提取相关字段计算RTT值
            
            RTT的计算位于发送端的RTCPReceiver中，以下是相关代码：
            
            ```cpp
            void RTCPReceiver::HandleReportBlock(const ReportBlock& report_block,
                                                 PacketInformation* packet_information,
                                                 uint32_t remote_ssrc) {
              // 过滤不属于当前端的Report Blocks
              if (!registered_ssrcs_.contains(report_block.source_ssrc()))
                return;
            
            	// 接收到Report Block的系统时间
              last_received_rb_ = clock_->CurrentTime();
              ...
            
              // 根据Report Block计算RTT
              int64_t rtt_ms = 0;
              // T0：接收端收到的最新的SR包中的NTP时间戳
              uint32_t send_time_ntp = report_block.last_sr();
            
              // send_time_ntp为0表示接收端暂未收到SR包
              if (send_time_ntp != 0) {
                // 接收端的处理时长
                uint32_t delay_ntp = report_block.delay_since_last_sr();
                // T1：将系统时间转换成NTP时间后再格式化成紧凑型NTP时间戳
                // Local NTP time.
                uint32_t receive_time_ntp =
                    CompactNtp(clock_->ConvertTimestampToNtpTime(last_received_rb_));
            
                // 计算RTT，因为在NTP时间戳中，秒和小数秒部分都是16位，所以单位是1/(2^16)秒
                // RTT in 1/(2^16) seconds.
                uint32_t rtt_ntp = receive_time_ntp - delay_ntp - send_time_ntp;
                // 将RTT的单位转换成毫秒
                // Convert to 1/1000 seconds (milliseconds).
                rtt_ms = CompactNtpRttToMs(rtt_ntp);
                report_block_data->AddRoundTripTimeSample(rtt_ms);
                if (report_block.source_ssrc() == main_ssrc_) {
                  rtts_[remote_ssrc].AddRtt(TimeDelta::Millis(rtt_ms));
                }
            
                packet_information->rtt_ms = rtt_ms;
              }
            	...
            }
            ```
            
- 基于接收端
    
    WebRTC中可通过设置RTCPSender中的xr_send_receiver_reference_time_enabled_来开启这种计算方式，其流程图如下所示：
    
	<div align="center"><img src="/imgs/webrtc/rtt_xr.png"></div>
    
    计算公式：
    
    <div>
    $$
    \begin{equation}
    \begin{aligned}
    RTT &= T_t - T_0 - delay\_since\_last\_sr \\ &= (T_t - T_0) - (t_1-t_0)
    \end{aligned}
    \end{equation}
    $$
    </div>
    
    与之相关的RTCP报文有：RRTR（Receiver Reference Time Report Block）、DLRR（DLRR Report Block）。
    
    - 相关报文的格式
        - Receiver Reference Time Report Block
            
            ```cpp
            // Receiver Reference Time Report Block (RFC 3611).
            
             0                   1                   2                   3
             0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |     BT=4      |   reserved    |       block length = 2        |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |              NTP timestamp, most significant word             |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             NTP timestamp, least significant word             |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            ```
            
            RRTRB其实就是一个简化版的SR，字段解析可参考上述SR报文的解析。
            
        - DLRR Report Block
            
            ```cpp
            // DLRR Report Block (RFC 3611).
             0                   1                   2                   3
             0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |     BT=5      |   reserved    |         block length          |
            +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
            |                 SSRC_1 (SSRC of first receiver)               | sub-
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ block
            |                         last RR (LRR)                         |   1
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |                   delay since last RR (DLRR)                  |
            +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
            |                 SSRC_2 (SSRC of second receiver)              | sub-
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ block
            :                               ...                             :   2
            ```
            
            sub-block其实就是一个简化版的Report Block，而DLRR则是sub-block的载体。一个DLRR报文可包含多个sub-block，字段解析可参考上述Report Block报文的解析。
            
    - 代码导读
        1. 接收端发送一个RRTRB包
            
            RRTRB包的发送逻辑同Report block一样，位于RTCPSender::PrepareReport中。
            
        2. 发送端收到RRTRB包后，返回一个DLRR包
            
            DLRR包的发送逻辑同Report block一样，位于RTCPSender::PrepareReport中。
            
        3. 接收端收到DLRR包后，提取相关字段计算RTT值
            
            RTT的计算位于接收端的RTCPReceiver中，以下是相关的代码：
            
            ```cpp
            void RTCPReceiver::HandleXrDlrrReportBlock(uint32_t sender_ssrc,
                                                       const rtcp::ReceiveTimeInfo& rti) {
              // 过滤不属于当前接收端的DLRR包
              if (!registered_ssrcs_.contains(rti.ssrc)) 
                return;
              
              // 这种RTT计算方式需要显示设置启动
              // Caller should explicitly enable rtt calculation using extended reports.
              if (!xr_rrtr_status_)
                return;
            
              // T0：发送端收到的最新的RRTRB包中的NTP时间戳，因为在NTP时间戳中，秒和小数秒部分都是16位，所以单位是1/(2^16)秒
              // The send_time and delay_rr fields are in units of 1/2^16 sec.
              uint32_t send_time_ntp = rti.last_rr;
              // send_time_ntp == 0表示发送端暂时尚未收到RRTRB包
              // RFC3611, section 4.5, LRR field discription states:
              // If no such block has been received, the field is set to zero.
              if (send_time_ntp == 0) {
                auto rtt_stats = non_sender_rtts_.find(sender_ssrc);
                if (rtt_stats != non_sender_rtts_.end()) {
                  rtt_stats->second.Invalidate();
                }
                return;
              }
             
              // 发送端的处理时长
              uint32_t delay_ntp = rti.delay_since_last_rr;
              // T1：接收端收到DLRR的NTP时间戳
              uint32_t now_ntp = CompactNtp(clock_->CurrentNtpTime());
              // 计算RTT，格式为NTP时间戳
              uint32_t rtt_ntp = now_ntp - delay_ntp - send_time_ntp;
              // 将RTT的单位转换成毫秒
              xr_rr_rtt_ms_ = CompactNtpRttToMs(rtt_ntp);
            
              non_sender_rtts_[sender_ssrc].Update(TimeDelta::Millis(xr_rr_rtt_ms_));
            }
            ```
            
        

### 参考资料

[RFC 3550 - RTP: A Transport Protocol for Real-Time Applications](https://datatracker.ietf.org/doc/html/rfc3550#section-6.4.1)

[RFC 3611 - RTP Control Protocol Extended Reports (RTCP XR)](https://datatracker.ietf.org/doc/html/rfc3611)