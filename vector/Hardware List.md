# 硬件清单



Vector需要的硬件清单


* 主板: 香橙派Orange Pi 5 Plus（RK3588）
* 屏幕: 7寸MIPI触控屏
* 电源: Type-C 5V4A电源
* 舵机: SG90舵机
* 喇叭: 无源小喇叭2P杜邦线
* 杜邦线
* 摄像头: USB摄像头
* 内置: WiFi6.0，BLE蓝牙5.0，麦克风




## Android Framework AOSP定制

RK3588 机器狗 AOSP 定制任务清单

### 核心目标
开机后直接进入表情App，用户无法操作任何系统界面，机器狗只能通过SpringBoot指令控制。


### 必须修改的AOSP源码（3个核心任务）

#### 任务1：移除桌面应用，锁定单一App

- 文件：device/rockchip/rk3588/device.mk
- 操作：注释或删除 Launcher3、Launcher2QuickStep 等桌面应用
- 目的：系统没有桌面，只能运行你的 emoji.apk

#### 任务2：全局拦截返回键和Home键

- 文件：frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
- 方法：interceptKeyBeforeDispatching()
- 操作：拦截 KEYCODE_BACK 和 KEYCODE_HOME，返回 -1 使其全局失效
- 目的：用户按任何键都无法退出表情App

#### 任务3：彻底禁用下拉状态栏

- 目录：frameworks/base/packages/SystemUI/
- 核心文件：PhoneStatusBarView.java 或类似触摸处理文件
- 操作：修改顶部下拉手势处理逻辑，使其完全失效
- 目的：任何情况下都不会弹出状态栏和通知栏

### 可选修改（体验优化，非必须修改AOSP源码）

#### 任务4：自定义开机动画
- 方式：替换 /system/media/bootanimation.zip
- 无源码修改，纯资源替换

#### 任务5：默认启动App（开机自启）
- 在 emoji.apk 中注册 BOOT_COMPLETED 广播
- 收到广播后启动主界面
- 纯应用层代码，无源码修改

#### 任务6：移除系统App
- 文件：device/rockchip/rk3588/device.mk
- 操作：注释或删除 Settings、FileManager 等
- 目的：彻底消除用户接触Android系统的隐患


### 不需要修改的部分

- 舵机控制：JNI + HAL 层，不动AOSP
- 摄像头：标准 Camera2 API，不动AOSP
- 表情显示：emoji.apk 独立应用，不动AOSP
- 指令接收：SpringBoot 通信，纯应用层，不动AOSP


### 学习路径建议

1. 先改 device.mk（编译配置，最简单）
2. 再改 PhoneWindowManager.java（系统服务，核心技能）
3. 最后改 SystemUI（系统界面，深入理解Framework）




