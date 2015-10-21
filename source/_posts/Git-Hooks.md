title: Git 钩子
date: 2015-10-16
categories: [Git 魔导之路]
tags: [Git]
toc: true

---

![Git 钩子](http://7xinjg.com1.z0.glb.clouddn.com/git-tips-hookhero.svg)

Git 钩子是在一个 Git 仓库中，在每一次特定事件触发时自动运行的脚本。它允许我们自定义 Git 的内部行为，并在开发的生命周期的关键时间点触发自定义行为。

<p><image
	src="http://7xinjg.com1.z0.glb.clouddn.com/git-tips-hook01.svg"
	alt="通过链接到版本控制的脚本维护钩子"
	width="80%"/></p>

Git 钩子通常的使用案例包含支持一个提交策略，根据仓库状态替换项目环境，以及实现连续集成的工作流。但由于脚本是可以任意指定的，所以我们几乎可以使用 Git 的钩子来自动化或优化我们开发工作流的方方面面。

本文的一开始我们先从概念上概述一下 Git 钩子是怎么工作的；之后我们会研究一些使用在本地和服务端仓库的最流行的钩子的使用。

<!--more-->

# 概念概述

所有的 Git 钩子都是普通的脚本，只是在仓库的特定时机会被 Git 执行。这是得钩子们的安装和配置十分容易。

钩子可以放置在本地或服务端的仓库中，它们仅对各自的仓库行为做出相应并执行。我们会在本文下面的部分具体看一下钩子的类型。应用在本地和服务端的钩子的配置会在余下的章节中进行讨论。

## 安装钩子

钩子存放在每个 Git 仓库的 `.git/hooks` 目录。当我们初始化仓库时，Git 会自动生成此目录并在里面放置一些示例脚本。如果你去看一眼 `.git/hooks` 目录的内容，就会看到如下的文件：

	applypatch-msg.sample       pre-push.sample
	commit-msg.sample           pre-rebase.sample
	post-update.sample          prepare-commit-msg.sample
	pre-applypatch.sample       update.sample
	pre-commit.sample

这里提供了许多可用的钩子，但是 `.simple` 的后缀使它们都不能默认执行。想要『安装』某一个钩子，我们所需要做的仅仅是移除 `.simple` 的扩展名。或者如果你动手撸了一个新脚本，那你可以将脚本添加到路径里，并用上面的文件名来命名你的脚本，记得去掉 `.simple` 后缀哟。

我们安装一个简单的 `prepare-commit-msg` 钩子作为示例。删除文件的 `.simple` 后缀，并将下面的内容添加到文件中：

```bash
#!/bin/sh

echo "# 请输入一条有用的提交信息！" > $1
```

钩子文件必须可执行，因此如果我们是徒手撸的脚本，我们需要更改文件的权限。例如，如果想要使 `prepare-commit-msg` 文件可执行，就需要运行下面的命令：

```bash
chmod +x prepare-commit-msg
```

现在我们应该能在每次运行 `git commit` 的时候看到默认的提交信息（就是上文脚本里的『请输入一条有用的提交信息』）。我们将在 **准备提交信息** 一章仔细研究其工作机制；而现在我们只需要臭美一下：我们已经能自定义 Git 内部的方法啦！

内置的样例脚本是非常有用的参考，它们展示了传递给每个钩子的参数详情（这些参数在钩子之间顺序传递）。

## 脚本语言

内置的脚本多是 shell 或 perl 脚本，但是只要我们编写的脚本能够执行，就可以使用任何喜欢的脚本语言。每个脚本文件的 **事务行（shebang line）** 定义了该文件的解释方式。因此想要使用其他的脚本语言，我们只需要改变事务行中解释器的路径就可以了。

举例来说，我们可以在 `prepare-commit-msg` 文件中编写一个可执行的 Python 脚本来替代 shell 脚本。下面的钩子和上一节的 shell 脚本的执行效果完全一致。

```python
#!/usr/bin/env python

import sys, os

commit_msg_filepath = sys.argv[1]
with open(commit_msg_filepath, 'w') as f:
    f.write("# Please include a useful commit message!")
```

留心一下文件的第一行指到了 Python 的解释器；同时，我们使用 `sys.argv[1]` 来代替 `$1` 来指代传入脚本的第一个参数（我们会在稍后对此进行详述）。

这是 Git 钩子的一个非常给力的特性，使得我们可以使用我们喜欢用的的任何脚本语言来编程。

## 钩子的作用域

钩子在存在于每一个 Git 仓库，且当我们运行 `git clone` 时它们 **不会** 被复制到新仓库中。而且由于钩子是本地的，它们就可以被任何有仓库访问权限的人修改。

对一个团队的开发者配置钩子时，有一个很重要的注意点。首先，我们需要找到一种方式来确保钩子在我们的团队成员之间实时更新；其次，我们不能强迫开发者来都用特定的方式来提交，顶多只能鼓励大家这么做。

维护一个开发团队的钩子可是有点棘手，因为 `.git/hooks` 目录不会随着项目的其他部分复制的，也不会被版本控制。对上面俩问题的最简单的解决方案就是把钩子存在实际的项目目录中（`.git` 目录之外），这就可以让我们像任何其他版本控制文件一样来编辑钩子。为了安装钩子，我们可以创造一个链接将其链接到 `.git/hooks`，或者是当有更新的时候，简单的将其复制粘贴到 `.git/hooks` 目录中。

<p><image
	src="http://7xinjg.com1.z0.glb.clouddn.com/git-tips-hook02.svg"
	alt="在提交创建的过程执行钩子"
	width="80%"/></p>

另外，Git 还提供一个[模板目录](http://git-scm.com/docs/git-init#_template_directory)的机制来更加便捷的自动安装钩子。模板目录的所有文件和目录在每次使用 `git init` 和 `git clone` 时都会被复制进 `.git` 目录里。

下文所述的所有本地钩子都可以被仓库管理员替换或删除，这完全取决于每个团队成员是否实际使用钩子。我们最好记住这一点，把 Git 钩子当做一个方便的开发者工具而非一个严格执行的开发政策。

即便如此，我们也可以用服务端钩子来拒绝那些不符合标准的提交。我们会在稍后讨论这一点。

# 本地钩子

本地钩子只影响它们所在的仓库。当我们读到这一节时，要记住每位开发者都可以替换其本地的钩子，我们不能使用钩子作为一种强制提交策略。然钩子却使得开发人员更容易坚持既定的开发方针。

在这一节里，我们将展示6个最常用的本地钩子：

- `pre-commit`
- `prepare-commit-msg`
- `commit-msg`
- `post-commit`
- `post-checkout`
- `pre-rebase`

前四个钩子允许我们在整个提交的生命周期中插入，后面两个允许我们分别为 `git checkout` 和 `git rebase` 执行一些额外的行为或者安全检查。

所有的以 `pre-` 为前缀的钩子都允许我们在后缀的行为 **即将发生** 的时候改变这些行为；而以 `post-` 为前缀的钩子仅用于通知。

我们也会看到一些实用的技术来解析钩子参数，以及实用底层 Git 命令请求仓库信息。

## 预提交钩子 Pre-Commit

`pre-commit` 的脚本在每次运行 `git commit` 命令时，在 Git 要求开发者填写提交信息和生成提交对象之前执行。我们可以用这个钩子在快照将要被提交的时刻对其进行检查。比方说，我们也许想要在此时运行一些自动测试，以防止本次提交破坏现有功能。

`pre-commit` 脚本没有传入参数，并且以非零状态退出（**译者注：** shell 脚本成功运行会返回零，非零状态表示运行失败）会终止整个提交。让我们看一下一个简化的内置 `pre-commit` 钩子。这个脚本在发现有空白符的错误时会终止整个提交，空白符错误定义在 `git diff-index` 命令中（行尾空白符、仅有空白符的行、行首缩进的 tab 后面有空格等格式在默认情况下都会被认为是错误的）。

```bash
#!/bin/sh

# Check if this is the initial commit
if git rev-parse --verify HEAD >/dev/null 2>&1
then
    echo "pre-commit: About to create a new commit..."
    against=HEAD
else
    echo "pre-commit: About to create the first commit..."
    against=4b825dc642cb6eb9a060e54bf8d69288fbee4904
fi

# Use git diff-index to check for whitespace errors
echo "pre-commit: Testing for whitespace errors..."
if ! git diff-index --check --cached $against
then
    echo "pre-commit: Aborting commit due to whitespace errors"
    exit 1
else
    echo "pre-commit: No whitespace errors :)"
    exit 0
fi
```

为了使用 `git diff-index`，我们需要指出需要用来做比较的提交引用。这个引用通常来说是 `HEAD`，但是在初始提交的时候不存在 `HEAD` 指针，因此我们的第一个任务就是处理这个边界值。我们使用 [`git rev-parse --verify`](https://www.kernel.org/pub/software/scm/git/docs/git-rev-parse.html) 命令，该命令可以检查参数是否是一个可用引用。`>/dev/null 2>&1` 的部分表示不显示所有 `git rev-parse` 命令的输出。将 HEAD 或是初始提交的空提交对象存放在 `against` 变量中来给 `git diff-index` 使用，而 `4b825d...` 这个哈希值是一个魔术提交 ID，表示一个空提交。

`git diff-index --cached` 命令将当前暂存区与一个提交进行对比，传入 `--check` 选项表示我们要求在改动包含空白符错误时给出警告。如果存在空白符错误，我们终止提交并返回退出状态 `1`，否则我们以 `0` 退出，且提交工作流正常工作。

【添加注释详细解释一下】

这仅仅是 `pre-commit` 钩子的一个简单例子：使用 Git 命令来在由请求提交引入的更改上运行测试。我们同样可以在 `pre-commit` 钩子中通过执行其他脚本来做任何想做的事情，例如运行一个第三方测试组件，或者使用 Lint 来检查代码格式。

## 准备提交信息 Prepare Commit Message

`prepare-commit-msg` 钩子在 `pre-commit` 钩子之后调用，用于给提交的文本编辑器填充提交信息。这是一个替换压缩提交和合并提交时自动生成提交信息的一个好时机。

有 1 到 3 个参数会被传递给 `prepare-commit-msg` 脚本：

1. 包含提交信息的临时文件名。我们通过替换这个文件来更改提交信息。
2. 提交的类型。包含 `message` （带有 `-m` 或 `-F`）、template （带有 `-t`）、`merge`（如果提交是一个合并提交）或者 `squash` （如果提交是从其他提交中合并的）。
3. 相关提交的 SHA1 哈希值。只有带有 `-c`、`-C` 或 `--amend` 的提交会给出这个参数。

【添加注释详细解释一下】

和 `pre-commit` 一样，如果脚本以非零状态退出会中止提交。

在上一节我们已经看到了一个简单的编辑提交信息的例子，但是还是让我们看一下一个更加有用的脚本。当使用 issue 来跟踪问题时，惯例是在一个单独的分支里解决一个对应的问题。如果我们在分支名称上包含了 issue 号，我们就可以编写一个 `prepare-commit-msg` 脚本来在这一分支的每一个提交上都自动导入 issue 号。

```python
#!/usr/bin/env python

import sys, os, re
from subprocess import check_output

# Collect the parameters
commit_msg_filepath = sys.argv[1]
if len(sys.argv) > 2:
    commit_type = sys.argv[2]
else:
    commit_type = ''
if len(sys.argv) > 3:
    commit_hash = sys.argv[3]
else:
    commit_hash = ''

print "prepare-commit-msg: File: %s\nType: %s\nHash: %s" % (commit_msg_filepath, commit_type, commit_hash)

# Figure out which branch we're on
branch = check_output(['git', 'symbolic-ref', '--short', 'HEAD']).strip()
print "prepare-commit-msg: On branch '%s'" % branch

# Populate the commit message with the issue #, if there is one
if branch.startswith('issue-'):
    print "prepare-commit-msg: Oh hey, it's an issue branch."
    result = re.match('issue-(.*)', branch)
    issue_number = result.group(1)

    with open(commit_msg_filepath, 'r+') as f:
        content = f.read()
        f.seek(0, 0)
        f.write("ISSUE-%s %s" % (issue_number, content))
```

首先，上面的 `prepare-commit-msg` 钩子展示了我们怎么获取传入脚本的所有参数。之后，它调用了 `git symbolic-ref --short HEAD` 来获取当前分支。如果分支名字以 `issue-` 开头，脚本就会重写提交信息文件的内容，在开头的一行引入 issue 号。因此，如果你的分支名是 `issue-224`，运行脚本就会生成下列提交信息：

```bash
ISSUE-224 

# Please enter the commit message for your changes. Lines starting 
# with '#' will be ignored, and an empty message aborts the commit. 
# On branch issue-224 
# Changes to be committed: 
#   modified:   test.txt
```

在使用 `pre-commit-msg` 钩子时，需要记住的一点是在使用者通过 `-m` 参数运行 `git commit` 命令时本钩子依然会被执行。这就意味着在使用 `-m` 输入提交信息的时候，上面的脚本会在提交信息里直接加入 `ISSUE-[#]` 的内容，用户也没法编辑它。我们可以通过判断第二个参数（`commit_type`）是否等于 `message` 的方式来处理这个特殊情况。

然后，在不带 `-m` 参数的情况下，`prepare-commit-msg` 钩子也会允许用户编辑自动生成的信息，所以这更多的是一个提供便利的脚本而非强制执行的提交政策。如果想要强制性的政策，我们需要在下一节讨论的 `commit-msg` 钩子。


































