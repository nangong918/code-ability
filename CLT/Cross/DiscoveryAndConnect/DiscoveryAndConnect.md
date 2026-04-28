# Cross 组播发现


## 设计方案


### Discovery

采用UDP组播进行局域网发现
在启用**多机协同**功能之后，本设备的Cross开始发送组播。
局域网内全部的设备开启**多机协同**功能之后，设备会发送和接收组播。
3秒心跳发送，5秒超时删除。
Cross会收集信息主动上推展示在对应的web上。

组播对等发现实例图
```mermaid
graph TD
    A[设备A] -->|组播| B[局域网]
    C[设备B] -->|组播| B
    D[设备C] -->|组播| B
    
    B -->|接收| A
    B -->|接收| C
    B -->|接收| D
```

组播活动图
```mermaid
flowchart TD
    subgraph 设备A
        A_start([CrossService启动]) --> A_enable{多机协同<br/>功能开启?}
        A_enable -->|否| A_wait[待命，不参与发现]
        A_enable -->|是| A_init[启动组播发送器<br/>创建MulticastSocket<br/>绑定网卡eth0]
        A_init --> A_loop[定时线程池<br/>每3秒执行]
        
        A_loop --> A_collect[采集本机信息]
        A_collect --> A_build[构建MulticastInfoDTO<br/>basicInfo + configInfo + deviceGroup]
        A_build --> A_json[JSON序列化]
        A_json --> A_send[组播发送<br/>目标: 225.5.5.5:8992]
        A_send --> A_loop
    end

    subgraph 设备B
        B_start([CrossService启动]) --> B_enable{多机协同<br/>功能开启?}
        B_enable -->|否| B_wait[待命，不参与发现]
        B_enable -->|是| B_init[启动组播接收器<br/>创建MulticastSocket端口8992<br/>加入组播组225.5.5.5 @ eth0]
        B_init --> B_recvLoop[定时3秒检查消息队列线程池]
        B_init --> B_timeout[定时线程池<br/>每5秒检查超时]
        
        B_recvLoop --> B_recv[receive接收数据包]
        B_recv --> B_parse[解析JSON → MulticastInfoDTO]
        B_parse --> B_valid{数据合法?}
        B_valid -->|否| B_recvLoop
        B_valid -->|是| B_update[更新心跳时间<br/>deviceHeartbeat]
        B_update --> B_both{本机与对端<br/>都Enable?}
        B_both -->|否| B_recvLoop
        B_both -->|是| B_notify[WebSocket通知前端]
        B_notify --> B_recvLoop
        
        B_timeout --> B_check{超过5秒<br/>无心跳?}
        B_check -->|是| B_remove[移除离线设备<br/>WebSocket通知前端]
        B_check -->|否| B_timeout
        B_remove --> B_timeout
    end

    A_send -.->|UDP组播 225.5.5.5:8992| B_recv
```

### Connect

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

```mermaid
flowchart TD
    subgraph 请求端
        S_start([用户触发建立协同]) --> S_select{选择目标设备}
        S_select --> S_type{选择协同类型}
        S_type -->|双机备份| S_check[检查硬件板卡<br/>软件版本一致性]
        S_type -->|多机协同| S_send
        S_type -->|分布式集群| S_send
        S_check --> S_match{配置一致?}
        S_match -->|否| S_error[弹窗提示<br/>硬件/版本不一致]
        S_match -->|是| S_send[TCP Socket单播<br/>发送建立请求]
        S_send --> S_wait[等待响应]
    end

    subgraph 目标端
        T_listen[TCP Socket监听] --> T_recv[接收建立请求]
        T_recv --> T_accept{接受请求?}
        T_accept -->|否| T_reject[回复拒绝]
        T_accept -->|是| T_agree[回复同意]
    end

    T_reject --> S_wait
    T_agree --> S_confirm[收到同意响应]
    S_confirm --> S_update[更新本地组播配置<br/>deviceGroup信息变更]
    S_update --> S_notify[WebSocket通知前端<br/>建立成功]

    T_agree --> T_update[更新本地组播配置<br/>deviceGroup信息变更]
    T_update --> T_notify[WebSocket通知前端<br/>建立成功]

    subgraph 心跳维护
        S_confirm --> S_heartbeat[启动心跳机制<br/>定时线程池]
        T_agree --> T_heartbeat[启动心跳机制<br/>定时线程池]
        
        S_heartbeat --> S_hbLoop[每3秒发送心跳]
        T_heartbeat --> T_hbLoop[每3秒发送心跳]
        
        S_hbLoop --> S_hbCheck{3秒内收到<br/>对方心跳?}
        S_hbCheck -->|否| S_delay[标注设备延迟<br/>提示用户]
        S_hbCheck -->|是| S_hbLoop
        
        S_delay --> S_hb10{10秒内收到<br/>对方心跳?}
        S_hb10 -->|否| S_offline[提示用户设备离线]
        S_hb10 -->|是| S_hbLoop
        
        T_hbLoop --> T_hbCheck{3秒内收到<br/>对方心跳?}
        T_hbCheck -->|否| T_delay[标注设备延迟<br/>提示用户]
        T_hbCheck -->|是| T_hbLoop
        
        T_delay --> T_hb10{10秒内收到<br/>对方心跳?}
        T_hb10 -->|否| T_offline[提示用户设备离线]
        T_hb10 -->|是| T_hbLoop

        S_offline --> S_retry[重试连接<br/>最多3次]
        S_retry --> S_retryOk{重连成功?}
        S_retryOk -->|是| S_hbLoop
        S_retryOk -->|否| S_abnormal[提示用户设备异常]

        T_offline --> T_retry[重试连接<br/>最多3次]
        T_retry --> T_retryOk{重连成功?}
        T_retryOk -->|是| T_hbLoop
        T_retryOk -->|否| T_abnormal[提示用户设备异常]
    end

    subgraph 读写超时
        S_rw[TCP读写操作] --> S_rwCheck{读写超时?}
        S_rwCheck -->|是| S_rwDelay[提示用户设备延迟高]
        S_rwCheck -->|否| S_rw
    end
```


双机备份
```mermaid
flowchart TD
    subgraph 主机环境
        A[主机 Cross] --> A1[Discovery 组播心跳]
        A --> A2[操控主机IJetty]
    end

    subgraph 备机环境
        B[备机 Cross] --> B1[Discovery 组播心跳]
        B --> B2[操控备机IJetty]
    end

    subgraph 网络组播
        M[组播域 225.5.5.5:8992]
    end

    A1 --> M
    B1 --> M
    M --> A1
    M --> B1

    A -->|建立主备关系| R{主机 Cross 决策}
    R -->|操控指令| A2
    R -->|操控指令| B2

    style A fill:#c8e6f5,stroke:#0066cc
    style B fill:#ffe0b5,stroke:#cc6600
    style R fill:#d4f1d4,stroke:#2e7d32
```

协同组

```mermaid
flowchart TD
    subgraph 主机环境
        A[主机 Cross] --> A1[Discovery 组播心跳]
        A --> A2[操控主机IJetty]
    end

    subgraph 协同机1环境
        B[协同机1 Cross] --> B1[Discovery 组播心跳]
        B --> B2[操控协同机1 IJetty]
    end

    subgraph 协同机2环境
        C[协同机2 Cross] --> C1[Discovery 组播心跳]
        C --> C2[操控协同机2 IJetty]
    end

    subgraph 网络组播
        M[组播域 225.5.5.5:8992]
    end

    A1 --> M
    B1 --> M
    C1 --> M
    M --> A1
    M --> B1
    M --> C1

    A -->|建立协同组| R{主机 Cross 决策}
    R -->|操控指令| A2
    R -->|操控指令| B2
    R -->|操控指令| C2

    style A fill:#c8e6f5,stroke:#0066cc
    style B fill:#e8d5f5,stroke:#7b1fa2
    style C fill:#e8d5f5,stroke:#7b1fa2
    style R fill:#d4f1d4,stroke:#2e7d32
```

分布式集群

```mermaid
flowchart TD
    subgraph 主机环境
        A[主机 Cross] --> A1[Discovery 组播心跳]
        A --> A2[操控主机IJetty<br/>画面区域1]
    end

    subgraph 节点1环境
        B[节点1 Cross] --> B1[Discovery 组播心跳]
        B --> B2[操控节点1 IJetty<br/>画面区域2]
    end

    subgraph 节点2环境
        C[节点2 Cross] --> C1[Discovery 组播心跳]
        C --> C2[操控节点2 IJetty<br/>画面区域3]
    end

    subgraph 节点3环境
        D[节点3 Cross] --> D1[Discovery 组播心跳]
        D --> D2[操控节点3 IJetty<br/>画面区域4]
    end

    subgraph 网络组播
        M[组播域 225.5.5.5:8992]
    end

    A1 --> M
    B1 --> M
    C1 --> M
    D1 --> M
    M --> A1
    M --> B1
    M --> C1
    M --> D1

    A -->|建立分布式集群| R{主机 Cross 决策}
    R -->|指令_区域1| A2
    R -->|指令_区域2| B2
    R -->|指令_区域3| C2
    R -->|指令_区域4| D2

    A2 --> S[大屏拼接]
    B2 --> S
    C2 --> S
    D2 --> S

    style A fill:#c8e6f5,stroke:#0066cc
    style B fill:#fff3e0,stroke:#ef6c00
    style C fill:#fff3e0,stroke:#ef6c00
    style D fill:#fff3e0,stroke:#ef6c00
    style R fill:#d4f1d4,stroke:#2e7d32
    style S fill:#fce4ec,stroke:#c62828
```


## 技术依托

组播相关计算机网络知识参考：[计算机网络.md](../../../knowledge/计算机网络.md)

组播技术依托:
* MulticastSocket: Java提供的专门用于组播通信的Socket类
    - 创建组播Socket实例
    - 加入组播组
    - 发送组播数据包
    - 接收组播数据包
```java
// 创建组播Socket
MulticastSocket ms = new MulticastSocket();

// 加入组播组
ms.joinGroup(groupAddress, networkInterface);

// 发送组播数据包
ms.send(dp);

// 接收组播数据包
ms.receive(dp);

// 离开组播组
ms.leaveGroup(group);
```

* InetSocketAddress: IP地址和端口的组合
    - 表示组播地址和端口
    - 用于指定组播组的地址信息

```java
// 创建组播地址
InetAddress group = InetAddress.getByName(CommonDef.MULTICAST_ADDRESS);
InetSocketAddress groupAddress = new InetSocketAddress(group, CommonDef.NSD_SERVER_PORT);
```
* DatagramPacket: UDP数据包的封装类
    - 封装组播数据
    - 包含数据内容、地址和端口信息
    - 用于Socket发送和接收

```java
// 封装组播数据包
byte[] data = json.getBytes(StandardCharsets.UTF_8);
DatagramPacket dp = new DatagramPacket(data, data.length, group, port);

// 接收组播数据包
DatagramPacket dp = new DatagramPacket(buffer, buffer.length);
ms.receive(dp);
```

* NetworkInterface: 网络接口的抽象表示
    - 表示物理或虚拟网卡
    - 支持多网卡环境下的组播
    - 选择特定网卡进行组播通信
```java
// 获取指定网卡
NetworkInterface networkInterface = NetworkUtils.getEth0NetworkInterface();
ms.setNetworkInterface(networkInterface);
```

## 核心代码



线程池相关知识参考：[操作系统.md](../../../knowledge/操作系统.md)

```java
/// 线程池
// 核心线程数
private static final int CORE_POOL_SIZE = 2;
// 最大线程数
private static final int MAX_POOL_SIZE = 5;
// 线程空闲时间
private static final long KEEP_ALIVE_TIME = 60L;
// 时间单位
private static final TimeUnit TIME_UNIT = TimeUnit.SECONDS;
// 任务队列
private static final BlockingQueue<Runnable> WORK_QUEUE = new LinkedBlockingQueue<>(100);
// 线程工厂
private static final ThreadFactory THREAD_FACTORY = new ThreadFactory() {
    private final AtomicInteger counter = new AtomicInteger(1);
    @Override
    public Thread newThread(Runnable r) {
        return new Thread(r, "multicast-pool-" + counter.getAndIncrement());
    }
};

// 拒绝策略
private static final RejectedExecutionHandler REJECTED_HANDLER = new ThreadPoolExecutor.AbortPolicy();
// 线程池实例
private static final ExecutorService EXECUTOR_SERVICE = new ThreadPoolExecutor(
        CORE_POOL_SIZE,
        MAX_POOL_SIZE,
        KEEP_ALIVE_TIME,
        TIME_UNIT,
        WORK_QUEUE,
        THREAD_FACTORY,
        REJECTED_HANDLER
);


/// 组播服务接口
public interface DeviceDiscoveryService {
    // 开始服务，只在开机时调用一次
    void start();
    // 结束服务，只调用一次
    void stop();
}

/// 组播服务实现
class DeviceDiscoveryServiceMulticastImpl implements DeviceDiscoveryService {
    private final MulticastSenderHelper sender = new MulticastSenderHelper();
    private final MulticastReceiverHelper receiver = new MulticastReceiverHelper();
    @Override
    public void start() {
        EXECUTOR_SERVICE.execute(() -> {
            sender.create();
            receiver.create();
        });
    }

    @Override
    public void stop() {
        sender.close();
        receiver.close();
    }
}


/// 组播发送
public class MulticastSenderHelper {
    private MulticastSocket ms;
    private InetAddress group;
    private volatile boolean isActive = true;

    // 使用线程池管理定时任务 (避免Sleep让线程浪费大量CPU资源) 
    private static final ScheduledExecutorService SCHEDULER = Executors.newScheduledThreadPool(2);
    private ScheduledFuture<?> future;

    public void create() {
        if (null != ms) return;
        try {
            ms = new MulticastSocket();
            // 选择网卡 
            NetworkInterface networkInterface = NetworkUtils.getEth0NetworkInterface();
            ms.setNetworkInterface(networkInterface);
            group = InetAddress.getByName("225.5.5.5");

            // 3s 一次心跳 
            future = SCHEDULER.scheduleAtFixedRate(
                    () -> {
                        if (isActive) {
                            sendHeartbeat();
                        }
                    },
                    0, 3, TimeUnit.SECONDS
            );
        } catch (IOException e) {
            Log.e(TAG, "create: failed create multicast", e);
            ms = null;
            group = null;
        }
    }

    public void close() {
        if (null == ms || null == group) return;

        // 取消定时任务 
        if (future != null) {
            future.cancel(true);
        }

        ms.close();
    }

    private void sendHeartbeat() {
        // 心跳用来将当前设备的信息广播到网络中，不用处理其他无用信息 
        try {
            // 获取设备类型和自定义名称 
            DeviceBaseInfoDTO baseInfoDTO = IJettyClientService.getService().getBaseInfo("127.0.0.1", new HashMap<>());
            // 获取设备配置信息 
            ConfigDTO configDTO = ConfigService.getService().getConfigDTO();
            // 获取当前协同组信息 
            List<DeviceGroupDTO> deviceGroupDTOS = DeviceManagerService.getService().getGroupList();

            // 发送 
            MulticastInfoDTO multicastInfoDTO = new MulticastInfoDTO();
            multicastInfoDTO.setDeviceInfo(baseInfoDTO);
            multicastInfoDTO.setConfigInfo(configDTO);
            multicastInfoDTO.setDeviceGroupInfo(deviceGroupDTOS);

            String multicastInfoJson = new Gson().toJson(multicastInfoDTO);
            byte[] multicastInfoJsonBytes = multicastInfoJson.getBytes(StandardCharsets.UTF_8);

            DatagramPacket dp = new DatagramPacket(multicastInfoJsonBytes, multicastInfoJsonBytes.length, group, CommonDef.NSD_SERVER_PORT);
            ms.send(dp);
        } catch (Exception e) {
            Log.e(TAG, "sendHeartbeat: failed send heartbeat", e);
        }
    }
}

/// 组播接收 
public class MulticastReceiverHelper {
    private MulticastSocket ms;
    private InetAddress group;
    private final byte[] buffer = new byte[8192];

    // 使用线程池管理任务 
    private static final ExecutorService EXECUTOR_SERVICE = Executors.newFixedThreadPool(2);
    private Future<?> receiverFuture;
    private ScheduledFuture<?> checkHeartbeatFuture;

    // 超时检测时间 
    private final long timeoutMs = 5 * 1000;  // 超时检测5s 

    public void create() {
        if (null != ms) return;
        try {
            ms = new MulticastSocket(8992);

            group = InetAddress.getByName("225.5.5.5");
            InetSocketAddress groupAddress = new InetSocketAddress(group, 8992);

            // 指定组播的网卡 
            NetworkInterface networkInterface = NetworkUtils.getEth0NetworkInterface();
            ms.joinGroup(groupAddress, networkInterface);

            // 启动接收任务 
            receiverFuture = EXECUTOR_SERVICE.submit(this::receive);

            // 启动心跳检测任务 
            checkHeartbeatFuture = EXECUTOR_SERVICE.scheduleAtFixedRate(
                    this::checkTimeouts, 5, 5, TimeUnit.SECONDS
            );
        } catch (IOException e) {
            Log.e(TAG, "create: failed create multicast", e);
            ms = null;
            group = null;
        }
    }

    public void close() {
        if (null == ms || null == group) return;

        try {
            // 取消接收任务 
            if (receiverFuture != null) {
                receiverFuture.cancel(true);
            }

            // 取消心跳检测任务 
            if (checkHeartbeatFuture != null) {
                checkHeartbeatFuture.cancel(true);
            }

            MulticastSocket.leaveGroup(group);
            ms.close();
        } catch (IOException e) {
            Log.e(TAG, "close: failed close multicast", e);
        }
    }

    private void receive() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                DatagramPacket dp = new DatagramPacket(buffer, buffer.length);
                ms.receive(dp);
                String remoteIp = dp.getAddress().getHostAddress();

                // 解析名称 
                String data = new String(dp.getData(), 0, dp.getLength());
                MulticastInfoDTO dto = new Gson().fromJson(data, MulticastInfoDTO.class);
                if (null == dto ||
                        null == dto.getDeviceInfo() ||
                        null == dto.getConfigInfo()) {
                    continue;
                }

                // 处理设备信息 
                handleDeviceInfo(remoteIp, dto.getDeviceInfo(), dto.getConfigInfo());

                // 处理协同组信息 
                boolean localEnable = ConfigService.getService().getConfigDTO().isEnable();
                boolean remoteEnable = dto.getConfigInfo().isEnable();
                // 当本机或者对方协同开关关闭时，不处理协同组信息 
                if (localEnable && remoteEnable) {
                    String localIp = NetworkUtils.getEth0IpAddress();
                    // 不处理本机的协同组信息，提高性能 
                    if (!remoteIp.equals(localIp)) {
                        handleGroupInfo(localIp, remoteIp, dto.getDeviceGroupInfo());
                    }
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } catch (Exception e) {
                Log.e(TAG, "receive: failed receive multicast", e);
            }
        }
    }

    private void checkTimeouts() {
        long now = System.currentTimeMillis();
        DeviceListDTO deviceListDTO = DeviceManagerService.getService().getDeviceAndGroupList();

        // 检查设备列表的在线情况 
        deviceListDTO.getDeviceArr().forEach(deviceDTO -> {
            if (Math.abs(now - deviceDTO.getLastTime()) > timeoutMs) {
                Log.e(TAG, "checkTimeouts: remove ip = " + deviceDTO.getIp());
                DeviceManagerService.getService().removeDevice(deviceDTO.getIp());
            }
        });

        // 检查协同组中设备的在线情况 
        deviceListDTO.getGroupArr().forEach(groupDTO -> {
            if (!groupDTO.isDelete()) {
                groupDTO.getDeviceArr().forEach(deviceDTO -> {
                    if (Math.abs(now - deviceDTO.getLastTime()) > timeoutMs) {
                        DeviceManagerService.getService().removeDevice(deviceDTO.getIp());
                    }
                });
            }
        });
    }

    private void handleDeviceInfo(String ip, @NonNull DeviceBaseInfoDTO baseInfoDTO, @NonNull ConfigDTO configDTO) {
        DeviceManagerService.getService().deviceHeartbeat(ip, baseInfoDTO, configDTO);
    }

    private void handleGroupInfo(String localIp, String remoteIp, List<DeviceGroupDTO> deviceGroupDTOS) {
        // 将其他设备的协同组信息转为map，方便后续处理
        HashMap<String, DeviceGroupDTO> otherGroup = new HashMap<>();
        for (DeviceGroupDTO dto : deviceGroupDTOS) {
            otherGroup.put(dto.getId(), dto);
        }

        // 获取当前设备的协同组信息
        List<DeviceGroupDTO> currentDeviceGroupDTOS = DeviceManagerService.getService().getGroupList();
        HashMap<String, DeviceGroupDTO> currentGroup = new HashMap<>();
        for (DeviceGroupDTO dto : currentDeviceGroupDTOS) {
            currentGroup.put(dto.getId(), dto);
        }

        // 1.合并协同组，当两个协同组id相同时，以最新的为准 
        currentGroup.forEach((key, value) ->
                otherGroup.merge(key, value, (v1, v2) -> ((v1.compareTo(v2) > 0) ? v1 : v2))
        );

        // 2.根据设备查找存在冲突的协同组 
        HashMap<String, List<DeviceGroupDTO>> deviceInGroupMap = new HashMap<>();
        otherGroup.forEach((groupId, groupDTO) -> groupDTO.getDeviceArr().forEach((deviceDTO) -> {
            if (deviceInGroupMap.containsKey(deviceDTO.getIp())) {
                deviceInGroupMap.get(deviceDTO.getIp()).add(groupDTO);
            } else {
                List<DeviceGroupDTO> tmp = new ArrayList<>();
                tmp.add(groupDTO);
                deviceInGroupMap.put(deviceDTO.getIp(), tmp);
            }
        }));

        // 根据冲突，删除旧的协同组 
        DeviceGroupAbnormalDTO groupAbnormalDTO = new DeviceGroupAbnormalDTO();
        deviceInGroupMap.forEach((key, value) -> {
            if (value.size() <= 1) {
                return;
            }

            // 按时间升序 
            Collections.sort(value);

            // 删除旧的协同组，第一个为最新的 
            for (int i = 0; i < value.size() - 1; i++) {
                DeviceGroupDTO groupDTO = value.get(i);
                otherGroup.remove(groupDTO.getId());

                // 加入到异常信息，该协同组不是一个已经删除的协同组，并且位于当前设备的协同组列表 
                if (!groupDTO.isDelete() && currentGroup.containsKey(groupDTO.getId())) {
                    DeviceGroupAbnormalDTO.GroupAbnormalInfo abnormalInfo = new DeviceGroupAbnormalDTO.GroupAbnormalInfo();
                    abnormalInfo.setId(groupDTO.getId());
                    abnormalInfo.setName(groupDTO.getName());
                    abnormalInfo.setConflictIp(key);
                    groupAbnormalDTO.getGroupArr().add(abnormalInfo);
                }
            }
        });

        // 与现有协同组合并 
        DeviceManagerService.getService().mergeGroup(localIp, otherGroup);

        // 通知异常协同组被删除 
        if (groupAbnormalDTO.getGroupArr().size() > 0) {
            WebSocketManagerService.getService().broadcastToClient(
                    WSResponse.create(WSCommandType.Device_UpdateGroupAbnormal, groupAbnormalDTO).toString()
            );
        }
    }
}

/// 网口工具
class NetworkUtils {
    public static NetworkInterface getEth0NetworkInterface() {
        try {
            // 获取所有网络接口
            Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
            while (interfaces.hasMoreElements()) {
                NetworkInterface networkInterface = interfaces.nextElement();
                // 检查接口名称
                String interfaceName = networkInterface.getName();
                if (interfaceName.equalsIgnoreCase(CommonDef.NET_INTERFACE_NAME)) {
                    return networkInterface;
                }
            }
        } catch (SocketException ex) {
            Log.e("NetworkUtils", "获取网络接口信息失败: ", ex);
        }
        return null;
    }
}
```









