---
layout: post
title: "使用rebase修改历史commit"
date: 2014-05-12 22:32:17 +0800
comments: true
categories: git 
---
最近在使用gerrit做code review，经常遇到的一个情况是需要更改已经commit的代码。

比如对于如下形式的branch tree:

	A -- B -- C -- D  (develop)

问题来了，gerrit上得到了反馈，C上的代码有问题，需要修改。而秉承着“一步一commit“的守则，已经有新的commit提交上去了，目前仓库和远程仓库的HEAD都指向了D。

虽然可以暂时把B merge进去，然后再直接提交一个新commit(E)去修复B上的bug的，但这显然不够优雅，而且把明知道有bug的commit也merge进去本身也是存在风险的，这时，git rebase就派上用场了。

整个主要过程分为四个步骤：

1. 修改C中的bug，commit到本地repo为E
2. 输入命令``git rebase -i HEAD~3`` 
3. 根据提示，调整commit顺序，将最新的commit放到C之后，根据提示将pick改为squash，保存退出。（若存在冲突则处理并git add，继续``git rebase --continue``）
4. ``git push --force origin develop`` # 假设目标分支为develop分支


git rebase本来是用来处理分支衍合的，具体原理可以参考官方手册：

> http://git-scm.com/book/en/Git-Branching-Rebasing

这里我们实际做的是重新调整合并了当前分支的commit顺序，举个例子：

当前分支名为myrebase，文件rebase_test.py与两个commit相关：
	
	# commit A
	print ('hello rebase')
	# commit B
	print ('bye')
	
现欲在commit A增加为：

	print ('May 18th')
	
依次做以下操作：

- 修改rabase_test.py
- 新提交一个commit C

	``git commit -am "fix on commit A" ``

- 使用命令

	``git rebase -i HEAD~3``
	
> -i 表示交互式rebase

> HEAD~3 表示对HEAD之前的三次commit进行rebase

也可以使用``git rebase -i $commit-id-before-commit-A``

其中$commit-id-before-commit-A为commit A之前commit的id，结果如下：

> pick 5b55a02 first commit add hello rebase

> pick 2f5c48f second commit add bye

> pick 225e901 fix on commit A

> \# Rebase 7f8e5eb..225e901 onto 7f8e5eb

> \#

> \# Commands:

> \#  p, pick = use commit

> \#  r, reword = use commit, but edit the commit message

> \#  e, edit = use commit, but stop for amending

> \#  s, squash = use commit, but meld into previous commit

> \#  f, fixup = like "squash", but discard this commit's log message

> \#  x, exec = run command (the rest of the line) using shell

> \#

> \# These lines can be re-ordered; they are executed from top to bottom.

> \#

> \# If you remove a line here THAT COMMIT WILL BE LOST.

> \#

> \# However, if you remove everything, the rebase will > be aborted.

> \#

> \# Note that empty commits are commented out


- 根据注释中对各参数的解释，将前三行改为

> pick 5b55a02 first commit add hello rebase

> squash 225e901 fix on commit A

> pick 2f5c48f second commit add bye

- 若存在冲突，则处理冲突后``git commit add .``并``git rebase --continue``
- 由于使用了squash参数，根据提示调整commit信息
- 看到rebase successful消息后，使用``git push --force origin myrebase``

整个过程有可能遇到很多问题，要多利用``git diff``和 ``git log``来分析