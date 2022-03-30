title: WebRTC之包组的时间间隔计算
date: 2021-4-20 23:10:07
mathjax: true
tags:
- 拥塞控制
categories:
- 编程
- WebRTC
keywords:
- inter-departure
- inter-arrival

---

### 简述

在WebRTC中，使用的GCC（Google Congestion Control）作为拥塞控制算法，且该算法有新旧两个版本。二者的主要区别之一就是基于延迟的带宽估计算法的实现不同：旧版采用的是基于接收端的Kalman滤波器带宽评估算法，而新版则是基于发送端的Trendline滤波器带宽评估算法。

此前在 [WebRTC之抖动估计](/2021/04/05/WebRTC之抖动估计) 中简单介绍过关于时延的概念，简单说就是指一个数据包或信号从发送端抵达接收端所需的时长。由于网络拥塞等原因，即便是以均匀时间间隔连续发送的数据包，在抵达接收端时的时间间隔也很可能是不均匀的。

新版采用的Trendline滤波器算法，是一种基于包的时延梯度的变化来评估当前网络变化趋势的算法，而时延梯度的计算基于发送时间间隔（inter-departure）和抵达时间间隔（inter-arrival）。

<div align="center"><img src="/imgs/webrtc/inter_arrival.png" width="60%" height"60%"></div>

关于Trendline滤波器带宽评估算法的具体实现留待后续，本文介绍的是关于包组时间间隔的计算方式。

<!-- more -->

在WebRTC中，引入了包组的概念，以此来计算发送时间间隔和抵达时间间隔，而非并基于单个包或帧进行计算，理由有二：

1. 由于同一帧的所有包的发送时间是相同的，所以理论上计算发送时间间隔的粒度为帧。但是当一帧数据被拆分成多个包发送后，可能因为丢包或乱序等原因，导致在接收端不能再还原成一个完整的帧。
2. 由于WebRTC在发送端会采用Pacer模块按簇发送包，因此包一般也是按簇抵达接收端。同一簇一般包含多个包，且包之间的抵达时间间隔很小。因此，使用包组不仅可以减少计算量，而且更符合网络传输中的真实情况，可以更好处理突发数据、丢包或乱序等情况。

### 代码导读

在WebRTC中有一个专用于计算包组时间间隔的类：InterArrivalDelta，具体的计算过程包括以下几个步骤：

- 包组划分
    
    计算包组的时间间隔的第一步就是划分包组，且至少有两个包组才能计算时间间隔。
    
    包组的划分是基于包的发送时间间隔，在WebRTC中，如果新抵达的包的发送时间与当前包组中第一个包的发送时间之间的间隔时长大于5毫秒，则被视为新包组的第一个包。以下是GCC-02 草案给出的相关说明：
    
    > The Pacer sends a group of packets to the network every burst_time interval.  RECOMMENDED value for burst_time is 5 ms.
    > 
    
    由于突发数据的存在，在划分包组之前需要对包进行预处理，避免因突发数据导致的计算误差。
    
    - 处理突发数据
        
        由于网络中断等非网络拥塞原因，路由器中待处理的数据包会被滞留在路由器的网络缓冲区。等网络重新恢复后，路由器为了及时清空缓冲区，会在短时间内将缓冲区的包发送出去，这些包就属于突发数据包。
        
        以下是判断突发数据包的相关代码：
        
        ```cpp
        bool InterArrivalDelta::BelongsToBurst(Timestamp arrival_time,
                                               Timestamp send_time) const {
          RTC_DCHECK(current_timestamp_group_.complete_time.IsFinite());
          // NOTE：由于突发数据包之间的发送和抵达时间间隔很短，因此，在判断突发数据包时，
          // 应该使用包组最新的发送时间和抵达时间来计算发送时间间隔和抵达时间间隔。
        
          // 当前包的抵达时间与当前包组最新抵达时间之间的间隔差值。
          TimeDelta arrival_time_delta =
              arrival_time - current_timestamp_group_.complete_time;
          // 当前包的发送时间与当前包组的最新发送时间之间的间隔差值。
          TimeDelta send_time_delta = send_time - current_timestamp_group_.send_time;
          // 发送时间间隔差值为0，表示当前包与当前包组中最近抵达的包属于同一帧，因为也属于当前包组。
          if (send_time_delta.IsZero())
            return true;
          // 当前包与当前包组的时延差值，即时延梯度。
          TimeDelta propagation_delta = arrival_time_delta - send_time_delta;
          // 判断当前包是否属于突发数据包，需同时满足一下三个条件：
          // 1. 时延梯度小于0，即抵达时间间隔大于发送时间间隔；
          // 2. 抵达时间间隔小于或等于5ms，即接收端的抵达时间间隔小于发送端的发送时间间隔，
          //    此外，结合条件1可得：发送时间间隔一定是小于5ms的；
          // 3. 当前包与当前包组中最早抵达的包之间的抵达时间间隔小于100ms，因为突发数据包一定是在短时间内抵达的；
          if (propagation_delta < TimeDelta::Zero() &&
              arrival_time_delta <= /*5ms*/kBurstDeltaThreshold &&
              arrival_time - current_timestamp_group_.first_arrival < /*100ms*/kMaxBurstDuration)
            return true;
        
          // 当前包不属于突发数据包。
          return false;
        }
        ```
        
    - 划分包组
        
        如果当前包不属于突发数据包，则根据发送时间间隔判断当前包是否属于新的包组：
        
        ```cpp
        bool InterArrivalDelta::NewTimestampGroup(Timestamp arrival_time,
                                                  Timestamp send_time) const {
          if (current_timestamp_group_.IsFirstPacket()) {
            // 当前包组的首个抵达的包一定是属于当前包组的。
            return false;
          } else if (BelongsToBurst(arrival_time, send_time)) {
            // 突发数据包属于当前包组。
            return false;
          } else {
        		// NOTE：此处使用的包组中最早抵达的包的发送时间来计算发送时间间隔，原因在于：
            // 包组在接收新包时已经将乱序包过滤掉，保证了包组中的其他所有包的发送时间都是
            // 小于首个包的发送时间。换言之，包组中的所有包可以理解为是有序的。
        
            // 计算当前包与当前包组中最早抵达的包之间的发送时间间隔，如果大于|send_time_group_length|
            //（默认值为5ms），则当前包被划分为新的包组，否则属于当前包组。
            return send_time - current_timestamp_group_.first_send_time >
                   send_time_group_length_;
          }
        }
        ```
        
- 计算包组的时间间隔
    
    相关的代码位于InterArrivalDelta::ComputeDeltas中：
    
    ```cpp
    bool InterArrivalDelta::ComputeDeltas(Timestamp send_time,
                                          Timestamp arrival_time,
                                          Timestamp system_time,
                                          size_t packet_size,
                                          TimeDelta* send_time_delta,
                                          TimeDelta* arrival_time_delta,
                                          int* packet_size_delta) {
      // 计算包组的时间间隔的前提是：已有两个完成的包组，因此每次更新满足不一定会满足计算条件。
      bool calculated_deltas = false;
      // 当前包是当前包组首个抵达的包，加入当前包组。
      if (current_timestamp_group_.IsFirstPacket()) {
        // We don't have enough data to update the filter, so we store it until we
        // have two frames of data to process.
        current_timestamp_group_.send_time = send_time;
        current_timestamp_group_.first_send_time = send_time;
        current_timestamp_group_.first_arrival = arrival_time;
      } else if (current_timestamp_group_.first_send_time > send_time) {
        // 如果当前包的发送时间早于比当前包组首个抵达的包，则视为乱序包。
        // 过滤乱序包，即包组中的其他包都是在首个抵达的包之后发送的，
        // 以确保在函数NewTimestampGroup中划分新包组时的有效性。
        // Reordered packet.
        return false;
      } else if (NewTimestampGroup(arrival_time, send_time)) {
        // 不包括新的包组，当前已经有两个完整的包组，则开始计算包组的时间间隔。
        // First packet of a later send burst, the previous packets sample is ready.
        if (prev_timestamp_group_.complete_time.IsFinite()) {
          // 计算相邻的两个包组的发送时间间隔。
          *send_time_delta =
              current_timestamp_group_.send_time - prev_timestamp_group_.send_time;
          // 计算相邻的两个包组的抵达时间间隔。
          *arrival_time_delta = current_timestamp_group_.complete_time -
                                prev_timestamp_group_.complete_time;
          // 计算包组的系统时间偏差值。
          TimeDelta system_time_delta = current_timestamp_group_.last_system_time -
                                        prev_timestamp_group_.last_system_time;
    
          // 系统时钟的偏差过大，超过3s，则重置包组信息。理由是什么？
          if (*arrival_time_delta - system_time_delta >=
              kArrivalTimeOffsetThreshold) {
            RTC_LOG(LS_WARNING)
                << "The arrival time clock offset has changed (diff = "
                << arrival_time_delta->ms() - system_time_delta.ms()
                << " ms), resetting.";
            Reset();
            return false;
          }
          // 如果接收端在某一段时间内收到大量的重传包或乱序包，可能会导致包组划分也出现乱序的情况。
          // 因此，在计算包组的时间间隔之前需要判断当前的两个包组是否是乱序的。
          // 如果当前相邻的两个包组是乱序的，则不计算包组的时间间隔。
          if (*arrival_time_delta < TimeDelta::Zero()) {
            // 记录包组乱序的累积次数，如果累积超过3次，表示当前时段网络中的重传包或乱序包就较多，
            // 则重置包组信息。
            // The group of packets has been reordered since receiving its local
            // arrival timestamp.
            ++num_consecutive_reordered_packets_;
            if (num_consecutive_reordered_packets_ >= kReorderedResetThreshold) {
              RTC_LOG(LS_WARNING)
                  << "Packets between send burst arrived out of order, resetting."
                  << " arrival_time_delta" << arrival_time_delta->ms()
                  << " send time delta " << send_time_delta->ms();
              Reset();
            }
            return false;
          } else {
            // 如果当前相邻的两个包组是顺序的，则忽略之前的乱序包组，使用当前的包组计算。
            num_consecutive_reordered_packets_ = 0;
          }
          // 计算包组之间数据大小的差值
          *packet_size_delta = static_cast<int>(current_timestamp_group_.size) -
                               static_cast<int>(prev_timestamp_group_.size);
          // 表示更新了包组的时间间隔
          calculated_deltas = true;
        }
        // 将新包组更新为当前包组
        prev_timestamp_group_ = current_timestamp_group_;
        // The new timestamp is now the current frame.
        current_timestamp_group_.first_send_time = send_time;
        current_timestamp_group_.send_time = send_time;
        current_timestamp_group_.first_arrival = arrival_time;
        current_timestamp_group_.size = 0;
      } else {
        // 使用包组中最新发送的包的发送时间作为包组的发送时间
        current_timestamp_group_.send_time =
            std::max(current_timestamp_group_.send_time, send_time);
      }
      // 更新当前包组的信息
      // Accumulate the frame size.
      current_timestamp_group_.size += packet_size;
      current_timestamp_group_.complete_time = arrival_time;
      current_timestamp_group_.last_system_time = system_time;
    
      return calculated_deltas;
    }
    ```
    
    NOTE：系统时钟是以系统启动为基准点开始计时的，因此每次系统启动后都会重新计时，具体介绍可参考 [计算机中几个与时间相关的概念](/2020/03/10/计算机中几个与时间相关的概念) 。
    

### 参考资料

[draft-ietf-rmcat-gcc-02](https://datatracker.ietf.org/doc/html/draft-ietf-rmcat-gcc-02)