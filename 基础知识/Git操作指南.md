#### Git 操作指南

##### 基本操作

一次正常提交流程：
```
git add .
git commit -m "提交信息"
git push / git push origin 分支名
```
查看日志：
git log 线型查看提交日志
git log --graph --decorate --all  图片的形式查看提交日志

git stash [--include-untracked] 保存到临时缓冲区，包括本地新增的文件,并恢复当前本地分支修改
git stash pop 恢复临时缓冲区顶部的数据到本地分支

git fetch 拉取远程分支最新修改，但是不会同步代码到本地分支

git branch 查看分支

git checkout 分支名 切换分支

git checkout -b 分支名 创建并切换分支

git checkout 文件名 丢弃修改，针对未执行“git add”的文件

##### 进阶操作

**一.合并操作merge，rebase**
**A.同一个分支，合并代码**

1.如果本地代码没有提交，还是存于暂存区，可以先git statsh 保存本地修改到缓冲区，然后git pull 拉取服务器最新代码，然后在修改冲突，在提交和推送
```
git statsh [--include-untracked]
git pull
git add .
git commit -m ""
git push
```

2.如果本地代码已提交，有两种方式：
a.采用上述1的方式，**缺点是会多一条提交记录：Merge branch xxx to xxx，优点是提交记录是按照时间顺序来的**
**b.采用git rebase**
**git base 会先找到本地分支和origin分支最新的且相同的提交为起点，在这个起点之后的本地的提交全部先checkout，并保存到一个临时的分支，然后把origin的在起点之后的提交都合并到本地，然后再逐一检测并且合并本地在起点之后提交的分支**

**注意：采用git base的好处是没有上述a点的多条提交记录的问题，整个提交的历史是线型的，缺点是时间不是按照顺序来的**
首先git fetch拉取服务器最新提交，注意此时并没有合并最新代码到本地分支，这个和git pull是有区别的，然后使用git rebase，会提示一下内容，需求applying的提交
```
firstproject git:(demo3) git rebase
First, rewinding head to replay your work on top of it...
Applying: 18
Using index info to reconstruct a base tree...
M	test.md
Falling back to patching base and 3-way merge...
Auto-merging test.md
CONFLICT (content): Merge conflict in test.md
error: Failed to merge in the changes.
Patch failed at 0001 18
hint: Use 'git am --show-current-patch' to see the failed patch

Resolve all conflicts manually, mark them as resolved with
"git add/rm <conflicted_files>", then run "git rebase --continue".
You can instead skip this commit: run "git rebase --skip".
To abort and get back to the state before "git rebase", run "git rebase --abort".

➜  firstproject git:(aaf78cf) ✗ git status
rebase in progress; onto aaf78cf
You are currently rebasing branch 'demo3' on 'aaf78cf'.
  (fix conflicts and then run "git rebase --continue")
  (use "git rebase --skip" to skip this patch)
  (use "git rebase --abort" to check out the original branch)

Unmerged paths:
  (use "git reset HEAD <file>..." to unstage)
  (use "git add <file>..." to mark resolution)

	both modified:   test.md

no changes added to commit (use "git add" and/or "git commit -a")
```

上述例子中，是提示本地修改的提交涉及到的test.md和服务端提交的有冲突，先解决冲突,解决冲突后：
```
git add . //标记冲突已解决
git rebase --continue //继续合并本地后面的提交
```
如果没有冲突，就可以直接push。


**B.不同分支，合并代码**
场景：需要将dev的代码合并到master

1.使用git merge 
```
//切换到master分支，然后使用上述A中的同分支合并代码的方式，获取最新代码
git checkout dev
git pull / git fetch + git rebase 
//然后切换到master，并merge dev
git checkout master
git merge dev
//解决冲突并提交
git commit -m "merge"
git push
```

2.使用git rebase
```
//切换到master分支，然后使用上述A中的同分支合并代码的方式，获取最新代码
git checkout dev
git pull / git fetch + git rebase
//然后切换到master，并rebase dev
git checkout master
git rebase dev
//解决冲突
git add .
git rebase --contine
//最后提交,由于改变了历史提交记录，需要强推，但是必须带上--force-with-lease，如果origin/master有新提交，此次强推会失败
git push --force-with-lease

```
如果选择 git merge 和 git rebase：
```
如果不同分支的合并使用rebase可能需要重复解决冲突，这样就得不偿失了。但如果是本地推到远程并对应的是同一条分支可以优先考虑rebase。所以我的观点是 根据不同场景合理搭配使用merge和rebase，如果觉得都行那优先使用rebase
```

**二.回撤操作**

1.git reset 
把当前分支指向另一个位置，并且相应的变动工作区和暂存区，**本地未push的提交可以直接改，已经push的提交，reset后需要强推到远程分支**

>git reset —soft [commit]	只改变提交点，暂存区和工作目录的内容都不改变
>git reset —mixed [commit]	改变提交点，同时改变暂存区的内容 默认是这个
>git reset —hard [commit]	暂存区、工作区的内容都会被修改到与提交点完全一致的状态
>git reset --hard HEAD	让工作区回到上次提交时的状态

git reset 文件名 丢弃修改，针对已执行“git add”的文件，注意和git checkout的区别

2.git revert
用一个新提交来消除一个历史提交所做的任何修改

```
git revert c011eb3c20ba6fb38cc94fe5a8dda366a3990c61
```
```
git revert是用一次新的commit来回滚之前的commit，git reset是直接删除指定的commit

看似达到的效果是一样的,其实完全不同.

第一:

上面我们说的如果你已经push到线上代码库, reset 删除指定commit以后,你git push可能导致一大堆冲突.但是revert 并不会.

第二:

如果在日后现有分支和历史分支需要合并的时候,reset 恢复部分的代码依然会出现在历史分支里.但是revert 方向提交的commit 并不会出现在历史分支里.

第三:

reset 是在正常的commit历史中,删除了指定的commit,这时 HEAD 是向后移动了,而 revert 是在正常的commit历史中再commit一次,只不过是反向提交,他的 HEAD 是一直向前的.

```

**pro git 上的一句话**
>“永远不要衍合那些已经推送到公共仓库的更新。如果你遵循这条金科玉律,就不会出差错。否则,人民群众会仇恨你,你的朋友和家人也会嘲笑你,唾弃你。” 


