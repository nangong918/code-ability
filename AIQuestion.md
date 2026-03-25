

### 项目问题


设备开发：
* 多机协同`Cross`
   * 双机备份
   * 主控协同
   * 集群同步协同

* 云平台接入`Uber`
   * SpringBoot控制RK上的CloudApp；CloudApp通过AIDL跟Ijetty交互

* XM项目`CA`前面板的显示，用FFmpeg

* `Ijetty`RK操控服务器

* `CLT语音助手`

* 预监板音频流传输

* 麒麟设备操控Flutter App： RTMP拉流播放



#### Ijetty

做过RK芯片上的 为上位机提供Http服务的 App：Ijetty
前端下发Json到Ijetty，IJetty通过JNI将数据转化为RK芯片接受的V协议（字节帧）。
接收RK的主动上推V协议帧并通过WebSocket将消息推送给前端。
实现Web前端对上位机的调用服务。


#### Cross

做过RK芯片上的 双机备份 和 集群协同 App：cross
cross在集群之间通过UDP广播discover对方，


#### Uber

做过云上操控上位机的RK芯片App：Uber。
硬件设备通过Uber链接互联网，
SpringBoot通过设备Mac地址映射id访问设备，并对设备发送指令。
设备将指令通过AIDL直接调用Ijetty。

#### 麒麟

操控上位机的Flutter App：跟Ijetty服务器通信，可操控瓶组，拼接设置，连接关系等。
开发RTMP拉取预监板卡的音视频流，并在麒麟App上展示LED/LCD大屏幕上的设备状态。

#### 灵境

做过卡莱特AI智能语音助手，通过科大讯飞的语音唤醒SDK，VAD语音活动检测，LLM大模型，
STT语音识别，TTS语音合成，VL视觉理解模型实现：
唤醒语音助手，并对其下发语音/视觉指令，让其调用预制场景与操控场设备。

#### CA前面板

操控设备的前面板App：














