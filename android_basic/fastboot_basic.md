#fastboot命令基础

##fastboot no permission问题修复
sudo chown root:root fastboot
sudo chmod +s fastboot

##基本命令

###fastboot boot

fastboot boot boot.img  ——用当前目录下的boot.img启动手机，在手机boot分区损坏的情况下可以用这个正常进入系统
fastboot boot recovery.img  ——用当前目录下的recovery.img启动手机到recovery模式，这个和手机上现有的系统完全无关，只要本地的 recovery.img是以前能正常进rec的，那就绝对没问题。

###fastboot flash
fastboot flash boot boot.img  ——把当前目录下的boot.img刷入手机的boot分区。
fastboot flash recovery recovery.img  ——把当前目录下的recovery.img刷入手机的recovery分区。

###fastboot devices
fastboot devices -l

###fastboot erase

fastboot erase boot
fastboot erase system
fastboot erase data
fastboot erase cache
上面的命令也可以简化成一条命令
fastboot erase system -w

fastbppt erase -w

###fastboot reboot

###fastboot getvar <variable>   display a bootloader variable  

fastboot getver:version

fastboot getvar version:version-bootloader:version-baseband:product:serialno:secure

version 客户端支持的fastboot协议版本

version-bootloader  Bootloader的版本号

version-baseband    基带版本

product             产品名称

serialno             产品序列号

secure              返回yes 表示在刷机时需要获取签名


###fastboot reboot-bootloader   reboot device into bootloader  

###fastboot update <filename>   reflash device from update.zip  

（1）创建包含boot.img，system.img，recovery.img文件的zip包。

   （2）执行：fastboot update {*.zip}
