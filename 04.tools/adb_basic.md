# adb 基本用法

### ubuntu 需要adb 有root权限才能使用

  chown root:root adb

###基本用法

+adb shell  ——登录到手机，可以执行各种linux命令。
+adb remount(需要完全root）
+adb push  object   /dest
+adb pull  object   desc
+adb reboot bootloader ——重启手机到fastboot模式
+adb reboot recovery ——重启手机到recovery模式
+adb install xxx.apk ——安装当前目录下的apk包到手机


###log

    adb logcat -s [TAG 过滤] //过滤
                -v time     //时间戳, thread, tag, process
                -c 清缓存
                -d 打完退出
                -b 指定buffer, main, system , radio, events
                -B  以二进制形式输出日志
                -g 查看缓冲区信息
                -t 10 // 输出最近10行后退出

#### tag 过滤

    adb logcat -s system.out

#### 清除缓存

    adb logcat -c

### 打完退出

    adb logcat -d

### 指定buffer

默认 main system

    adb logcat -b main
    adb logcat -b system
    adb logcat -b radio
    adb logcat -b events


adb logcat -b crashes   // only see crashes

### 指定手机上的文件

使用如下命令可以执行后断开PC和手机持续收集LOG。
    shell@pc$ adb shell  
    shell@android$ logcat -f /sdcard/log.txt &   #这里的&符号表示后台执行，别少了。  
    shell@android$ exit

###过滤

过滤项格式 : <tag>[:priority] , 标签:日志等级, 默认的日志过滤项是 " *:I " ;

adb logcat WifiHW:D dalvikvm:I *:S


忽略大小写过滤

adb logcat | grep -i wifi
