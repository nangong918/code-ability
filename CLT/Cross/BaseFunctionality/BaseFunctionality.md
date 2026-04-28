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
    style P1 fill:#fff9c4,stroke:#f9a825
    style P2 fill:#fff9c4,stroke:#f9a825
    style P3 fill:#fff9c4,stroke:#f9a825
    style S fill:#c8e6c9,stroke:#2e7d32
```

## 分布式集群

主机和分布式集群配置无需一致。

* 画面源：多屏组，数据源为分布式集群所有机器。
* 画面职责：各设备负责大屏不同区域，需配置连接关系（主机Web通过拖拽的方式配置）。
* 建立方式：主机通过TCP Socket单播向各节点发起建立请求，
  请求中携带该节点在大屏的position信息。
* 位置配置：组播心跳deviceGroup中扩展position字段，
  描述每台设备在大屏的行列位置。
* 指令分发：主机根据各节点position下发不同区域画面参数。

* 节点健康检测：
    - 心跳超时判定节点不健康（复用现有3秒/10秒/3次重试逻辑）

* 节点顶替：
    - 节点离线后，主机选择集群内其他健康节点顶替
    - 顶替节点接管离线节点的画面区域，同时输出原区域 + 顶替区域


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
        N1_IJetty[节点1 IJetty]
    end

    subgraph 节点2
        N2_IJetty[节点2 IJetty]
    end

    subgraph 节点3
        N3_IJetty[节点3 IJetty]
    end

    subgraph 组播
        M[组播域]
    end

    Web -->|配置连接关系| H_Cross

    H_Cross -->|TCP单播建立集群<br/>携带position信息| N1_IJetty
    H_Cross -->|TCP单播建立集群<br/>携带position信息| N2_IJetty
    H_Cross -->|TCP单播建立集群<br/>携带position信息| N3_IJetty

    H_Cross -->|组播心跳<br/>含position| M
    N1_IJetty -.->|组播心跳| M
    N2_IJetty -.->|组播心跳| M
    N3_IJetty -.->|组播心跳| M

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

    H_IJetty --> P1[画面 区域1]
    N1_IJetty --> P2[画面 区域2]
    N2_IJetty --> P3[画面 区域3]
    N3_IJetty --> P4[画面 区域4]

    P1 --> S[大屏拼接]
    P2 --> S
    P3 --> S
    P4 --> S

    subgraph 节点顶替
        H_Cross -->|检测心跳超时| HC{节点健康?}
        HC -->|节点2离线| RP[主机选择健康节点<br/>顶替节点2区域]
        RP -->|节点1顶替| N1_IJetty
        N1_IJetty --> P2_2[输出区域1+区域2]
        HC -->|节点2恢复| RB[自动切回<br/>释放顶替]
        RB --> P2
    end

    style H_Cross fill:#c8e6f5,stroke:#0066cc
    style H_IJetty fill:#c8e6f5,stroke:#0066cc
    style N1_IJetty fill:#fff3e0,stroke:#ef6c00
    style N2_IJetty fill:#fff3e0,stroke:#ef6c00
    style N3_IJetty fill:#fff3e0,stroke:#ef6c00
    style P1 fill:#fff9c4,stroke:#f9a825
    style P2 fill:#fff9c4,stroke:#f9a825
    style P3 fill:#fff9c4,stroke:#f9a825
    style P4 fill:#fff9c4,stroke:#f9a825
    style S fill:#c8e6c9,stroke:#2e7d32
    style P2_2 fill:#ffcdd2,stroke:#c62828
```
