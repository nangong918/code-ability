# Framework

一、核心架构分层
1. Linux Kernel 层：驱动、电源、内存、网络、Binder
2. HAL 硬件抽象层：隔离内核与Framework，硬件厂商对接
3. Native 层：C/C++库，如OpenGL、SQLite、WebKit、媒体库、ART虚拟机
4. Android Framework 层（Java框架层）：你重点要背的

二、Android Framework 层（Java 核心服务）
1. Activity Manager Service (AMS)：Activity、Service、进程管理
2. Package Manager Service (PMS)：安装、卸载、权限、应用信息
3. Window Manager Service (WMS)：窗口、界面、Surface、View管理
4. View 体系：View、ViewGroup、UI渲染
5. Content Provider：跨进程数据共享
6. Broadcast Receiver：跨进程消息通信
7. Service：后台服务
8. Intent / IntentFilter：组件跳转、消息路由
9. Resource Manager：资源管理（布局、图片、字符串）
10. Notification Manager：通知
11. Telephony / Connectivity Manager：电话、网络
12. Location Manager：定位
13. Audio / Video Manager：音视频
14. Alarm Manager：定时唤醒
15. Power Manager：电源、休眠、亮灭屏
16. Binder IPC：跨进程通信核心（Android最重要）