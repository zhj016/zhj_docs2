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
