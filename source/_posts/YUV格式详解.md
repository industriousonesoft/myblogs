title: YUV格式详解
date: 2021-3-20 7:00:07
tags:
- 音视频
categories:
- 编程
- 音视频
keywords:
- YUV

---

### 简述

YUV色彩模型不同于RGB的新模型，其原理是利用人类视觉对色彩的亮度比色差更为敏感的特点，将亮度信息从色度信息中分离出来，即使没有色度信息一样可以显示完整的图像，只不过是黑白的。这样的设计很好的解决了彩色电视与黑白电视兼容的问题。

<!-- more -->

YUV使用三个分量来表示像素颜色，分别是：

- Y：Luma
    
    表示明亮度，描述像素的灰度值
    
- U 和 V ：Chroma
    
    表示色度，描述像素的色调及饱和度
    

YUV可以通过缩放和偏移衍生出很多变种，其中YCbCr是在计算机系统中应用最多的一种，JPEG和MPEG均采用这种格式。YCbCr中的Y表示亮度分量，Cb表示蓝色色度分量，Cr表示红色色度分量。

### 多种采样格式

YUV色彩模型支持Y（亮度分量）和UV （色度分量）使用不同的采样率，主流的采样方式有三种，分别是：

- YUV4:4:4 采样
    
    Y分量和UV分量的采样比例相同，即每1个Y分量对于1组UV分量。这种采用方式和RGB色彩模型的图像大小一样。
    
- YUV4:2:2 采样
    
    Y分量和UV分量按照2:1的比例采样，即每采样两个Y分量才采样一组UV分量，因此每2个Y分量对于1组UV分量。这种采用方式比RGB格式节省1/3的存储空间。
    
- YUV4:2:0 采样
    
    Y分量和U或V分量按照2:1的比例采样，即每采样两个Y分量才采样一个U分量或V分量，因此每4个Y分量对于1组UV分量。这种采用方式比RGB格式节省1/2的存储空间。因此，YUV4:2:0被选为主流的采样方式。
    

### 两种存储格式

YUV有两种存储格式，分别是：

- planar格式
    
    先连续存储所有像素点的Y分量，然后是所有像素点的U分量，最后是所有像素点的V分量。比如YUV422P、YUV420P、YUV420SP、YV12和YU12 (属于YUV420)等。
    
- packed格式
    
    每个像素点的Y、U、V分量连续交叉存储。大部分的采样方式都是采用packed格式的存储方式。
    

以16x16的图像为例，以下是常见的YUV存储格式：

1. YUYV格式
    
    YUYV格式属于YUV422采样格式，采用packed存储方式。像素点还原方式：相邻的两个Y分量共用其相邻的两个Cb、Cr分量，比如像素点Y00和Y01共用Cb00和Cr00，其他的像素点以此类推。
    
    ```markdown
    start + 0:  Y00  Cb00  Y01  Cr00  Y02  Cb01  Y03  Cr01
    start + 8:  Y10  Cb10  Y11  Cr10  Y12  Cb11  Y13  Cr11
    start + 16: Y20  Cb20  Y21  Cr20  Y22  Cb21  Y23  Cr21
    start + 24: Y30  Cb30  Y31  Cr30  Y32  Cb31  Y33  Cr31
    ```
    
2. UYVY格式 ，采用packed存储格式
    
    UYVY格式属于YUV422采样格式，采用packed存储方式。与YUYV格式区别在于UV的排列顺序不同，还原像素点的方式与YUYV一样。
    
    ```markdown
    start + 0:  Cb00  Y00  Cr00  Y01  Cb01  Y02  Cr01  Y03
    start + 8:  Cb10  Y10  Cr10  Y11  Cb11  Y12  Cr11  Y13
    start + 16: Cb20  Y20  Cr20  Y21  Cb21  Y22  Cr21  Y23
    start + 24: Cb30  Y30  Cr30  Y31  Cb31  Y32  Cr31  Y33
    ```
    
3. YUV422P
    
    YUV422P，又叫I422，属于YUV422采样格式，采用planar存储方式。
    
    ```markdown
    -------------------------------Y分量
    start + 0:  Y00  Y01  Y02  Y03
    start + 4:  Y10  Y11  Y12  Y13
    start + 8:  Y20  Y21  Y22  Y23
    start + 12: Y30  Y31  Y32  Y33
    -------------------------------U分量
    start + 16: Cb00 Cb01  
    start + 18: Cb10 Cb11 
    start + 20: Cb20 Cb21 
    start + 22: Cb30 Cb31
    -------------------------------V分量
    start + 24: Cr00 Cr01 
    start + 26: Cr10 Cr11 
    start + 28: Cr20 Cr21 
    start + 30: Cr30 Cr31 
    ```
    
4. I420、YV12
    
    I420和YV12都属于YUV420采用格式，采用planar存储方式。I420格式和YV12格式的不同处在U分量和V分量存放的顺序不同。在I420格式中，U分量在V分量之前，故又叫YU12。YV12则恰好相反，U分量在V分量之后。以下是I420的存储格式：
    
    ```markdown
    -------------------------------Y分量
    start + 0:  Y00  Y01  Y02  Y03
    start + 4:  Y10  Y11  Y12  Y13
    start + 8:  Y20  Y21  Y22  Y23
    start + 12: Y30  Y31  Y32  Y33
    -------------------------------U分量
    start + 16: Cb00 Cb01  
    start + 18: Cb10 Cb11
    -------------------------------V分量
    start + 20: Cr00 Cr01 
    start + 22: Cr10 Cr11 
    ```
    
5. NV12、NV21
    
    NV12和NV21属于YUV420采样格式，是一种two-plane模式，Y分量视为一个plane，UV合并视为一个plane。Y和UV两个plane采用planar存储方式，但是UV内部为packed存储方式。
    
    ```markdown
    -------------------------------Y分量
    start + 0:  Y00  Y01  Y02  Y03
    start + 4:  Y10  Y11  Y12  Y13
    start + 8:  Y20  Y21  Y22  Y23
    start + 12: Y30  Y31  Y32  Y33
    -------------------------------UV分量
    start + 16: Cb00 Cr00 Cb01 Cr01
    start + 20: Cb10 Cr10 Cb11 Cr11 
    ```
    

### YUV与RGB转换

鉴于使用YUV422格式或YUV420格式，能够在RGB格式的基础上显著地减少的数据量，且YUV422和YUV420的数据量为RGB的2/3和1/2。因此常将RGB转换成YUV格式进行传输，然后再将YUV格式转换成RGB格式进行显示。二者的转换公式如下：

- YUV转RGB：
    
    ```markdown
    Y = 0.299 * R + 0.587 * G + 0.144 * B
    U = -0.168 * R - 0.331 * G + 0.5 * B + 128
    V = 0.5 * R - 0.419 * G - 0.081 * B + 128 
    ```
    
- RGB转YUV:
    
    ```markdown
    R = Y + 1.13983 * (V - 128)
    G = Y - 0.39465 * (U - 128) - 0.58060 * (V - 128)
    B = Y + 2.03211 * (U - 128)
    ```
    
### 参考资料

[YUV - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/YUV)

[YUV色彩模型与RGB色彩模型详解_jane_6091的博客-CSDN博客_yuv颜色模型](https://blog.csdn.net/jane_6091/article/details/80633680)

[YUV编码格式](https://www.cnblogs.com/-9-8/p/4692653.html)

[图文详解YUV420数据格式](https://www.cnblogs.com/azraelly/archive/2013/01/01/2841269.html)