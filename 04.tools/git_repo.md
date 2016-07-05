#git  tig memo


##tig related
##看修改记录

  tig

  apt-get install tig

  与vi命令一致

   tig dir //check history of one folder


gitk 图形话工具

###show
tig show --pretty=fuller

git show | tig


## git diff

git diff branch1..branch2 --filename

###view switch



Key	Action
m	Switch to main view.

d	Switch to diff view.

l	Switch to log view.

g	Switch to grep view.

b	Switch to blame view.

s	Switch to status view

c   Switch to stage view

### folder

tig <folder or file name>


#git related
##统计

git log --stat

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

## diff

git diff branch1..branch2 --filename

#repo
repo forall -c ""

##下载
git clone git://code.thundersoft.com/tools/repo –b stable

##使用repo下载整个Android代码树

repo init -u manifest库的地址 -b 分支名

##建立本地的工作分支


repo start <branch_name> {<git库名(可多个) | --all}


##grep

repo grep {pattern | -e pattern} [<project>...]
在指定或所有库中查找关键字


##git hub ssh key permission issue

    git remote rm origin
    git remote add origin https://github.com/zhj016/zhj_docs2.git
    git push -u origin master
    input user name and password
