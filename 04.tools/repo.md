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
