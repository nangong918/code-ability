# Framework


## 目录

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




## 常见问题

**1)如何将Apk设置为Android系统应用？**

* 在Manifest中标记为系统应用：
  `android.uid.system`
  让这个 App 共享系统进程的 UID
  拥有后，App 直接获得系统级权限，可以调用普通第三方 App 完全无法使用的系统 API
  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
            package="org.xxx.packagename>"
            android:sharedUserId="android.uid.system">
  </manifest>
  ```

*  系统签名 打包
   系统签名 = 手机厂商 / ROM 内置的签名文件: platform.keystore
   使用AOSP（安卓原生系统）自带的系统签名文件

* 必须放在系统分区
  路径：`/system/app/`，`/system/priv-app/`
  普通应用在：第三方 App 安装在 `/data/app`，无法成为系统应用

* 其他配置:
  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
            package="org.xxx.packagename>"
            android:sharedUserId="android.uid.system">
    <application
        android:persistent="true">
        <service android:name="com.color.webserver.app.HttpService" android:exported="true"/>
        <service android:name="com.color.webserver.app.WebSocketService" android:exported="true"/>
        <receiver android:name="com.color.webserver.app.BootCompleteReceiver">
            <intent-filter android:priority="1000">
                <action android:name="android.intent.action.BOOT_COMPLETED" />
            </intent-filter>
            <intent-filter>
                <action android:name="com.color.intent.action.RESTART_SERVICE" />
            </intent-filter>
        </receiver>
    </application>
  </manifest>
  ```

  - 常驻进程: `android:persistent="true"`
    - 系统进程级别常驻
    - 意外杀死会自动重启
    - 只有系统应用才能生效，普通 App 设置无效

  - 监听系统开机成功广播: 开机自启（最高优先级）
    - 监听 Android 系统开机启动完成的信号
    - 系统开机 → 发出 BOOT_COMPLETED 广播 → App 收到 → 自动启动服务
  ```xml
    <intent-filter android:priority="1000">
       <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
  ```
    - 优先级 1000 = 系统最高
    - 开机第一时间启动服务
    - 普通第三方 App 无法做到这种稳定自启

*  监听系统开机成功广播
```java
package com.color.webserver.app;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.os.Environment;
import android.util.Log;

import java.io.DataOutputStream;
import java.io.File;
import java.io.IOException;

import java.io.File;

/**
 * Android广播接收器，用于设备上电启动时自动启动http服务和ws服务，以及处理服务重启
 * @author Reed
 * @since 2025-12-24 14:23:10
 */
public class BootCompleteReceiver extends BroadcastReceiver {
    private final static String TAG = "BootCompleteReceiver";
    private static final boolean DBG = true;

    /**
     * 停止HTTP和WebSocket服务
     * @param context 上下文
     */
    public static void stopService(Context context) {
        Intent intent1 = new Intent(context, HttpService.class);
        context.stopService(intent1);

        Intent intent2 = new Intent(context, WebSocketService.class);
        context.stopService(intent2);
    }

    /**
     * 启动HTTP和WebSocket服务
     * @param context 上下文
     */
    public static void startService(Context context) {
        if (DBG) Log.d(TAG, "startService. HttpService");
        Intent intent1 = new Intent(context, HttpService.class);
        context.startService(intent1);

        if (DBG) Log.d(TAG, "startService. WebSocketService");
        Intent intent2 = new Intent(context, WebSocketService.class);
        context.startService(intent2);
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        if (DBG) Log.d(TAG, "consoleWeb boot complete ....");
        clearCacheDir(context); // 清理缓存目录

        if (DBG) Log.d(TAG, "onReceive. [intent=" + intent);
        String action = intent.getAction();
        // 处理自定义重启服务广播
        if ("com.color.intent.action.RESTART_SERVICE".equals(action)) {
            if (DBG) Log.d(TAG, "onReceive. [RESTART service.");
            stopService(context);
            startService(context);
        } else {
            // 开机完成后启动服务
            startService(context);
        }
    }

    /**
     * 设备上电启动时清理临时缓存文件
     * @param context 上下文
     */
    private void clearCacheDir(Context context) {
        File cacheDir;
        // 优先使用外部存储缓存目录，否则使用内部缓存
        if (Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())) {
            cacheDir = context.getExternalCacheDir();
        } else {
            cacheDir = context.getCacheDir();
        }
        if (DBG) Log.d(TAG, "clearDir. cacheDir absolutePath= " + cacheDir.getAbsolutePath());

        // 清空缓存目录下所有文件（root权限执行rm -rf）
        if (cacheDir.isDirectory() && cacheDir.listFiles().length > 0) {
            String cmd1 = "rm -rf " + cacheDir.getAbsolutePath() + "/*";
            runAsRoot(new String[]{cmd1});
        }
    }
}

public static Process runAsRoot(String[] cmds) {
    Process p = null;
    try {
        p = Runtime.getRuntime().exec("su");

        DataOutputStream os = new DataOutputStream(p.getOutputStream());
        // 临时修改权限掩码
        os.writeBytes("umask 000\n");
        for (String tmpCmd : cmds) {
            os.writeBytes(tmpCmd + "\n");
        }
        os.writeBytes("exit\n");
        os.close();
    } catch (IOException e) {
        Log.e(TAG, "runAsRoot: ", e);
    }
    return p;
}
```