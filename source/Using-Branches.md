title: 使用分支
date: 2015-10-05
categories: [Git 魔导之路]
tags: [Git]
toc: true

---

![使用分支](http://7xinjg.com1.z0.glb.clouddn.com/git-coll-brhero.svg)

这篇教程对 Git 的分支进行了全面详尽的介绍。首先我们会研究下如何新建分支，其过程和获取一个新的项目历史有点类似；之后我们会了解到如何使用 `git checkout` 来选择分支；最后我们会学习到 `git merge` 命令是如何将几个相互独立的分支上进行整合的。
 
在阅读文本时，请记住 Git 的分支与 SVN 分支的不同。SVN 的分支是用来处理偶尔会进行的大规模开发才使用的，而 Git 的分支确是我们日常工作流不可或缺的一部分。

<!--more-->

# git branch

一个分支就是开发中的一条独立的路线。Git 的基本过程包含：编辑/暂存/提交，这一点我们之前在『保存更改』一节中有所提及。而分支作为 Git 最基本过程的抽象，是本节的首要模块。我们可以认为分支就是一种获取崭新工作目录、暂存区和项目历史的一种方式。新的提交记录在当前分支的历史上，同时也记录在整个项目历史的分岔之中。

`git branch` 命令允许我们创建、展示、重命名和删除分支，却不允许我们在不同的分支之间进行切换，或者将一个分岔的历史记录接回主干上。因此 `git branch` 命令需要与 `git checkout` 命令以及 `git merge` 命令紧密结合使用。

## 使用

	git branch

该命令表示展示当前仓库的所有分支。

	git branch <branch>

该命令表示创建一个叫做 `<branch>` 的新分支，但 **不会** 检出到新分支上。

	git branch -d <branch>	

该命令表示删除一个指定分支。这是一个安全的操作，因为 Git 不允许我们删除哪些还有没合并更改的分支。

	git branch -D <branch>

该命令表示强制删除指定的分支，不考虑该分支是否有没合并的更改。这条命令的使用场景是我们想彻底抛弃某条开发路线上的所有提交。

	git branch -m <branch>

将当前分支重命名为 `<branch>`。

## 详述

在 Git 中，分支是我们每天开发过程的一部分。当我们想要添加一个新特性或修复一个 bug，不管这改动大还是小，我们都需要产生一个新分支并在上面封装我们的改动。这确保了不稳定的代码永远不会被提交到主干代码上，并给予我们在将代码和并到主干之前整合分支历史的机会。

<p><image src="http://7xinjg.com1.z0.glb.clouddn.com/git-coll-br01.svg" width="80%"/></p>

例如上图展示了一个拥有两条独立开发路线的仓库，一条是一个小特性，另一条是一个需要长期开发的特性。通过在分支中开发这些特性，不仅可以保证它们的开发得以并行，同时也保证了主分支 `master` 上不会产生有问题的代码。

### 分支指针

Git 分支背后的实现相比 SVN 的实现模式更加轻量。Git 将分支存储为对一个提交的引用，而非 SVN 式的将文件从一个目录拷贝到另一个目录。换言之，一个分支表示了对一系列提交的 **指针**，而非提交的 **容器**。分支的历史是通过提交的关系推断的出的。

 这对于 Git 合并模型有着戏剧性的影响。在 SVN 中合并是基于文件的，而 Git 让合并基于提交这一更加抽象的层面上。我们可以在项目历史中确切的看到：合并是作为两个独立的提交历史的连接的。

## 示例

### 创建分支

重要的一点是了解分支仅仅是指向提交的的 **指针**。当我们创建一个分支的时候，Git 仅需要创建一个新指针——而不会对仓库产生其他的更改。因此如果我们的仓库看上去如下所示：

<p><image src="http://7xinjg.com1.z0.glb.clouddn.com/git-coll-br02.svg" width="80%"/></p>

然后我们使用下面的命令创建分支：

	git branch crazy-experiment

分支历史依然没有任何更改，我们的到的仅仅是一个指向当前提交的新指针：

<p><image src="http://7xinjg.com1.z0.glb.clouddn.com/git-coll-br03.svg" width="80%"/></p>

注意到这一操作仅仅 **创建了** 新分支。如果想要在新分支上添加提交，我们需要使用 `git checkout` 命令选择它，并使用标准的 `git add` 和 `git commit` 命令，更多信息请参阅下一章节 git checkout。

### 删除分支

当我们结束在一个分支上的工作并将其合并到主干代码上是，我们就可以自由的删除分支而不丢失任何历史记录：

	git branch -d crazy-experiment

然而，如果该分支还没有被合并，上面的命令会输出如下的信息：

	error: The branch 'crazy-experiment' is not fully merged.
	If you are sure you want to delete it, run 'git branch -D crazy-experiment'.	

这就防止了我们丢失对于这些提交的引用，丢失引用意味着我们会失去有效的进入整个开发路线的手段。如果我们确实想删除整个分支，我们可以使用大写的 `-D` 参数：

	git branch -D crazy-experiment

这将会无视分支的状态，且不输出警告地删除分支，请审慎使用这一命令。

# git checkout

`git checkout` 命令让我们在使用 `git branch` 命令建立的分支之间切换。检出一个分支使用存储在该分支的版本的文件来更新工作目录，并且告知 Git 将所有新提交记录到该分支上。我们可以把这想象成一种选择我们当前工作路线的方式。

在上一章中，我们了解了 `git checkout` 是如何被用作检查旧有提交的。检出分支的操作有点像是将工作区更新到选择的分支或者版本的状态；同时新带的更改会被保存到项目历史中，也就是说，这不是一个只读的操作。

## 使用

	git checkout <existing-branch>

检出制定的分支，这一分支必须已经被 `git branch` 命令创建。`<existing-branch>` 会成为当前分支，且会将工作目录更新至匹配状态。

	git checkout -b <new-branch>

创建并检出到 `<new-branch>`。`-b` 是一个方便的参数，会告诉 Git 在运行 `git branch <new-branch>` 之前运行 `git checkout <new-branch>`。

	git checkout -b <new-branch> <existing-branch>

和上一命令相同，不过检出的分支是基于 `<existing-branch>` 而非当前分支。

## 详述
	



































































