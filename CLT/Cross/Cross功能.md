# Cross功能


## 设计

双机备份：两台设备配置，数据完全一致，主机挂掉备机顶上。
多机协同：多态设备配置不一致，数据可以不一致，所有设备执行主机的操作，失败则抛异常。
分布式集群：多设备组成设备集群，每个这杯负责不同的画面，整体组成一个大画面集群。

### 发现与连接

[DiscoveryAndConnect.md](DiscoveryAndConnect/DiscoveryAndConnect.md)

#### 1. 局域网发现

采用UDP组播进行局域网发现
在启用**多机协同**功能之后，本设备的Cross开始发送组播。
局域网内全部的设备开启**多机协同**功能之后，设备会发送和接收组播。
3秒心跳发送，5秒超时删除。
Cross会收集信息主动上推展示在对应的web上。

#### 2. 建立双机备份/协同组/集群
一个设备的Cross向另一个设备的Cross发送原生Java的**TCP Socket**单播请求建立主备关系的请求。
建立主备关系成功之后两者的组播配置变化，其他设备看到两台设备是建立好协同关系的。
超时逻辑：
 - 心跳间隔：3
 - 心跳超时：10
 - 心跳重试次数：3
心跳只要3秒中没收到就会标注设备延迟，并提示用户。
然后如果10秒都收不到心跳才会提示用户设备离线。
然后充实3次还是无法重连就提示用户设备异常。
心跳发送和检测都是用的是定时线程池而不是线程sleep。
写入和读取超时也会提示用户设备延迟高。

### 数据同步

[DataSynchronization.md](DataSynchronization/DataSynchronization.md)

#### 3. 主动数据同步

数据同步检测：同时请求双机的IJetty的接口，IJetty请求RK提供的参数配置文件的当前Hash值。
Cross比对两者的Hash值，如果一样就提示设备数据一致。不一样则提示主备数据不一致，需要进行数据同步。
同步方法，利用RK提供的备份数据功能，主机Cross将本机的数据导出为备份文件，
导出成功之后通知备机进行参数同步并告知备机的Cross文件路径进行下载。
备机下载成功同步文件之后调用RK的导入备份文件，并上推当前状态。链路执行完成则提示数据同步完成。

#### 4. 数据同步机制

##### 双机备份
1. POST请求同时发送给主机和备机，需要同步等待主备机都回复相同的成功结果，否则报错提示主备不一致。
2. GET请求只从主机获取。
3. WS长连接：接收主备的WS，但是只向前端转发主机的WS

##### 多机协同
1. POST请求同时发送给协同组，无需检查。
2. GET请求从所有设备获取。
3. WS长连接：接收并转发所有设备的WS

##### 分布式集群
1. POST请求按照不同的参数分别发送给不同的设备，需要异步上推结果，在Web页面展示哪个设备完成了。
2. GET请求分别从不同的设备获取，异步收集并展示在前端Web。
3. 接收并转发所有设备的WS。

##### WS长连接介绍
进度等信息需要等待主机和备机同时完成，有些进度需要合并，有些进度需要分别展示。

### 基本模块

#### 5. 双机备份
主备配置必须完全一致；

* 画面源：单屏幕，数据源为主机

* 系统鲁棒性
  - 备机掉线：弹窗提示并关闭主备状态，重新登录。
  - 主机掉线：主机的Web（eg：192.168.1.10）页面与主机的Cross长连接通讯丢失，从缓存获取到缓存的双机备份组，拿到备机IP（eg：192.168.1.11）并跳转到备机Web页面。
            同时屏幕的信号源也切换为备机的信号源。
            如果组播检测到主机上线则切回主备模式，切回主机Web（192.168.1.10）

* 异常情况：
  - 主备机硬件板卡布局，数据不一致：弹窗提示


#### 6. 多机协同
主机和协同组的配置不必要一致。

* 画面源：多屏组，数据源为协同组所有机器
* 画面职责：协同组画面一致

* 多机协同方案：
  - 局域网：
  - 云上：

#### 7. 分布式集群
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


## 局域网设备发现

基于UDP组播的设备发现

组播（Multicast）是一种网络通信技术，一个发送者向多个接收者发送相同的数据包，而不需要为每个接收者单独发送。

### 核心心跳数据结构




```json
{
  "basicInfo": {
    "ip": "192.168.1.100",
    "mac": "00:11:22:33:44:55",
    "name": "Device-001",
    "version": "V1.3.2",
    "isMainDevice": true,
    "boardType": "CLT-1000",
    "mode": "distributed"
  },
  "configInfo": {
    "enable": true,
    "ip": "192.168.1.100",
    "port": 8080,
    "multicastAddress": "224.0.0.1",
    "multicastPort": 5000,
    "syncParamHash": "1234567890abcdef"
  },
  "deviceGroup": {
    "id": "group-123",
    "name": "Production Line A",
    "type": "distributed_cluster",
    "createTime": 1620000000000,
    "layout": {
      "rows": 2,
      "cols": 2
    },
    "deviceArr": [
      {
        "ip": "192.168.1.100",
        "mac": "00:11:22:33:44:55",
        "name": "Device-001",
        "role": "primary",
        "position": {
          "row": 0,
          "col": 0
        },
        "backupDevice": {
          "ip": "192.168.1.101",
          "mac": "00:11:22:33:44:56",
          "name": "Device-002"
        },
        "healthy": true
      },
      {
        "ip": "192.168.1.101",
        "mac": "00:11:22:33:44:56",
        "name": "Device-002",
        "role": "node",
        "position": {
          "row": 0,
          "col": 1
        },
        "backupDevice": {
          "ip": "192.168.1.102",
          "mac": "00:11:22:33:44:57",
          "name": "Device-003"
        },
        "healthy": true
      },
      {
        "ip": "192.168.1.102",
        "mac": "00:11:22:33:44:57",
        "name": "Device-003",
        "role": "node",
        "position": {
          "row": 1,
          "col": 0
        },
        "backupDevice": {
          "ip": "192.168.1.103",
          "mac": "00:11:22:33:44:58",
          "name": "Device-004"
        },
        "healthy": true
      },
      {
        "ip": "192.168.1.103",
        "mac": "00:11:22:33:44:58",
        "name": "Device-004",
        "role": "node",
        "position": {
          "row": 1,
          "col": 1
        },
        "backupDevice": {
          "ip": "192.168.1.100",
          "mac": "00:11:22:33:44:55",
          "name": "Device-001"
        },
        "healthy": true
      }
    ]
  }
}
```

Java数据类型定义：
```java
/**
 * 组播心跳数据包，局域网设备发现的核心载体。
 * 每3秒通过UDP组播发送一次，包含本机信息、服务配置及当前协同组状态。
 */
public class MulticastInfoDTO {
    /** 设备基础信息 */
    public BasicInfo basicInfo;
    /** 组播及服务配置 */
    public ConfigInfo configInfo;
    /** 协同组/集群信息，未加入任何组时为null */
    public DeviceGroup deviceGroup;
}

/**
 * 设备基础信息，描述本机硬件与身份。
 */
public class BasicInfo {
    public String ip;
    public String mac;
    public String name;
    public String version;
    /** 是否为主设备，用于Cross角色判定 */
    public boolean isMainDevice;
    public String boardType;
    /** 当前运行模式：primary_backup / collaborative / distributed_cluster */
    public String mode;
}

/**
 * 组播及服务配置，控制发现开关与通信参数。
 */
public class ConfigInfo {
    /** 多机协同功能是否开启，关闭则停止组播收发 */
    public boolean enable;
    public String ip;
    public int port;
    /** 组播地址，默认225.5.5.5 */
    public String multicastAddress;
    /** 组播端口，默认8992 */
    public int multicastPort;
    /** 参数配置文件Hash值，用于主备数据一致性比对 */
    public String syncParamHash;
}

/**
 * 协同组/集群信息，描述当前设备所在组的完整状态。
 * 组内所有设备通过组播交换此信息，维持一致视图。
 */
public class DeviceGroup {
    /** 组唯一标识，建立协同关系时生成 */
    public String id;
    public String name;
    /** 组类型：primary_backup / collaborative / distributed_cluster */
    public String type;
    /** 组创建时间，冲突裁决时时间戳大者优先 */
    public long createTime;
    /** 大屏布局，仅distributed_cluster类型有效，其他类型为null */
    public Layout layout;
    /** 组内设备列表，包含所有在线设备 */
    public DeviceInfo[] deviceArr;
}

/**
 * 大屏拼接布局，描述分布式集群的画面排列。
 * 仅集群类型为distributed_cluster时填充。
 */
public class Layout {
    /** 大屏行数 */
    public int rows;
    /** 大屏列数 */
    public int cols;
}

/**
 * 组内设备节点信息，描述单个设备在组内的角色与状态。
 */
public class DeviceInfo {
    public String ip;
    public String mac;
    public String name;
    /** 设备角色：primary-主机 / backup-备机 / node-集群节点 */
    public String role;
    /** 大屏位置，仅distributed_cluster类型有效，其他类型为null */
    public Position position;
    /** 顶替设备，节点离线时由该设备自动接管画面区域 */
    public BackupDevice backupDevice;
    /** 心跳状态：true在线，false离线（组播心跳超时5秒判定） */
    public boolean healthy;
}

/**
 * 设备在大屏中的行列位置。
 * 配合Layout中的rows/cols完成画面区域划分。
 */
public class Position {
    /** 行索引，从0开始 */
    public int row;
    /** 列索引，从0开始 */
    public int col;
}

/**
 * 顶替设备引用，仅记录ip/mac/name用于标识。
 * 节点离线时由主机Cross查找该设备并下发顶替指令。
 */
public class BackupDevice {
    public String ip;
    public String mac;
    public String name;
}
```

