---
layout:     post
title:      WebRTC 源码分析 二 本地 Candidate 收集
date:       2021-05-20
author:     xflyme
header-img: img/post-bg-2021-05-20.png
catalog: true
tags:
    - 源码分析
    - WebRTC
---


* 本文基于 WebRTC M89

![图一](/img/webrtc-2-1.png)

上篇文章简要的看了一下视频从采集到发送到网络的整个 pipeline。从本篇开始分析从用户进入房间到成功建立 P2P 连接收发数据的过程。由于信令服务不是 WebRTC 的一部分，本文将从 Candidate 收集开始。

![图二](/img/webrtc-2-2.png)

#### Candidate 分类
Candidate 分为四类
* host
* srflx
* relay
* prflx

其中是经过对端反射得到的 Candidate，不在本文讨论之列。

#### PortConfiguration
PortConfiguration 中有两个重要成员 stun_servers 和 relays，其中分别记录了可用的 Stun Server 和 Turn Server 的地址。它们用于在收集阶段探测 srflx candidate 和创建 relay candidate 。

```c++
struct RTC_EXPORT PortConfiguration : public rtc::MessageData {
  rtc::SocketAddress stun_address;
  ServerAddresses stun_servers;
  typedef std::vector<RelayServerConfig> RelayList;
  RelayList relays;
};

```


#### ALLOCATION PHASE
针对每个 NetWork Candidate 收集分为三个阶段：PHASE_UDP、PHASE_RELAY 和 PHASE_TCP 。

##### PHASE UDP
PHASE_UDP 中又分为 host 和 srflx ：

```
case PHASE_UDP:
      CreateUDPPorts();
      CreateStunPorts();
      break;
```

Create*Ports 是创建不同类型的 Port，Port 创建完成之后会调用 port->PrepareAddress()。
UDPPort 添加 Local Address 之后会一路回调到 PeerConnection:OnIceCandidate 完成 Candidate 添加。而 StunPort 还要与 StunServers 通信完成 Candidate 收集。

##### PHASE RELAY 
略

##### PHASE TCP 
略












































