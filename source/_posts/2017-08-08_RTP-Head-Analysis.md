title: RTP头字段参数解析
date: 2017-08-08 12:51:06
tags: [C,VoLTE,RTP]
categories: [Blog]
---
## 概述

最近在做VoLTE的数据包分析，发现了一些现有的资料中描述的不太清楚的地方，现将我的个人感受整理出来，供大家批评指正。

VoLTE是Voice over Long Term Evolution的缩写，也就是基于LTE网络的语音通信方案。因为VoLTE的架构设计非常的复杂，还涉及到基站间链路交换、IPv6地址的分配等等，此处不再详述。

在我们的测试中，采用Samsung A5108移动定制机作为测试终端，TCPDUMP作为抓包工具。得到的结果在WireShark中进行分析。

<!-- more -->

## VoLTE包结构

VoLTE的数据包主要分为以下几类：

1. ** 通话控制包 **
2. ** RTCP-传输控制包 **
3. ** RTP数据包 **

### 通话控制包

在拨号和通话建立时，双方会发出一系列ESP协议包，包中包含了主叫、被叫、IP地址、支持的编码格式等等信息。

不过，需要注意的是，在WireShark中，ESP包的内容是不会解析的。而如果我们需要了解双方的IP、电话号码或者所在基站，则需要换一种方式。

用文本编辑器，比如NotePad++或者Edit Plus。取决于抓包文件的大小。如果是小文件，可以用NotePad++打开，但如果文件过大，就需要用Edit Plus了。Edit Plus的好处，就是具有64位版本，在数据处理能力上略胜一筹。

![](/images/2017-08-08/capture-volte-head.jpg)

在开始，可以看见邀请消息“INVITE”，以及双方的电话信息、IPv6地址、所属网络等身份信息。

![](/images/2017-08-08/capture-volte-voice-support.jpg)

在下面的数据中，可以看到不同语音编码方式，这和WireShark分析得到的PT类型是对应的。并且，不同类型的采样频率，是计算RTP时间戳的重要依据。

## RTP包结构

RTP包是通过UDP协议进行传输的，在UDP的负载中，包含着RTP的数据帧。

![](/images/2017-08-08/capture-rtp-header.jpg)

    version (V): 2 bits
      This field identifies the version of RTP.The version defined by
      this specification is two (2).
      RTP的版本号，1和0有其他定义


    padding (P): 1 bit
      If the padding bit is set, the packet contains one or more
      additional padding octets at the end which are not part of the
      payload.The last octet of the padding contains a count of how
      many padding octets should be ignored, including itself.
      是否在包的末尾进行填充，如果进行了填充，则其不属于负载的一部分。用于某些特定的加密算法


    extension (X): 1 bit
      If the extension bit is set, the fixed header MUST be followed by
      exactly one header extension, with a format defined below.
      
![](/images/2017-08-08/capture-rtp-header-extension.jpg)
      
      RTP头是否进行了拓展，否则必须按照特定的格式进行。


    CSRC count (CC): 4 bits
      The CSRC count contains the number of CSRC identifiers that follow
      the fixed header.
      
      
    marker (M): 1 bit
      The interpretation of the marker is defined by a profile.  It is
      intended to allow significant events such as frame boundaries to
      be marked in the packet stream.  A profile MAY define additional
      marker bits or specify that there is no marker bit by changing the
      number of bits in the payload type field.
      帧的分隔，接收到带有marker标记的包，意味着一个数据帧的结束。


    payload type (PT): 7 bits
      This field identifies the format of the RTP payload and deand ded 
      de dedeeeand ded de dedee by the application.  
      如前文提到的数据编码方式，不同的数据类型在此字段有体现。


    sequence number: 16 bits
      The sequence number increments by one for each RTP data packet
      sent, and may be used by the receiver to detect packet loss and to
      restore packet sequence.  The initial valal vall valvalallhe sequence number
      SHOULD be random (unpr to make known-plaintext attacks
      on encryption more difficult, even if the source itself does not
      encrypt according to the method in Section 9.1, because the
      packets may flow through a translator that does. 
      包的序号，每次选用一个随机的数字，防止已知明文攻击。


    timestamp: 32 bits
      The timestamp reflects the sampling instant of the first octet in
      the RTP data packet.  The sampling instant MUST be derived from a
      clock that increments monotonically and linearly in time to allow
      synchronization and jitter calculations.
      RTP头中一个非常重要的字段，可以通过时间戳的变化计算还原一帧的采样时间。
      具体的计算方法见下方。
      
    SSRC: 32 bits
      The SSRC field identifies the synchronization source.  This
      identifier SHOULD be chosen randomly, with the intent that no two
      synchronization sources within the same RTP session will have the
      same SSRC identifier. 

    CSRC list: 0 to 15 items, 32 bits each
      The CSRC list identifies the contributing sources for the payload
      contained in this packet.  The number of identifiers is given by
      the CC field.
      
      
## RTP时间戳计算

因为RTP在传输的过程中，会对一帧数据进行分包，导致不同的数据包具有相同的TimeStamp。在进行时间还原时，必须跟踪具有不同时间戳的包才具有还原的意义。

正如之前提到的，RTP Header中的marker代表着一个数据帧的结束，也代表着接下来的数据包将会有不同的TimeStamp值。

因为RTP的时间戳增量是按照采样时间递增的，所以必须要知道此次传输的数据帧格式。在RTP包格式中，提到过PT字段。PT字段代表着RTP包所传输的数据类型，在我们的VoLTE视频测试中，得到的PT值是115。

得到PT值之后，需要对照通话控制包中的格式及帧率的对应关系，用来进一步计算。

以我们的测试为例，对于数据包i和j，具有不同的TimeStamp值*T<sub>i</sub>*和*T<sub>j</sub>*。
PT类型115对应的时钟频率为*f* = 90000 Hz，视频帧率为*fps* = 30 fps。

那么，真实的数据帧时间间隔是多少呢？

*ΔT* ＝ (*T<sub>i</sub>* - *T<sub>j</sub>*) / (*f* / *fps*) * (1 / *fps*) (s)

时间戳的默认递增长度，是时钟频率/帧率，但在我们的测试中，发现其递增规律，并不是严格的3000，而是会在一定范围内浮动。通过数学计算，发现：虽然存在浮动，但仍然存在严格的比例，即TimeStamp的3000对应真实的1/30秒。