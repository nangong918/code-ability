

### 项目介绍



#### Ijetty

[IJetty.md](CLT/IJetty/IJetty.md)
做过RK芯片上的 为上位机提供Http服务的 App：Ijetty
前端下发Json到Ijetty，IJetty通过JNI将数据转化为RK芯片接受的V协议（字节帧）。
接收RK的主动上推V协议帧并通过WebSocket将消息推送给前端。
实现Web前端对上位机的调用服务。


#### Cross

[Cross.md](CLT/Cross/Cross.md)
做过RK芯片上的 双机备份 和 集群协同 App：cross
cross在集群之间通过UDP广播discover对方，
双机备份：开启后同步双机配置，并且主机出问题掉线由背脊顶上。
集群协同：主机操控整个协同组，主机的行为在备机中也要做到同步操作。

#### Uber
[Uber.md](CLT/Uber/Uber.md)
做过云上操控上位机的RK芯片App：Uber。
硬件设备通过Uber链接互联网，
SpringBoot通过设备Mac地址映射id访问设备，并对设备发送指令。
设备将指令通过AIDL直接调用Ijetty。

#### 麒麟
[QiLing.md](CLT/QiLing/QiLin.md)
操控上位机的Flutter App：跟Ijetty服务器通信，可操控瓶组，拼接设置，连接关系等。
开发RTMP拉取预监板卡的音视频流，并在麒麟App上展示LED/LCD大屏幕上的设备状态。

#### 灵境
[LingJing.md](CLT/LingJing/LingJing.md)
做过卡莱特AI智能语音助手，通过科大讯飞的语音唤醒SDK，VAD语音活动检测，LLM大模型，
STT语音识别，TTS语音合成，VL视觉理解模型实现：
唤醒语音助手，并对其下发语音/视觉指令，让其调用预制场景与操控场设备。

#### CA前面板
[CA.md](CLT/CA/CA.md)
[Framework.md](CLT/Framework.md)
操控设备的前面板App：功能与Ijetty类似，图形化界面操控上位机。
也是直接组装V协议发送给RK调用。
Android Framework定制：系统App，包括：AMS，PMS，WMS定制（开机自动启动前面板，进制下拉，待机动画等）

### 云上（MAC地址烧录）
[Mac.md](CLT/Mac/Mac.md)
基于SpringBoot开发MAC地址集中管理平台，实现MAC地址的生成分配、设备绑定、一键烧录功能，
与Uber云端控制设备联动，采用Docker容器化部署云端服务。



### 项目问题









