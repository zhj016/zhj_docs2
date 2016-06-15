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
