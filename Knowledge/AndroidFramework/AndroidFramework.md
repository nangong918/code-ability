# Android Framework



## 基本体系





## 场景


### 开机自动启动前面板 APK
修改 Framework 开机启动逻辑（底层定制）
* 找到源码路径：
```text
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
```
* 定位开机初始化方法：systemReady() 系统启动完成后执行
* 新增启动 APK 的代码:
```java
// 在systemReady()方法末尾添加
try {
// 替换为你的APK包名+主Activity
String pkgName = "com.vector.frontpanel";
        String clsName = "com.vector.frontpanel.MainActivity";
        Intent intent = new Intent(Intent.ACTION_MAIN);
    intent.addCategory(Intent.CATEGORY_LAUNCHER);
    intent.setComponent(new ComponentName(pkgName, clsName));
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    mContext.startActivity(intent);
} catch (Exception e) {
    Slog.e("AutoStart", "启动前面板APK失败：" + e.getMessage());
}
```

### 待机不息屏（禁用休眠 / 屏幕超时）
修改 Framework 电源管理（彻底禁用）
* 找到源码路径：
```text
frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java
```
* 定位屏幕超时方法：updateUserActivityTimeout()
* 修改超时时间为永久（禁用休眠）：
```java
// 找到以下代码，替换超时值为0（0表示永不超时）
private long getUserActivityTimeout(int reason) {
    // 原代码：return mScreenOffTimeoutSetting;
    return 0; // 禁用屏幕自动关闭
}
```
* 同时禁用休眠：在PowerManagerService.java的systemReady()中添加
```java
// 禁用休眠
mWakeLock = new WakeLock(WakeLock.PARTIAL_WAKE_LOCK, "NoSleepLock");
mWakeLock.acquire(); // 永久持有唤醒锁，禁止系统休眠
```

### 禁止下拉状态栏 + 返回键锁死 APK
* 源码路径：
```text
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/PhoneStatusBar.java
```
* 定位下拉触发方法：expandNotificationsPanel()
1. 注释 / 禁用下拉逻辑：
    ```java
    // 原方法：
    public void expandNotificationsPanel() {
        // 新增判断：直接返回，禁用下拉
        return;
        // 原逻辑注释掉...
    }
    // 同时禁用快速设置下拉
    public void expandSettingsPanel() {
        return;
    }
    ```
2. 返回键锁死 APK（仅目标 APK 屏蔽返回键）
   1. 源码路径：frameworks/base/core/java/android/view/KeyEvent.java
   2. 定位返回键事件：KEYCODE_BACK（键值为 4）
   3. 修改事件分发逻辑：在dispatchKeyEvent()方法中添加判断，屏蔽目标 APK 的返回键：
   ```java
    // 找到dispatchKeyEvent()方法，新增判断
    if (keyCode == KeyEvent.KEYCODE_BACK) {
        // 获取当前前台应用包名
        String pkgName = getTopActivityPackageName();
        // 替换为你的APK包名
        if ("com.xxx.frontpanel".equals(pkgName)) {
            // 消费事件，不向下传递（屏蔽返回键）
            return true;
        }
    }
      ```

