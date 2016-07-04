##ubuntu log filter

sudo apt-get install glogg
glogg

##ddms

./ddms

##hierarchyviewer

hierarchyviewer


##文件浏览

nautilus

## tee

tee指令会从标准输入设备读取数据，将其内容输出到标准输出设备，同时保存成文件。

 who | tee who.out

## awk

##代码统计

cloc --diff dir1 dir2 --exclude-dir=out,.repo,.git

##Grep

###rgrep search

#### find

  find . -name *.java | xargs grep static

#### rgrep
rgrep 指令的功能和grep 指令类似，可查找内容中包含指定的范本样式的文件。如果发现某文件的内容符合所指定的范本样式，预设rgrep指令会把含有范本样式的那一列显示出来。

  rgrep hello *

  grep -r "static" .

  -i ignore case
  -w word only
  -n line number
  -s no error
  -l list only file name
  -L list file names that not match

##web proxy

使用条款及免责声明： 仅供加速访问、提高安全性及学习交流之用，禁止用其浏览任何非法内容，凡因违规浏览而引起的任何法律纠纷，本站概不负责。


##ssh, sshfs

###ssh

  ssh xxx@<ip address>

###scp

======
从 本地 复制到 远程
======
* 复制文件：
        * 命令格式：
                scp local_file remote_username@remote_ip:remote_folder
                或者
                scp local_file remote_username@remote_ip:remote_file
                或者
                scp local_file remote_ip:remote_folder
                或者
                scp local_file remote_ip:remote_file

                第1,2个指定了用户名，命令执行后需要再输入密码，第1个仅指定了远程的目录，文件名字不变，第2个指定了文件名；
                第3,4个没有指定用户名，命令执行后需要输入用户名和密码，第3个仅指定了远程的目录，文件名字不变，第4个指定了文件名；

======
从 远程 复制到 本地
======

从 远程 复制到 本地，只要将 从 本地 复制到 远程 的命令 的 后2个参数 调换顺序 即可；

例如：
    scp root@www.cumt.edu.cn:/home/root/others/music /home/space/music/1.mp3
    scp -r www.cumt.edu.cn:/home/root/others/ /home/space/music/

###sshfs

+to mount ssh folder
  mkdir tmp
  sshfs toto@lecole.fr:Document/Blagues tmp

+ to unmount
  fusermount -u tmp

+ nj08

  sshfs zhaohj0612@192.168.65.18:/home/zhaohj0612 nj08_home
  funermount -u nj08_home

+ nj16
   sshfs tangjj1112@192.168.65.26:/home/tangjj1112 nj16_home
   funermount -u nj16_home


#tar 使用

##解包

tar xvfz name.tar.gz
tar xvfj name.tar.bz

##压缩

tar czvf folder/
tar cjvf folder/

##分割


在Linux下使用 tar 命令来将文件打包并压缩是很通常的用法了。可是Linux的文件系统对文件大小有限制，也就是说一个文件最大不能超过2G，如果压缩包的的内容很大，最后的结果就会超过2G，那么该怎么办呢？又或者压缩包希望通过光盘来进行备份，而每张光盘的容量只有700M，那么该如何存储呢？解决的办法就是将最后的压缩包按照指定大小进行分割，这就需要用到split命令了。

举例说明：
要将目录logs打包压缩并分割成多个1M的文件，可以用下面的命令：
tar cjf logs/ |split -b 1m logs.tar.bz

##合并

cat result_prefixaa result_prefixab result_prefixac > target_file
