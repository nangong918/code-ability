# 基本功能



## 双机备份

主备配置必须完全一致；

* 画面源：单屏幕，数据源为主机

* 系统鲁棒性
    - 备机掉线：弹窗提示并关闭主备状态，重新登录。
    - 主机掉线：主机的Web（eg：192.168.1.10）页面与主机的Cross长连接通讯丢失，从缓存获取到缓存的双机备份组，拿到备机IP（eg：192.168.1.11）并跳转到备机Web页面。
      同时屏幕的信号源也切换为备机的信号源。
      如果组播检测到主机上线则切回主备模式，切回主机Web（192.168.1.10）

* 异常情况：
    - 主备机硬件板卡布局，数据不一致：弹窗提示


活动图：
```mermaid
flowchart TD
    subgraph 主备正常运行
        N[主备模式运行中<br/>Web在主机]
    end

    subgraph 备机掉线
        N --> B1[备机掉线]
        B1 --> B2[弹窗提示<br/>关闭主备状态]
        B2 --> B3[主机单机运行]
    end

    subgraph 主机掉线与恢复
        N --> H1[主机掉线]
        H1 --> H2[Web与主机Cross断连]
        H2 --> H3[从缓存取备机IP<br/>跳转备机Web]
        H3 --> H4[信号源切为备机]
        H4 --> H5[备机接管运行]
        
        H5 --> H6[组播检测到主机上线]
        H6 --> H7[弹窗提示主机已上线<br/>是否同步数据?]
        H7 -->|用户选择| H8[跳转主机Web<br/>重新登录]
        H8 --> H9[信号源切回主机]
        H9 --> N
    end

    subgraph 异常
        N --> E1[硬件板卡/数据不一致]
        E1 --> E2[弹窗提示]
    end

    style N fill:#c8e6c9,stroke:#2e7d32
    style B2 fill:#fff3e0,stroke:#ef6c00
    style H7 fill:#e8eaf6,stroke:#3949ab
    style E2 fill:#ffcdd2,stroke:#c62828
```

## 多机协同
主机和协同组的配置不必要一致。

* 画面源：多屏组，数据源为协同组所有机器
* 画面职责：协同组画面一致

* 多机协同方案：

  - 局域网方案：
    - 基于UDP组播发现设备
    - 主机通过TCP Socket单播建立协同组
    - 指令通过HTTP转发，WS长连接推送状态
    - 适用场景：同一局域网内多台设备

  - 云上方案：
    - 基于云服务器中转，解决跨网段远距离协同
    - 设备通过WebSocket长连接注册到云服务器
    - 主机指令经云服务器转发到各协同机
    - 适用场景：异地多屏，无法组建局域网

* 流媒体传输：
  - 实时直播场景：管理员通过OBS/FFmpeg推送RTMP流到云服务器
  - 各协同机从云服务器拉取同一路RTMP流播放
  - 画面大致同步，无需帧级精确对齐
  - 控制指令（切换源、音量等）通过Cross的WS长连接下发
  - RTMP优势：推流生态成熟，OBS/FFmpeg直接可用，延迟1-3秒

本地局域网活动图：
```mermaid
flowchart TD
  subgraph 前端
    Web[Web前端]
  end

  subgraph 主机
    H_Cross[主机 Cross]
    H_IJetty[主机 IJetty]
  end

  subgraph 协同机1
    C1_IJetty[协同机1 IJetty]
  end

  subgraph 协同机2
    C2_IJetty[协同机2 IJetty]
  end

  subgraph 信号源
    VS[RTMP视频源<br/>管理员OBS推流]
  end

  subgraph 组播
    M[组播域]
  end

  H_Cross -->|组播心跳| M

  Web -->|请求| H_Cross

  H_Cross -->|POST 广播| H_IJetty
  H_Cross -->|POST 广播| C1_IJetty
  H_Cross -->|POST 广播| C2_IJetty

  H_Cross -->|GET| H_IJetty
  H_Cross -->|GET| C1_IJetty
  H_Cross -->|GET| C2_IJetty

  H_IJetty -->|WS| H_Cross
  C1_IJetty -->|WS| H_Cross
  C2_IJetty -->|WS| H_Cross

  H_Cross -->|转发全部WS| Web

  VS -->|拉RTMP流| H_IJetty
  VS -->|拉RTMP流| C1_IJetty
  VS -->|拉RTMP流| C2_IJetty

  H_IJetty --> P1[画面1]
  C1_IJetty --> P2[画面2]
  C2_IJetty --> P3[画面3]

  P1 --> S[画面一致]
  P2 --> S
  P3 --> S

  style H_Cross fill:#c8e6f5,stroke:#0066cc
  style H_IJetty fill:#c8e6f5,stroke:#0066cc
  style C1_IJetty fill:#e8d5f5,stroke:#7b1fa2
  style C2_IJetty fill:#e8d5f5,stroke:#7b1fa2
  style VS fill:#ffcdd2,stroke:#c62828
  style P1 fill:#fff9c4,stroke:#f9a825
  style P2 fill:#fff9c4,stroke:#f9a825
  style P3 fill:#fff9c4,stroke:#f9a825
  style S fill:#c8e6c9,stroke:#2e7d32
```

云上活动图：
```mermaid
flowchart TD
  subgraph 主机侧 - 深圳
    H_Web[Web前端]
    H_Cross[主机 Cross]
    H_IJetty[主机 IJetty]
  end

  subgraph 云服务器
    Cloud[云协同服务<br/>WebSocket设备注册/指令转发<br/>RTMP流媒体分发]
  end

  subgraph 信号源
    VS[管理员OBS推流<br/>RTMP推流到云]
  end

  subgraph 协同机1 - 北京
    C1_Cross[协同机1 Cross]
    C1_IJetty[协同机1 IJetty]
  end

  subgraph 协同机2 - 上海
    C2_Cross[协同机2 Cross]
    C2_IJetty[协同机2 IJetty]
  end

  H_Cross -->|WebSocket注册| Cloud
  C1_Cross -->|WebSocket注册| Cloud
  C2_Cross -->|WebSocket注册| Cloud

  VS -->|RTMP推流| Cloud

  H_Web -->|请求| H_Cross
  H_Cross -->|POST/GET指令| Cloud

  Cloud -->|转发指令| C1_Cross
  Cloud -->|转发指令| C2_Cross

  C1_Cross -->|执行| C1_IJetty
  C2_Cross -->|执行| C2_IJetty

  C1_IJetty -->|WS状态| C1_Cross
  C2_IJetty -->|WS状态| C2_Cross

  C1_Cross -->|状态上报| Cloud
  C2_Cross -->|状态上报| Cloud
  Cloud -->|状态推送| H_Cross
  H_Cross -->|推送| H_Web

  Cloud -->|拉RTMP流| H_IJetty
  Cloud -->|拉RTMP流| C1_IJetty
  Cloud -->|拉RTMP流| C2_IJetty

  H_IJetty --> P1[画面1]
  C1_IJetty --> P2[画面2]
  C2_IJetty --> P3[画面3]

  P1 --> D[多屏组<br/>画面一致]
  P2 --> D
  P3 --> D

  style H_Cross fill:#c8e6f5,stroke:#0066cc
  style H_IJetty fill:#c8e6f5,stroke:#0066cc
  style Cloud fill:#d4f1d4,stroke:#2e7d32
  style VS fill:#ffcdd2,stroke:#c62828
  style C1_Cross fill:#e8d5f5,stroke:#7b1fa2
  style C2_Cross fill:#e8d5f5,stroke:#7b1fa2
  style C1_IJetty fill:#e8d5f5,stroke:#7b1fa2
  style C2_IJetty fill:#e8d5f5,stroke:#7b1fa2
  style P1 fill:#fff9c4,stroke:#f9a825
  style P2 fill:#fff9c4,stroke:#f9a825
  style P3 fill:#fff9c4,stroke:#f9a825
  style D fill:#c8e6c9,stroke:#2e7d32
```

## 分布式集群

主机和分布式集群配置无需一致。

* 画面源：多屏组，数据源为分布式集群所有机器。
* 画面职责：各设备负责大屏不同区域，需配置连接关系（主机Web通过拖拽的方式配置）。
* 建立方式：主机通过TCP Socket单播/云WebSocket向各节点发起建立请求，
  请求中携带该节点在大屏的position信息。
* 位置配置：组播心跳/云注册信息中deviceGroup扩展position字段，
  描述每台设备在大屏的行列位置。
* 指令分发：主机根据各节点position下发不同区域画面参数。

* 部署方案：
  - 局域网方案：同一机房/展厅，交换机直连，低延迟
  - 云上方案：跨楼宇/跨江，每栋楼RK设备通过4G/5G/光纤注册到云服务器

* 云上分布式集群：
  - 各节点通过WebSocket长连接注册到云服务器
  - 主机通过云服务器下发position + 同步指令
  - 各节点独立拉流 → 按position裁剪 → HDMI输出LED屏

* 流媒体传输：
  - 信号源：
    - 单一信号源：所有节点拉取同一路视频流（RTSP/HLS）
    - 多信号源：各节点根据配置拉取不同视频流（RTSP/HLS）
  - 主机通过云服务器广播当前播放帧号/时间戳
  - 各节点seek到对应帧后裁剪输出，保证拼接后画面完整同步
  - 帧同步方案：
    - 各节点NTP校时，保证时间基准一致
    - 主机下发playAt指令（绝对时间+帧号）
    - 各节点提前缓存2-3秒视频帧，按playAt时间点统一播放

* 画面裁剪：
  - 节点根据layout.rows/cols和自身position计算裁剪区域
  - 裁剪后缩放至HDMI输出分辨率


  
局域网方案活动图
```mermaid
flowchart TD
  subgraph 前端
    Web[Web前端<br/>拖拽配置连接关系]
  end

  subgraph 主机
    H_Cross[主机 Cross]
    H_IJetty[主机 IJetty]
  end

  subgraph 节点1
    N1_Cross[节点1 Cross]
    N1_IJetty[节点1 IJetty]
  end

  subgraph 节点2
    N2_Cross[节点2 Cross]
    N2_IJetty[节点2 IJetty]
  end

  subgraph 节点3
    N3_Cross[节点3 Cross]
    N3_IJetty[节点3 IJetty]
  end

  subgraph 信号源
    VS[RTSP视频源]
  end

  subgraph 组播
    M[组播域]
  end

  Web -->|配置连接关系| H_Cross

  H_Cross -->|TCP单播建立集群<br/>携带position| N1_Cross
  H_Cross -->|TCP单播建立集群<br/>携带position| N2_Cross
  H_Cross -->|TCP单播建立集群<br/>携带position| N3_Cross

  H_Cross -->|组播心跳<br/>含position| M
  N1_Cross -->|组播心跳| M
  N2_Cross -->|组播心跳| M
  N3_Cross -->|组播心跳| M

  Web -->|请求| H_Cross

  H_Cross -->|POST 区域1参数| H_IJetty
  H_Cross -->|POST 区域2参数| N1_IJetty
  H_Cross -->|POST 区域3参数| N2_IJetty
  H_Cross -->|POST 区域4参数| N3_IJetty

  H_Cross -->|GET 区域1| H_IJetty
  H_Cross -->|GET 区域2| N1_IJetty
  H_Cross -->|GET 区域3| N2_IJetty
  H_Cross -->|GET 区域4| N3_IJetty

  H_IJetty -->|WS| H_Cross
  N1_IJetty -->|WS| H_Cross
  N2_IJetty -->|WS| H_Cross
  N3_IJetty -->|WS| H_Cross

  H_Cross -->|转发全部WS| Web

  VS -->|拉RTSP流| H_IJetty
  VS -->|拉RTSP流| N1_IJetty
  VS -->|拉RTSP流| N2_IJetty
  VS -->|拉RTSP流| N3_IJetty

  H_Cross -->|广播帧号/时间戳| N1_Cross
  H_Cross -->|广播帧号/时间戳| N2_Cross
  H_Cross -->|广播帧号/时间戳| N3_Cross

  H_IJetty -->|seek+裁剪| P1[画面 区域1]
  N1_IJetty -->|seek+裁剪| P2[画面 区域2]
  N2_IJetty -->|seek+裁剪| P3[画面 区域3]
  N3_IJetty -->|seek+裁剪| P4[画面 区域4]

  P1 --> S[大屏拼接]
  P2 --> S
  P3 --> S
  P4 --> S

  style H_Cross fill:#c8e6f5,stroke:#0066cc
  style H_IJetty fill:#c8e6f5,stroke:#0066cc
  style N1_Cross fill:#fff3e0,stroke:#ef6c00
  style N2_Cross fill:#fff3e0,stroke:#ef6c00
  style N3_Cross fill:#fff3e0,stroke:#ef6c00
  style N1_IJetty fill:#fff3e0,stroke:#ef6c00
  style N2_IJetty fill:#fff3e0,stroke:#ef6c00
  style N3_IJetty fill:#fff3e0,stroke:#ef6c00
  style VS fill:#ffcdd2,stroke:#c62828
  style P1 fill:#fff9c4,stroke:#f9a825
  style P2 fill:#fff9c4,stroke:#f9a825
  style P3 fill:#fff9c4,stroke:#f9a825
  style P4 fill:#fff9c4,stroke:#f9a825
  style S fill:#c8e6c9,stroke:#2e7d32
```



云上方案活动图
```mermaid
flowchart TD
  subgraph 前端
    Web[Web前端<br/>拖拽配置连接关系]
  end

  subgraph 主机侧
    H_Cross[主机 Cross]
    H_IJetty[主机 IJetty]
  end

  subgraph 云服务器
    Cloud[云协同服务<br/>WebSocket长连接<br/>设备注册/指令转发/帧号广播]
  end

  subgraph 楼1
    N1_Cross[节点1 Cross]
    N1_IJetty[节点1 IJetty]
  end

  subgraph 楼2
    N2_Cross[节点2 Cross]
    N2_IJetty[节点2 IJetty]
  end

  subgraph 楼3
    N3_Cross[节点3 Cross]
    N3_IJetty[节点3 IJetty]
  end

  subgraph 信号源
    VS[RTSP视频源<br/>Camera采集/文件推流]
  end

  subgraph NTP
    NTP_Server[NTP校时服务器]
  end

  Web -->|配置连接关系| H_Cross

  H_Cross -->|WebSocket注册+建集群<br/>携带position| Cloud
  N1_Cross -->|WebSocket注册+建集群<br/>携带position| Cloud
  N2_Cross -->|WebSocket注册+建集群<br/>携带position| Cloud
  N3_Cross -->|WebSocket注册+建集群<br/>携带position| Cloud

  N1_Cross -->|NTP校时| NTP_Server
  N2_Cross -->|NTP校时| NTP_Server
  N3_Cross -->|NTP校时| NTP_Server

  Web -->|请求| H_Cross

  H_Cross -->|POST/GET指令| Cloud
  Cloud -->|转发指令| H_Cross
  Cloud -->|转发指令| N1_Cross
  Cloud -->|转发指令| N2_Cross
  Cloud -->|转发指令| N3_Cross

  H_Cross -->|执行| H_IJetty
  N1_Cross -->|执行| N1_IJetty
  N2_Cross -->|执行| N2_IJetty
  N3_Cross -->|执行| N3_IJetty

  H_IJetty -->|WS状态| H_Cross
  N1_IJetty -->|WS状态| N1_Cross
  N2_IJetty -->|WS状态| N2_Cross
  N3_IJetty -->|WS状态| N3_Cross

  N1_Cross -->|状态上报| Cloud
  N2_Cross -->|状态上报| Cloud
  N3_Cross -->|状态上报| Cloud
  Cloud -->|状态推送| H_Cross
  H_Cross -->|推送| Web

  VS -->|拉RTSP流<br/>预缓存3秒| H_IJetty
  VS -->|拉RTSP流<br/>预缓存3秒| N1_IJetty
  VS -->|拉RTSP流<br/>预缓存3秒| N2_IJetty
  VS -->|拉RTSP流<br/>预缓存3秒| N3_IJetty

  H_Cross -->|playAt指令<br/>绝对时间+帧号| Cloud
  Cloud -->|广播playAt| N1_Cross
  Cloud -->|广播playAt| N2_Cross
  Cloud -->|广播playAt| N3_Cross

  H_IJetty -->|seek+裁剪| P1[画面 区域1]
  N1_IJetty -->|seek+裁剪| P2[画面 区域2]
  N2_IJetty -->|seek+裁剪| P3[画面 区域3]
  N3_IJetty -->|seek+裁剪| P4[画面 区域4]

  P1 --> S[湘江对岸<br/>完整画面]
  P2 --> S
  P3 --> S
  P4 --> S

  style H_Cross fill:#c8e6f5,stroke:#0066cc
  style H_IJetty fill:#c8e6f5,stroke:#0066cc
  style Cloud fill:#d4f1d4,stroke:#2e7d32
  style N1_Cross fill:#fff3e0,stroke:#ef6c00
  style N2_Cross fill:#fff3e0,stroke:#ef6c00
  style N3_Cross fill:#fff3e0,stroke:#ef6c00
  style N1_IJetty fill:#fff3e0,stroke:#ef6c00
  style N2_IJetty fill:#fff3e0,stroke:#ef6c00
  style N3_IJetty fill:#fff3e0,stroke:#ef6c00
  style VS fill:#ffcdd2,stroke:#c62828
  style NTP_Server fill:#e8eaf6,stroke:#3949ab
  style P1 fill:#fff9c4,stroke:#f9a825
  style P2 fill:#fff9c4,stroke:#f9a825
  style P3 fill:#fff9c4,stroke:#f9a825
  style P4 fill:#fff9c4,stroke:#f9a825
  style S fill:#c8e6c9,stroke:#2e7d32
```
