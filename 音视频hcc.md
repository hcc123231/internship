# 音视频

## 一、采集

摄像头采集作为嵌入式摄像头工作的起点，需要与硬件协调，操控硬件以实现光信号转为电信号最终转为数字信号

### 1、上电

```
---摄像头由两部分硬件模组组成：传感器(Sensor)、片上系统(SoC)
传感器负责将采集到的光信号转为电信号，而片上系统相当于控制端，执行指令，发布命令，与Sensor交互
```



>* Sensor模组和SoC模组上电，配置sensor模组的参考时钟(通过SoC模组的MCLK引脚连上Sensor模组的XCLK引脚直接输出)。
>* MIPI CSI（Mobile Industry Processor Interface Camera Serial Interface）接口的SoC端控制器上电，配置该控制器的速率和lane（实际使用的车道数），MIPI CSI是多车道串行通信，车道物理上限存在，这里的lane是在配置逻辑上使用到的车道数
>* I2C接口的SoC端控制器上电，配置I2C访问Sensor，I2C是设备通信总线，主机可以发布指令给从机执行

### 2、传感器配置

```
---Sensor需要SoC提供的参数来决定其采集的行为
```

>* Sensor通过I2C总线写Sensor寄存器表：分辨率、帧率、曝光行、增益、出图格式、MIPI参数（协商Sensor内部的MIPI CSI控制器行为）等

**注意** I2C是控制路径，MIPI CSI是图像传输路径

### 3、图像流水线传输

```
---摄像头两部分上电，Sensor侧通过I2C配置完成，SoC侧也已配置完成，接下来就是图像数据的单边传输
```

>* SoC在最后一次配置时向Sensor的流控制寄存器写入开始值，Sensor开始传输图像数据
>* Sensor通过内部的MIPI TX控制器将像素数据打包成MIPI CSI协议格式的数据包
>* SoC端的MIPI CSI协议解析器解包得到原始的像素数据然后写入到MIPI CSI控制器的FIFO（硬件先进先出队列）中，等到FIFO中数据量达到预设值，控制器就会发起DMA请求，DMA将FIFO中数据搬运到内存中的帧缓冲区，等缓冲区收到完整一帧时，控制器会产生中断通知CPU，然后CPU或者ISP进行降噪、AE/AWB 统计与校正、色彩与 Gamma 等，输出 YUV/RGB。

### 4、驱动

```
---除了SoC之外所有的硬件模块都需要在操作系统中找到对应的驱动才能正常工作
```

> * Sensor驱动：robe、电源管理、寄存器初始化、与 V4L2/私有媒体框架对接。
> * ISP 驱动：申请 buffer、配置硬件寄存器、处理中断（帧完成、错误）

## 二、编码

```
经过上述流程，此时YUV格式图像数据存储在内存中，接下来可以由显示控制器来取数据预览显示（相机，手机摄像等），也可以本地不显示直接通过网络发送（压缩编码发送）
```

### 硬件编码

```
在嵌入式安防摄像头场景下，采集到的数据最终是要推送到云端的，所以必须将原始的YUV数据编码压缩，并且7X24小时不间断采集、编码，对编码的要求较高，硬件编码功耗低，不占用cpu，所以采用硬件编码
```

> 以H.264为例，硬件编码器从YUV内存缓冲区中读取原始YUV数据，根据编码器参数编码，输出NALU,，NALU是由NALU头+压缩后数据组成，这里的压缩后的数据可能是图像数据，也可能是sps,pps，一开始会输出sps,pps的NALU，需要在IDR帧之前再次输出sps,pps的NALU

### H.264编码输出码流

![H.264编码器输出NALU结构图](https://raw.githubusercontent.com/hcc123231/pictures/main/H264%20NALU%20construction.png)

> 这就是H.264编码器输出单元NALU的结构，更具体一点的码流序列就是
>
> **[ (start code) (NALU header) (sps) ]         [ (start code) (NALU header) (pps) ]          [ (start code) (NALU header) (IDR帧slice data) ] ... **
>
> **[ (start code) (NALU header) (P帧或B帧slice data) ]  ...**
>
> 可以看到每个IDR帧NALU输出之前都要先输出sps,ppsNALU,因为IDR帧是**瞬时解码刷新帧**，他会清空参考帧缓存，所以后续解码器需要依赖新的sps,pps才能正常完成解码。当然也并不是固定这种排列顺序，有的编码器只会在开头输出sps NALU,pps NALU，这点了解就行。另外编码器可能还会输出SEI，全称是**补充增强信息**，它也是作为参数信息被编码器输出的，和sps,pps一样，编码器往往会**[SPS] [PPS] [SEI(如果存在)] [IDR] [P] [P] [B] [B]这样输出NALU单元**



> 既然编码器一定会输出SPS,PPS这些信息，那SPS,PPS到底是什么呢？
>
> **SPS：定义全局解码语法和约束（分辨率、参考帧结构、POC 等）**
>
> **PPS：定义具体的编码/解码策略（QP、熵编码、滤波等），供 slice 引用**
>
> 编码器根据初始化时传入的配置参数生成SPS,PPS

**注意**：再补充一点，H.265/HEVC除了SPS,PPS之外还有VPS参数集

#### SPS字段及含义

> | 字段名称                             | 作用/含义                                                    |
> | :----------------------------------- | :----------------------------------------------------------- |
> | **`profile_idc`**                    | 定义了编码所用的“档次”（如66为Baseline，77为Main）。不同档次支持不同的编码工具集。 |
> | **`level_idc`**                      | 定义了编码的“级别”，限制了该档次下支持的最大分辨率、帧率、码率等。 |
> | **`seq_parameter_set_id`**           | 为当前SPS分配一个ID，供PPS引用。                             |
> | **`pic_order_cnt_type`**             | 指定了计算图像显示顺序(POC, Picture Order Count)的方法，确保画面按正确顺序播放。 |
> | **`pic_width_in_mbs_minus1`**        | **用于计算视频宽度**。计算值为：`宽度 = (pic_width_in_mbs_minus1 + 1) * 16`。单位是宏块(Macroblock)，每个宏块为16x16像素。 |
> | **`pic_height_in_map_units_minus1`** | **用于计算视频高度**。计算值为：`高度 = (pic_height_in_map_units_minus1 + 1) * 16`。 |
> | **`frame_cropping_flag`**            | 裁剪标志。当视频分辨率不是16的整数倍时，会通过此标志和后续的裁剪偏移量字段进行裁剪。 |
> | **`frame_mbs_only_flag`**            | 标志位，指明编码的帧是逐行扫描(progressive)还是隔行扫描(interlaced)。 |
> | **`vui_parameters_present_flag`**    | 标志位，指明SPS之后是否跟随VUI (Video Usability Information)参数。VUI参数提供了色彩、宽高比等信息。 |

#### PPS字段及含义

> | 字段名称                                     | 作用/含义                                                |
> | :------------------------------------------- | :------------------------------------------------------- |
> | **`pic_parameter_set_id`**                   | 为当前PPS分配一个ID，供Slice引用。                       |
> | **`seq_parameter_set_id`**                   | 指明当前PPS引用了哪个SPS，从而将两者关联起来。           |
> | **`entropy_coding_mode_flag`**               | 指明熵编码模式：0通常为CAVLC或指数哥伦布编码，1为CABAC。 |
> | **`num_slice_groups_minus1`**                | 指明一帧图像被划分为多少个片组(Slice Group)。            |
> | **`num_ref_idx_l0_active_minus1`**           | 指明参考图像列表0的最大参考帧索引号。                    |
> | **`pic_init_qp_minus26`**                    | 指明初始量化参数(QP)，用于控制画面质量与压缩率。         |
> | **`deblocking_filter_control_present_flag`** | 标志位，指明是否启用去块效应滤波器及相关的控制信息。     |

#### 编码流程

![](https://raw.githubusercontent.com/hcc123231/pictures/main/code%20process.png)

#### 编码参数

![](https://raw.githubusercontent.com/hcc123231/pictures/main/h264%20params.png)

## 三、推送

设备作为服务端等待客户端连接请求拉流，本质上就是网络通信

### 1、rtsp协议

rtsp推拉流涉及到三种协议：rtsp(会话控制层)，rtp(媒体传输层)，rtcp(质量监督层)

#### OPTIONS

OPTIONS请求示例

```
OPTIONS rtsp://192.168.1.100/stream RTSP/1.0
CSeq: 1
User-Agent: VLC/3.0.18
```

可以看到格式与http极为类似，请求行、请求头、请求体（为空），请求行以方法名作为起始标识，紧跟着url，再然后就是RTSP版本号，请求头以key: value形式罗列多行，CSeq表示每一次rtsp交互的序列号，要求响应包中带有相同的CSeq，最后是一个可选的用户代理字段



OPTIONS响应示例

```
RTSP/1.0 200 OK
CSeq: 1
Public: OPTIONS, DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE, GET_PARAMETER, SET_PARAMETER
Server: Live555 Streaming Media v2023.01.19
Date: Wed, Apr 20 2026 10:30:00 GMT
```

遵循响应行、响应头、响应体（为空）的格式，响应行以RTSP版本为起始标识，紧跟着状态码。响应头也是key: value的形式存在，CSeq序列号，Public服务端支持的方法名，Server服务器软件标识（可选），Date时间（可选）

#### DESCRIBE

DESCRIBE请求示例

```
DESCRIBE rtsp://192.168.1.100:554/stream RTSP/1.0
CSeq: 2
Accept: application/sdp
User-Agent: VLC/3.0.18
```

在OPTIONS基础上添加了一个Accept字段，表示客户端希望接收的文本格式描述



DESCRIBE响应示例

```
RTSP/1.0 200 OK
CSeq: 2
Content-Type: application/sdp
Content-Base: rtsp://192.168.1.100:554/stream/
Content-Length: 542

v=0
o=- 1624567890 1624567890 IN IP4 192.168.1.100
s=H.264 Stream
c=IN IP4 0.0.0.0
t=0 0
a=control:*
a=range:npt=0-999.999
m=video 0 RTP/AVP 96
b=AS:2000
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1; profile-level-id=42e01f; sprop-parameter-sets=Z0LAH9oBQO0K,Qy5R0A==
a=control:track1
```

Content-Type响应体文本格式，Content-Base：sdp描述中出现的所有相对路径都应该基于这个基础URL来解析出绝对URL，Content-Lenght：响应体长度

SDP字段解析

> * v=0 ,SDP版本号，固定为0
> * o=- 1624567890 1624567890 IN IP4 192.168.1.100，会话发起者信息：用户名、会话id、版本、网络类型、地址
> * s=H.264 Stream ：会话名称，可自定义
> * c=IN IP4 0.0.0.0：连接信息，ip地址为0.0.0.0表示在SETUP阶段再指定
> * t=0 0：会话起始/结束时间（0表示无限制）
> * a=control:*：整个会话的控制URL基地址
> * a=range:npt=0-999.999：支持的时间范围
> * m=video 0 RTP/AVP 96：媒体描述：视频类型、端口0（表示在SETUP阶段指定）、RTP/AVP传输，负载类型96（动态分配）
> * b=AS:2000：带宽（单位是kbps)
> * a=rtpmap:96 H264/90000：负载类型96对应H.264编码，时钟频率90kHz
> * a=fmtp:96 packetization-mode=1; profile-level-id=42e01f; sprop-parameter-sets=Z0LAH9oBQO0K,Qy5R0A==：fmtp:96 packetization-mode打包模式，0：单一NALU，1：非交错模式（允许单个NALU或分片），2：交错模式（支持NALU分片交错）。profile-level-id：H.264档次和级别。sprop-parameter-sets：序列参数集SPS，图像参数集PPS的Base64编码后的值，用逗号分隔
> * a=control:track1：该媒体流的控制URL后缀，用于SETUP和后续的PLAY，需要注意的是如果a=control:*那么就是说匹配基地址，后续PLAY可以使用base url一次启动该会话下所有媒体流

**注意** ：由于服务端可能提供多个流服务，比如视频流，音频流，字幕流，数据流，为了在SETUP这里区分出不同流，我们需要在DESCRIBE这里和客户端协商好不同流的请求url，这样服务端就可以根据控制URL的不同做出不同的行为

#### SETUP

##### UDP

SETUP请求示例（UDP）

```
SETUP rtsp://192.168.1.100:554/stream/track1 RTSP/1.0
CSeq: 3
Transport: RTP/AVP/UDP;unicast;client_port=1234-1235
User-Agent: VLC/3.0.18
```

SETUP请求行的URL是由上一步DESCRIBE中根据提取出的a=control结合Content-Base基地址组合而来的。

Transport作为SETUP请求核心，字段解析如下

> * RTP/AVP/UDP：表示使用RTP over UDP，传输层使用UDP协议，应用层使用RTP协议
> * unicast：单播模式（multicast表示多播模式）
> * client_port=1234-1235：客户端用来接收RTP和RTCP的UDP端口对（偶数为RTP端口，奇数为RTCP端口）、

既然Transport中有指定传输层协议为UDP，那么我们就立即可以得到RTSP协议同时支持TCP传输



SETUP响应示例（UDP）

```
RTSP/1.0 200 OK
CSeq: 3
Transport: RTP/AVP/UDP;unicast;client_port=1234-1235;server_port=5678-5679;ssrc=0x12345678
Session: 12345678;timeout=60
Date: Wed, Apr 20 2026 10:30:01 GMT
```

Session字段是服务器分配的会话ID，以及可选的超时时间（超过这个时间无活动则销毁会话），SETUP一旦建立完成，后续客户端的所有请求都必须在请求头中携带Session这个参数。Session是服务器为每个客户端会话分配一个全局唯一的 ID。

Transport字段

> * RTP/AVP/UDP：服务端回显确认
> * unicast：服务端回显确认
> * client_port=1234-1235：服务端回显确认
> * server_port=5678-5679：服务端分配的UDP端口号，偶数作为RTP端口，奇数作为RTCP端口
> * ssrc=0x12345678：同步源标识符（32位十六进制），用于标识RTP流

**注意** ：ssrc用于在RTP会话中唯一标识一个数据源，假如说在多人视频会议的场景下，同一个RTP会话会有多个不同的贡献者，所以接收端必须要区分开这些不同的贡献源流。怎么区分呢？后续RTP包的头部会有一个ssrc字段值

##### TCP

SETUP请求示例（TCP）

```
SETUP rtsp://192.168.1.100/stream/track1 RTSP/1.0
CSeq: 3
Transport: RTP/AVP/TCP;unicast;interleaved=0-1
```

interleaved：交织信道，由于SETUP协商传输层使用TCP协议，所以还是复用该TCP连接，但是TCP连接只有一条，所以为了区分RTP数据和RTCP数据，请求方需要和响应方协商RTP和RTCP的信道号，RTP使用信道0，RTCP使用信道1。

开始play之后，每个RTP或RTCP包在发送之前会被加上4字节标识头，0x24 channel length然后紧跟着RTP头或RTCP头，再就是RTP负载或RTCP负载



SETUP响应示例（TCP）

```
RTSP/1.0 200 OK
CSeq: 3
Transport: RTP/AVP/TCP;unicast;interleaved=0-1;ssrc=0x12345678
Session: 87654321;timeout=60
```

#### PLAY

PLAY请求示例

##### 基础请求

```
PLAY rtsp://192.168.1.100:554/stream RTSP/1.0
CSeq: 4
Session: 12345678
User-Agent: VLC/3.0.18
```

请求行：PLAY方法+Base URL+RTSP version

请求头：

Session：某个流的会话id

表示从头开始播放所有流

**注意** ：如果这里的url不是base url而是控制url，那么只播放控制url对应的那路流

##### 带时间范围参数请求

```
PLAY rtsp://192.168.1.100:554/stream RTSP/1.0
CSeq: 4
Session: 12345678
Range: npt=10.000-
```

Range：从第10秒开始播放

##### 带播放速度请求

```
PLAY rtsp://192.168.1.100:554/stream RTSP/1.0
CSeq: 4
Session: 12345678
Scale: 2.0
```

Scale；2倍速播放



##### 响应

###### 非聚合控制

```
RTSP/1.0 200 OK
CSeq: 4
Session: 12345678
Range: npt=0.000-
RTP-Info: url=rtsp://192.168.1.100/stream/track1;seq=12345;rtptime=987654321
Date: Wed, Apr 20 2026 10:30:02 GMT
```

Range：服务器回显或修正的实际播放范围

RTP-Info：

> * url：对应的RTP流URL
> * seq：即将发送的第一个RTP包的序列号
> * rtptime：第一个RTP包的时间戳（基于SDP中a=rtpmap:96 H264/90000这里的90000也就是90kHz时钟）

###### 聚合控制

```
RTSP/1.0 200 OK
CSeq: 4
Session: 12345678
Range: npt=0.000-
RTP-Info: url=rtsp://192.168.1.100/stream/track1;seq=12345;rtptime=987654321,
          url=rtsp://192.168.1.100/stream/track2;seq=54321;rtptime=123456789
Date: Wed, Apr 20 2026 10:30:02 GMT
```

这里就返回两路流的RTP-Info

#### PAUSE

PAUSE请求示例

###### 非聚合

```
PAUSE rtsp://192.168.1.100/stream/track1 RTSP/1.0
CSeq: 5
Session: 12345678
```

url是控制url

###### 聚合

```
PAUSE rtsp://192.168.1.100/stream RTSP/1.0
CSeq: 5
Session: 12345678
```

url使用base url

PAUSE响应示例

```
RTSP/1.0 200 OK
CSeq: 5
Session: 12345678
Date: Wed, Apr 20 2026 10:30:05 GMT
```

**注意** ：PAUSE只是临时暂停媒体流，保留会话资源

#### TEARDOWN

终止整个会话

TEARDOWN请求示例

```
TEARDOWN rtsp://192.168.1.100/stream RTSP/1.0
CSeq: 6
Session: 12345678
```

使用base url终止整个会话



TEARDOWN响应示例

```
RTSP/1.0 200 OK
CSeq: 6
Session: 12345678
Date: Wed, Apr 20 2026 10:30:10 GMT
```

### 2、RTP协议

##### RTP头

![](https://raw.githubusercontent.com/hcc123231/pictures/main/rtp%20fmt.png)

- **V (Version, 版本号)**: 2 bits，表示 RTP 的版本。目前的标准是 2。
- **P (Padding, 填充标志)**: 1 bit，标记数据包的末尾是否包含填充字节，这些字节不属于有效载荷。
- **X (Extension, 扩展标志)**: 1 bit，标志固定头部后面是否跟随了一个扩展头部。
- **CC (CSRC Count, 特约信源计数器)**: 4 bits，指示跟在标准头部后面的 CSRC 标识符的数量。
- **M (Marker, 标记位)**: 1 bit，一个重要的边界标志。对于视频，它常被设置在一帧数据的最后一个 RTP 包上，表示一个“完整帧”的结束。
- **PT (Payload Type, 有效载荷类型)**: 7 bits，用来告诉接收端“这是什么格式的数据”。对于 H.264 这类较新的编码，这个值通常使用 96-127 之间的动态值，具体含义在之前的 SDP 协商中就已确定。
- **Sequence Number (序列号)**: 16 bits，这是 RTP 的重要标志之一。**每发送一个 RTP 包，序列号就自动加 1**。接收方利用它来检测网络传输中发生的丢包，并能对乱序到达的包进行重新排序。
- **Timestamp (时间戳)**: 32 bits，这是 RTP 的另一大核心标志。它记录的是该 RTP 包中**第一个字节数据的采样时刻**，是接收端实现**同步播放和计算网络抖动**的关键依据。对于视频，时间戳以一个固定的 90kHz 时钟频率递增。
- **SSRC (Synchronization Source identifier, 同步源标识符)**: 32 bits，用于唯一标识一个 RTP 数据流的“源头”。在一个会话中，每个发送源（比如一个摄像头）的 SSRC 都应是随机的且唯一的。
- **CSRC (Contributing Source identifiers, 特约信源标识符)**: 0-15 项，每项 32 bits。当数据包经过 RTP 混频器（Mixer）等设备时，CSRC 列表用来标识**哪些源的数据共同组成了这个包**

##### RTP负载

由于H.264编码器输出单元是NALU，而NALU的大小是不确定的，有的极小，有的极大，所以在打包RTP数据包时可能会出现单一NALU包和分片单元包（将一个NALU切分成多个分片，分别装入RTP包中进行发送），聚合包（将多个NALU单元放到同一个RTP包中进行发送）

NALU在被编码器输出的时候就自带一个字节的NALU头，如下

> * F：1bit，应为0
> * NRI：2bit，00表示不用于参考，非00表示用于参考
> * Type：5bit，NAL单元类型



如果NALU包太大，那么RTP协议在封装RTP包时会主动对这个大包进行分片

RTP协议会申请一个字节FU indicator和一个字节FU header，然后复制原NALU头的前两个字段到FU indicator的前两个字段中，将28写入到FU indicator的type字段，然后根据当前上下文决定FU header的前两个字段值，然后再将原NALU头的type字段值拷贝到FU header的type字段

内存布局如下

RTP header|FU indicator|FU header|payload

> * S：1bit，该分片是否为源NALU的第一个分片
> * E：1bit，该分片是否为最后一个分片
> * R：0
> * Type：5bit，原NALU类型，被分片之前那个完整的NALU的类型

### 3、RTCP协议



### 4、RTMP协议

#### rtmp消息头

> rtmp协议并没有传统意义上的rtmp头，而是经过分块器分块之后添加chunk header，在chunk header之前还有一个basic header

#### 网络建立

> rtmp是基于tcp的应用层传输协议，通信第一步就是建立tcp连接

#### 握手

客户端发出C0包（1字节，固定版本号，当前为0x3）========>> 服务端响应S0（1字节，版本号）========>> 客户端发出C1（1536字节，4字节时间戳+4字节全0字段+1528字节随机数据）========>> 服务端响应S1（1536字节，4字节时间戳+4字节全0字段+1528字节随机数据）========>> 客户端发出C2（1536字节，S1中时间戳+C1时间戳+S1中1528字节随机数据副本）========>> S2（1536字节，C1时间戳+S1时间戳+C1随机数据）

#### 命令交互

##### AMF0编码格式

AMF0是一种二进制数据格式，采用**类型 - 长度(部分有) - 值**的形式

###### 类型标记

| Marker | 数据类型   | 说明                                                         |
| :----- | :--------- | :----------------------------------------------------------- |
| `0x00` | Number     | 8字节双精度浮点数                                            |
| `0x01` | Boolean    | 1字节，`0x01`为`true`，`0x00`为`false`                       |
| `0x02` | String     | 2字节长度（网络字节序）+ UTF-8字符串数据                     |
| `0x03` | Object     | 一系列键值对，以`String`为键，任意类型为值，最后以`0x00 0x00 0x09`结束 |
| `0x05` | Null       | 无数据                                                       |
| `0x08` | ECMA Array | 4字节数组长度 + 键值对列表 + `0x00 0x00 0x09`                |

**注意**：Object是键值对形式出现，键固定是字符串，必须在前面标记出键长，值是任意AMF0类型

##### connect

> connect：命令名("connect")+事务ID(1)+命令对象(Object)+可选参数

###### 协议控制消息

在响应**_result**之前需要先回发协议控制消息，常见协议控制消息如下：

| 消息名称                        | 类型ID | 作用                                           | 载荷格式                                  |
| :------------------------------ | :----- | :--------------------------------------------- | :---------------------------------------- |
| **Set Chunk Size**              | 1      | 通知对端使用新的最大块大小                     | 4字节：新块大小（无符号32位整数）         |
| **Abort Message**               | 2      | 通知对端中止正在处理的消息                     | 4字节：要中止的块流ID（CSID）             |
| **Acknowledgement**             | 3      | 对端收到一定字节数后的确认                     | 4字节：已接收的总字节数（无符号32位）     |
| **User Control**                | 4      | 发送用户事件（如流开始、流结束、缓冲区设置等） | 2字节事件类型 + 可变事件数据              |
| **Window Acknowledgement Size** | 5      | 设置接收窗口大小                               | 4字节：窗口大小（无符号32位）             |
| **Set Peer Bandwidth**          | 6      | 告知对端自己的带宽限制                         | 5字节：4字节带宽 + 1字节限制类型（0/1/2） |

消息由 类型ID+载荷组成

###### _result

服务器响应**_result**，**_result**：命令名（"_result"）+事务ID(1)+properties(object,描述服务器能力)+information(object,连接结果状态信息)

##### createStream

> createStream：命令名（"createStream"）+事务ID(2)+命令对象（无附加信息可填Null）

###### 响应成功

> **_result**：命令名（"_result"）+事务ID(2)+命令对象（无附加信息，可填Null）+流ID（服务器分配的流标识）

###### 响应失败

> **_error**：命令名（"_error"）+事务ID(2)+命令对象（无附加信息，可填Null）+错误描述对象

##### 推拉流命令交互

###### 推流

> publish：命令名（"publish"）+事务ID（0）+命令对象（Null）+流名（String类型）+发布类型（"live"/"record"/"append"）

###### 推流响应

> onStatus：命令名（"onStatus"）+事务ID（0）+命令对象（Null）+信息对象（Object类型）

###### 拉流

> play：命令名（"play"）+事务ID（0）+命令对象（Null）+流名（String类型）+起始位置（Number）+时长（Number）

###### 拉流响应

服务端会先响应一个协议控制消息，同上

然后再响应onStatus

> onStatus：命令名（"onStatus"）+事务ID（0）+命令对象（Null）+信息对象（Object类型）

##### 推流

> ```
> ┌─────────────────────────────────────────────────────────────────────────────┐
> │ 编码器输出（Annex B）                                                         │
> │ [起始码] [NALU Header] [NALU Payload] + [起始码] [NALU Header] [NALU Payload] │
> └─────────────────────────────────────────────────────────────────────────────┘
>                                       │
>                                       ▼
> ┌─────────────────────────────────────────────────────────────────────────────┐
> │ Annex B → AVCC 转换                                                          │
> │ • 提取 SPS/PPS → 构建 AVCDecoderConfigurationRecord（Sequence Header）        │
> │ • 普通 NALU：移除起始码，添加4字节NALU长度                                      │
> └─────────────────────────────────────────────────────────────────────────────┘
>                                       │
>                                       ▼
> ┌─────────────────────────────────────────────────────────────────────────────┐
> │ FLV Video Tag Body 封装                                                      │
> │ [Frame Type(4) + Codec ID(4)] + [AVCPacketType] + [Composition Time] + [Data]│
> └─────────────────────────────────────────────────────────────────────────────┘
>                                       │
>                                       ▼
> ┌─────────────────────────────────────────────────────────────────────────────┐
> │ RTMP 消息（Message）                                                          │
> │ ┌─────────────────────┬─────────────────────────────────────────────────────┐│
> │ │ 消息头（Message Header） │ 消息体 = FLV Video Tag Body                        ││
> │ │ • 消息类型ID = 9      │                                                     ││
> │ │ • 消息长度            │                                                     ││
> │ │ • 时间戳              │                                                     ││
> │ │ • 消息流ID            │                                                     ││
> │ └─────────────────────┴─────────────────────────────────────────────────────┘│
> └─────────────────────────────────────────────────────────────────────────────┘
>                                       │
>                                       ▼
> ┌─────────────────────────────────────────────────────────────────────────────┐
> │ 分块器（Chunking）                                                            │
> │ 消息被拆分成多个Chunk，每个Chunk添加：                                          │
> │ [Basic Header] + [Chunk Message Header] + [可选Extended Timestamp] + [Chunk Data] │
> └─────────────────────────────────────────────────────────────────────────────┘
>                                       │
>                                       ▼
>                                   TCP 发送
> ```

> 推流端拿到编码器输出，封装成flv tag格式作为消息体，然后根据RTMP协议栈动态构造出消息头，之后根据chunk_size将消息体分块，为每一个分片构造一个chunk,chunk由basic header+message header+chunk data组成，message header就是消息头，chunk data就是分片数据，
>
> flv和flv tag完全是两个概念，flv是一种容器封装格式，而flv tag才是帧封装格式，flv tag封装分为两部分，第一部分是tag header，第二部分是tag data，
>
> 而这个tag data又由两个部分构成，第一部分是一个字节元数据头，第二部分是原始载荷数据。

##### 拉流

根据上述推流格式解析得到flv video tag body即可

## 四、码率预估

### 背景

> 在直播场景下，发送方将编码数据通过网络发送给接收方，这个过程离不开网络，理想情况下的网络是不丢包，没有网络延迟的，这样发送方将恒定编码后的
