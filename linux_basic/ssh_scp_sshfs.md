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
