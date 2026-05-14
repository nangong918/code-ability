# 麒麟功能



## 设备操控
LED，LCD点屏拼接，设置连接关系，预置场景操控等。

## 预监
监控RK内部的流媒体清空，用于多用户。
拉内部流播放，拉RK内部梯控的RTSP，RTMP在App上播放。

## 局域网视频播放点屏
RK在局域网内通过MediaMTX创建RTSP url，麒麟App通过url，使用FFmpeg + RTSP 将本地的视频文件播放在屏幕上。

## 云上
* 文件推流：麒麟App通过FFmpeg + RTSP将本机的视频文件推流到云上的MediaMTX服务器，RK拉流交给播控程序最后交给FPGA播放。
* 直播：连接采集设备：
  - 采集设备会给出YUV + PCM原始数据 -> 麒麟将原始数据编码成(H.264 + AAC)封包成RTMP发送给云上的MediaMTX服务器 -> RK拉流交给播控程序最后交给FPGA播放。
  - 已经封装好的RTMP数据 -> 透传给云上的MediaMTX服务器 -> RK拉流交给播控程序最后交给FPGA播放


