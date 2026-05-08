# Linux & ADB





## Linux


## ADB


## 场景

### 升级设备

CLT升级脚本
```shell
#!/system/bin/sh
# 脚本解释器声明：指定使用Android/嵌入式系统的sh解释器执行本脚本
# 作用：告诉系统该用哪个程序来运行这个脚本，/system/bin/sh是安卓系统默认的shell路径

echo 开启调试模式
# 日志输出：打印"开启调试模式"到控制台
# 作用：标记脚本进入调试阶段，便于人工观察脚本执行进度

set -x
# 开启调试输出模式（xtrace）
# 作用：执行后续每一行命令时，会先打印「命令本身」再执行，执行结果也会输出
# 实操价值：调试脚本时能清晰看到哪一步出错，是定位问题的核心开关

test_file=/system/web/jetty/webapps/ROOT/test_file.log
# 定义变量：将测试文件路径赋值给test_file
# 作用：后续用这个文件验证根目录是否真的挂载为可读写（rw），避免"挂载成功但实际不可写"的假阳性

copy_files() {
    # 定义函数：封装所有文件拷贝、权限修改的核心逻辑
    # 作用：把升级的核心操作封装成函数，便于调用和重试（脚本后续会重试该函数）

    mount -o rw,remount "/dev/root" "/"
    # 核心挂载命令：将根分区（/dev/root）重新挂载到根目录（/），权限改为可读写（rw）
    # 背景：安卓系统根目录默认是只读（ro）挂载，无法修改/system目录下的文件，这行是升级的前提
    # 参数说明：-o 表示指定挂载选项；rw=可读写；remount=重新挂载（无需先卸载）

    echo 进行拷贝
    # 日志输出：标记进入文件拷贝阶段
    # 作用：在控制台打印进度，便于判断拷贝操作是否开始执行

    cp -f /data/local/web/package/web/dist/index.html /system/web/jetty/webapps/ROOT/index.html
    # 强制拷贝文件：将升级包中的index.html覆盖到jetty的根网页目录
    # 参数说明：-f=强制覆盖（即使目标文件存在/只读，也直接替换）
    # 实操意义：更新前端主页文件，是升级的核心文件之一

    rm -rf /system/web/jetty/webapps/ROOT/static/*
    # 递归强制删除目录下所有文件：清空旧的静态资源（js/css/img等）
    # 参数说明：-r=递归处理目录；-f=强制删除（忽略不存在的文件，不报错）
    # 作用：避免旧静态资源和新资源混合，导致前端样式/功能异常

    cp -rf /data/local/web/package/web/dist/static/* /system/web/jetty/webapps/ROOT/static
    # 递归强制拷贝：将新的静态资源全部拷贝到目标目录
    # 参数说明：-r=递归拷贝目录；-f=强制覆盖
    # 作用：替换所有前端静态资源，完成前端资源升级

    chmod -R 755 /system/web/jetty/webapps/ROOT/static
    # 递归修改权限：将static目录及其子文件/目录权限设为755
    # 参数说明：-R=递归；755=所有者（root）可读可写可执行，其他用户可读可执行
    # 实操意义：保证jetty服务器能读取静态资源，同时避免权限过大导致安全问题

    rm -rf /system/web/jetty/webapps/ROOT/*.worker.js
    # 递归强制删除：清空旧的前端worker脚本（后台运行的js文件）
    # 通配符说明：*.worker.js 匹配所有以.worker.js结尾的文件
    # 作用：清理旧的后台脚本，避免和新脚本冲突

    cp -rf /data/local/web/package/web/dist/*.worker.js /system/web/jetty/webapps/ROOT/
    # 递归强制拷贝：将新的worker脚本拷贝到目标目录
    # 作用：更新前端后台运行的脚本，保证前端异步功能正常

    chmod 755 /system/web/jetty/webapps/ROOT/*.worker.js
    # 修改权限：将新的worker脚本权限设为755
    # 作用：保证jetty能执行这些脚本，同时控制权限安全

    cp -f /data/local/web/package/server/iJetty_config/webdefault.xml /system/web/jetty/etc/webdefault.xml
    # 强制拷贝：替换jetty的web默认配置文件
    # 作用：更新jetty服务器的核心配置，适配新的前端/应用逻辑

    cp -f /data/local/web/package/server/iJetty_config/WEB-INF/web.xml /system/web/jetty/webapps/ROOT/WEB-INF/web.xml
    # 强制拷贝：替换jetty的web应用配置文件
    # 作用：更新ROOT应用的路由、过滤器等配置，保证升级后应用正常运行

    cp -f /data/local/web/package/global.config /system/web/jetty/webapps/ROOT/global.config
    # 强制拷贝：替换全局配置文件
    # 作用：更新系统级别的全局参数（如接口地址、运行模式等）

    chmod 755 /system/web/jetty/webapps/ROOT/global.config
    # 修改权限：将全局配置文件设为755
    # 作用：保证应用能读取该配置文件

    cp -f /data/local/web/package/web2_backup.sh /system/web/jetty/webapps/ROOT/web2_backup.sh
    # 强制拷贝：替换备份脚本
    # 作用：更新升级后的备份逻辑，保证后续备份操作可用

    chmod 755 /system/web/jetty/webapps/ROOT/web2_backup.sh
    # 修改权限：将备份脚本设为755
    # 作用：赋予脚本可执行权限，保证能运行

    cp -f /data/local/web/package/server/apk/iJetty.apk /system/priv-app/iJetty/iJetty.apk
    # 强制拷贝：替换iJetty应用的APK文件
    # 作用：升级iJetty应用程序，是安卓端核心应用升级

    rm -rf /system/priv-app/iJetty/lib
    # 递归强制删除：清空iJetty应用的旧依赖库目录
    # 作用：清理旧的so库（原生库），避免和新APK的库冲突

    # 测试文件系统是否可写
    if echo "write test" > "$test_file" 2>/dev/null; then
        # 尝试向测试文件写入内容：验证根目录是否真的可写
        # 2>/dev/null：将错误输出重定向到空（避免写入失败时打印错误日志）
        # 逻辑：如果写入成功，说明挂载rw生效

        rm -f "$test_file"
        # 删除测试文件：清理临时文件，避免残留

        return 0
        # 函数返回0：表示拷贝操作成功（0在shell中代表成功）
    else
        return 1
        # 函数返回1：表示拷贝操作失败（非0代表失败）
    fi
}

if copy_files; then
    # 调用copy_files函数，并判断返回值
    # 逻辑：如果函数返回0（成功），执行以下代码

    echo "文件系统 $test_file 可写"
    # 日志输出：确认文件系统可写，拷贝成功
else
    # 如果函数返回非0（失败），执行以下代码

    echo "文件系统 $test_file 不可写，重新尝试拷贝"
    # 日志输出：提示拷贝失败，准备重试

    sleep 2
    # 等待2秒：给系统留出挂载状态恢复的时间，避免立即重试导致再次失败

    copy_files
    # 再次调用copy_files函数：容错重试，提升升级成功率
fi

echo 删除拷贝目录
# 日志输出：标记进入清理阶段

rm -rf /data/local/web/package
# 递归强制删除：删除升级包源目录
# 作用：升级完成后清理临时文件，节省安卓系统存储空间

sync
# 同步磁盘写入：强制将内存中的文件修改刷到物理磁盘
# 背景：安卓系统为了性能，文件修改会先存在内存缓存，sync可避免断电/重启导致数据丢失

echo 查看升级结果
# 日志输出：标记进入结果校验阶段

find /system/web/jetty/webapps/ROOT -type f 
# 查找文件：列出目标目录下所有普通文件
# 参数说明：-type f = 仅查找普通文件（排除目录）
# 作用：打印升级后的文件列表，便于人工核对是否所有文件都拷贝成功

chmod -R 777 /system/priv-app/iJetty
# 递归修改权限：将iJetty应用目录权限设为777
# 参数说明：777=所有用户（root/普通用户）可读可写可执行
# 实操意义：最大化权限，避免iJetty应用因权限不足无法启动

sync
# 再次同步磁盘写入：确保权限修改生效

# mount -o ro,remount "/dev/root" "/"
# 注释掉的只读挂载命令：原本用于将根目录还原为只读（ro）
# 注释原因：你可能需要保持根目录可写，或后续手动还原，避免自动ro导致其他操作失败

sync
# 同步磁盘写入：确保挂载状态（即使注释了ro，也同步一次，兜底）

echo 关闭调试模式
# 日志输出：标记调试阶段结束

set +x
# 关闭调试输出模式
# 作用：停止打印后续命令的执行日志，减少控制台冗余输出

sleep 1
# 等待1秒：给系统留出进程清理时间，避免立即杀进程时目标进程未响应

kill `ps -e | grep org.mortbay.ijetty | awk -F "\ " '{print $2}'`
# 强制杀死iJetty进程：让应用重启，加载升级后的文件
# 拆解说明：
# 1. ps -e：列出所有进程
# 2. grep org.mortbay.ijetty：筛选出iJetty的进程行
# 3. awk -F "\ " '{print $2}'：以空格为分隔符，提取进程号（PID）
# 4. kill 进程号：杀死进程，iJetty会被系统自动重启（安卓特性），从而加载新文件
```

vector定制升级脚本
```shell
#!/system/bin/sh
# 适配RK3588 Android的Framework定制升级脚本
echo 开启调试模式
set -x

# 定义核心路径（根据你的实际路径修改）
# 1. 升级包存放路径（提前通过ADB push到该目录）
UPGRADE_PACKAGE="/data/local/frame_custom"
# 2. 测试文件（验证分区可写）
TEST_FILE="/system/framework/test_frame.lock"
# 3. 目标APK包名（用于后续校验）
TARGET_APK="com.vector.frontpanel"

# 核心函数：替换定制文件
copy_custom_files() {
    # ========== 第一步：解锁根分区为可读写 ==========
    # RK3588的根分区设备名（需确认：lsblk查看，通常是/dev/mmcblk0p7）
    mount -o rw,remount "/dev/mmcblk0p7" "/"
    # 验证挂载是否成功
    if ! mount | grep "/dev/mmcblk0p7 on / type .*rw," >/dev/null 2>&1; then
        echo "根分区挂载为可读写失败！"
        return 1
    fi

    echo 开始替换定制文件
    # ========== 第二步：替换Framework核心文件 ==========
    # 替换framework.jar（定制开机自启/电源管理/按键事件）
    cp -f "$UPGRADE_PACKAGE/framework.jar" "/system/framework/framework.jar"
    chmod 644 "/system/framework/framework.jar" # 系统文件标准权限

    # ========== 第三步：替换SystemUI.apk（禁用下拉状态栏） ==========
    rm -rf "/system/priv-app/SystemUI/SystemUI.apk"
    cp -f "$UPGRADE_PACKAGE/SystemUI.apk" "/system/priv-app/SystemUI/SystemUI.apk"
    chmod 644 "/system/priv-app/SystemUI/SystemUI.apk"

    # ========== 第四步：替换/安装前面板APK ==========
    # 删除旧APK
    rm -rf "/system/priv-app/FrontPanel/FrontPanel.apk"
    # 拷贝新APK（系统级APK放priv-app目录，获取更高权限）
    cp -f "$UPGRADE_PACKAGE/FrontPanel.apk" "/system/priv-app/FrontPanel/FrontPanel.apk"
    chmod 644 "/system/priv-app/FrontPanel/FrontPanel.apk"
    # 修复APK目录权限
    chmod -R 755 "/system/priv-app/FrontPanel"

    # ========== 第五步：配置开机自启脚本（简化版） ==========
    # 写入自启rc文件
    cat > "/system/etc/init/frontpanel.rc" << EOF
service frontpanel /system/bin/am start -n $TARGET_APK/.MainActivity
    class main
    user root
    group root
    oneshot
    seclabel u:r:init:s0
EOF
    chmod 644 "/system/etc/init/frontpanel.rc"

    # ========== 第六步：禁用屏幕休眠（脚本兜底） ==========
    settings put system screen_off_timeout 0
    dumpsys power setStayOn true

    # ========== 验证可写性 ==========
    if echo "custom success" > "$TEST_FILE" 2>/dev/null; then
        rm -f "$TEST_FILE"
        return 0
    else
        return 1
    fi
}

# ========== 执行核心函数 + 容错重试 ==========
if copy_custom_files; then
    echo "Framework定制文件替换成功"
else
    echo "文件替换失败，2秒后重试..."
    sleep 2
    copy_custom_files
fi

# ========== 清理临时文件 + 同步磁盘 ==========
echo 删除升级包临时目录
rm -rf "$UPGRADE_PACKAGE"
sync

# ========== 恢复根分区为只读 ==========
mount -o ro,remount "/dev/mmcblk0p7" "/"
sync

echo 关闭调试模式
set +x

# ========== 重启系统（使Framework修改生效） ==========
echo 定制完成, 3秒后重启系统...
sleep 3
reboot
```





