#git memo

##看修改记录

  tig

  apt-get install tig

  与vi命令一致


gitk 图形话工具



##关于三个区域

committed -> 本地数据目录
modified -> 工作目录
staged -> 暂存区域

##config

三处配置，优先级由高到低：/etc/gitconfig > ~/.gitconfig > .git/config


##pull

git pull [远程库地址 [远程分支名:本地分支名]]


##fetch

git fetch [远程库地址 [远程分支名:本地分支名]]

git pull 可以用来合并分支,
git fetch 不可以合并分支


##add

git add . 提交所有修改的文件,包括新增文件,不包括删除文件
git add -u 提交所有修改文件,包括删除文件,不包括新增文件
git add -A 提交包括新增和删除文件的所有文件

##rm

git rm   从暂存区域移除,并连带从工作目录中删除指定的文件,
git rm -f  如果删除之前修改过并且已经放到暂存区域的话,则必须要用强制删除选项 –f


##reset

git reset HEAD <file>

取消已经暂存的文件


##checkout

checkout -- <file>
取消对文件的修改,把之前版本的文件复制过来重写此文件。


##git clean

删除未暂存的文件


##git commit --amend

修改最后一次提交。


##tag

git tag [-a] [tag name] [-m message] [commitID]
打标签或查看标签


##diff

git diff   查看尚未暂存的文件更新了哪些部分(和暂存区中)
git diff --cached   看已经暂存起来的文件和上次提交时的快照之间的差
异
git diff SHA1..SHA2   查看两次提交之间的区别

##log
git log –p 以patch形式显示提交


##git 命令别名

$ git config --global alias.co checkout
$ git config --global alias.br branch
$ git config --global alias.ci commit
$ git config --global alias.st status
$ git config --global alias.unstage 'reset HEAD --'
$ git config --global alias.last 'log -1 HEAD'



##合并

git merge
git rebase
get cherry-pick
