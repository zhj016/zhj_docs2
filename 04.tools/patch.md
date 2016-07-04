

#diff and patch

diff 和format-patch的比较：

+    兼容性：很明显，git diff生成的Patch兼容性强。如果你在修改的代码的官方版本库不是Git管理的版本库，那么你必须使用git diff生成的patch才能让你的代码被项目的维护人接受。
+    除错功能：对于git diff生成的patch，你可以用git apply --check 查看补丁是否能够干净顺利地应用到当前分支中；如果git format-patch 生成的补丁不能打到当前分支，git am会给出提示，并协助你完成打补丁工作，你也可以使用git am -3进行三方合并，详细的做法可以参考git手册或者《Progit》。从这一点上看，两者除错功能都很强。
+    版本库信息：由于git format-patch生成的补丁中含有这个补丁开发者的名字，因此在应用补丁时，这个名字会被记录进版本库，显然，这样做是恰当的。因此，目前使用Git的开源社区往往建议大家使用format-patch生成补丁。





##标准patch


生成两个commit之间的patch, 标准patch

git diff commit1 commit2 > mypatch.diff



git apply mypatch.diff


##git专用补丁


git format-patch -1 1bbe3c8c197a35f79bfddaba099270a2e54ea9c7 生成的patch有统计信息和git的版本信息



在当前的工作目录下生成diff.patch文件。


##2. 合入patch：

在CODE上查看代码片派生到我的代码片

    git apply diff.patch  


    patch -p1 < 0001-Added-liuxingde-test.patch


合入当前目录下的diff.patch文件。

##3. git am

git am -3 xxx.patch


在使用git-am之前， 你要首先git am –abort 一次，来放弃掉以前的am信息，这样才可以进行一次全新的am。
不然会遇到这样的错误。

    .git/rebase-apply
    still exists but mbox given.

git-am 可以一次合并一个文件，或者一个目录下所有的patch，或者你的邮箱目录下的patch.

下面举两个例子：
你现在有一个code base： small-src, 你的patch文件放在~/patch/0001-trival-patch.patch

      cd small-src
      git am ~/patch/0001-trival-patch.patch



##搜索commit

git log --oneline --no-merges --author=yunong


## git cherry-pick


## format-patch


    # git format-patch -M master         // 当前分支所有超前master的提交

    # git format-patch -s 4e16                // 某次提交以后的所有patch, --4e16指的是SHA1

    # git format-patch -1                   //  单次提交

    # git format-patch -3                    // 从master往前3个提交的内容，可修改为你想要的数值

    # git format-patch –n 07fe            // -n指patch数，07fe对应提交的名称, 某次提交（含）之前的几次提交
    # git format-patch -1 commit-id   // + commit-id 选定的那个commit打patch


## apply patch

应用patch：
先检查patch文件：

    # git apply --stat newpatch.patch

检查能否应用成功：

    # git apply --check  newpatch.patch

打补丁：

    # git am --signoff < newpatch.patch

(使用-s或--signoff选项，可以commit信息中加入Signed-off-by信息)
