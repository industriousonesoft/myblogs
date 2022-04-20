title: H264之IDR帧和I帧的区别
date: 2022-4-19 22:17:07
mathjax: true
tags:
- 音视频
- H264
categories:
- 编程
- 音视频
keywords:
- H264

---

> 面试时被问到的一个问题，没答上来，算是查漏补缺。
> 

### 简述

IDR（Instantaneous Decoder Refresh），即时解码刷新，是一种特殊的I帧。IDR帧是为了防止H264解码器在解码时参考无意义的帧而设置的。

当H264解码器收到IDR帧时，意味着后续抵达的帧不会再参考IDR帧之前的帧，因此，H264解码器会“清空”参考缓冲区（the reference buffer），而所谓的“清空”有可能是将参考缓冲区中的所有帧标识为“不可参考（unused for reference）”状态。

相较而言，当H264解码器收到普通的I帧时，后续抵达的帧有可能会参考这个I帧之前的帧，即参考缓冲区中的帧。

<!-- more -->

### 举例

假设某一段H264视频使用了多重参照帧，且帧序列如下所示：

<div>
$$
\begin{equation}
I\ P\ B\ P\ B\ P\ B\ B\ P_i\ I_j\ P_k\ B...
\end{equation}
$$
</div>

那么，$P_k$帧在参考$I_j$的同时，还有可能会参考$P_i$帧。但是如果$I_j$帧前后是一个场景的切换，即$I_j$帧前后的场景可能会存在很大的反差，此时$P_k$帧参考$I_j$帧之前的帧已经没有太大的意义，反而降低了解码的效率。

此时，如果将$I_j$帧替换成IDR帧，即帧序列如下所示：

<div>
$$
\begin{equation}
I\ P\ B\ P\ B\ P\ B\ B\ P_i\ I\!D\!R_j\ P_k\ B...
\end{equation}
$$
</div>

由于IDR帧禁止后续的帧参考在其之前的帧，故$P_k$帧不会再参考$P_i$帧。

### 用途

IDR帧的引入可解决在视频播放过程中进行点播、快进或快退等操作导致视频失真等问题。此外，H264编码器的第一个I帧通常都是IDR帧，可实现快速播放。

### 参考资料

[Difference between I frame and IDR frame](https://malleshamdasari.wordpress.com/2013/07/31/difference-between-i-frame-and-idr-frame/)