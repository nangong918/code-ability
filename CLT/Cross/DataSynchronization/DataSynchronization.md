# 数据同步


## 设计方案

数据同步检测：同时请求双机的IJetty的接口，IJetty请求RK提供的参数配置文件的当前Hash值。
Cross比对两者的Hash值，如果一样就提示设备数据一致。不一样则提示主备数据不一致，需要进行数据同步。
同步方法，利用RK提供的备份数据功能，主机Cross将本机的数据导出为备份文件，
导出成功之后通知备机进行参数同步并告知备机的Cross文件路径进行下载。
备机下载成功同步文件之后调用RK的导入备份文件，并上推当前状态。链路执行完成则提示数据同步完成。


活动图
```mermaid
flowchart TD
    subgraph 触发
        S0([用户触发数据同步检测]) --> S1
        S1([建立主备关系后自动触发]) --> A
    end

    subgraph 数据一致性检测
        A[Cross同时请求] --> B[请求主机IJetty<br/>获取参数配置文件Hash]
        A --> C[请求备机IJetty<br/>获取参数配置文件Hash]
        
        B --> D[返回主机Hash值]
        C --> E[返回备机Hash值]
        
        D --> F{Cross比对<br/>Hash值}
        E --> F
        
        F -->|一致| G[提示<br/>设备数据一致]
        F -->|不一致| H[提示<br/>主备数据不一致<br/>需要同步]
    end

    subgraph 数据同步流程
        H --> I[主机Cross调用RK<br/>导出本机数据为备份文件]
        I --> J{导出成功?}
        J -->|否| K[提示导出失败]
        J -->|是| L[通知备机进行参数同步<br/>告知备份文件路径]
        
        L --> M[备机Cross下载<br/>备份文件]
        M --> N{下载成功?}
        N -->|否| O[提示下载失败]
        N -->|是| P[备机调用RK<br/>导入备份文件]
        
        P --> Q{导入成功?}
        Q -->|否| R[提示导入失败]
        Q -->|是| S[备机上推当前状态<br/>WebSocket通知前端]
        
        S --> T[链路执行完成<br/>提示数据同步完成]
    end

    style G fill:#c8e6c9,stroke:#2e7d32
    style T fill:#c8e6c9,stroke:#2e7d32
    style K fill:#ffcdd2,stroke:#c62828
    style O fill:#ffcdd2,stroke:#c62828
    style R fill:#ffcdd2,stroke:#c62828
```

实例图
```mermaid
flowchart TD
    subgraph 主机环境
        A[主机 Cross] --> A1[主机 IJetty]
        A1 --> A2[RK备份导出]
        A2 --> A3[备份文件]
    end

    subgraph 备机环境
        B[备机 Cross] --> B1[备机 IJetty]
        B1 --> B2[RK备份导入]
    end

    subgraph 文件传输
        F[备份文件下载]
    end

    A -->|1.调用导出| A1
    A1 -->|2.RK导出| A2
    A2 -->|3.生成| A3
    A3 -->|4.通知备机路径| B
    B -->|5.下载文件| F
    F -->|6.下载完成| B
    B -->|7.调用导入| B1
    B1 -->|8.RK导入| B2
    B2 -->|9.导入完成| B
    B -->|10.上推状态| W[WebSocket<br/>通知前端]

    style A fill:#c8e6f5,stroke:#0066cc
    style B fill:#ffe0b5,stroke:#cc6600
    style A3 fill:#fff9c4,stroke:#f9a825
    style F fill:#e8eaf6,stroke:#3949ab
    style W fill:#d4f1d4,stroke:#2e7d32
```

