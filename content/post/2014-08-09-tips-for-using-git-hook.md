---
layout: post
title: "Tips for using Git hook"
date: 2014-08-09T13:27:00
draft: false
comments: true
categories: Tips
published: true
toc: false
---

## 前言


Git大法好, 前段时间帮助公司QA做客户端自动化测试搭建, 过程中实践了些Git Hook的实用方法,想想还是有必要记录下来.

<!-- more -->

## **Git pre-commit hook**: 悬崖勒马, 提交前做好测试.

### 前期准备

首先创建 hook script (`pre-commit.sh`) 在当前工程根目录下, 当然也可以放在自命名的文件夹中,不过推荐放在项目根目录下面, 保证所有的开发人员能够明显的知道这个文件的存在.
然后对这个文件做软连接, 方便后续不同的开发人员在提交代码前运行同一套测试并保持脚本的同步更新.

``` bash
ln -s ../../pre-commit.sh .git/hooks/pre-commit
```

首先我们要确保 `git commit` 前项目代码是不是经过测试的, 通过 `git stash` 来暂存我们的工作区和暂存区内容,经过 Test 之后再进行恢复.

``` bash
# pre-commit.sh
git stash -q --keep-index

# Test prospective commit
...

git stash pop -q
```

### 每次提交前运行项目的测试脚本, 保证项目处于可用性状态.

当然最好把测试脚本写在单独一个脚本文件中 (`tests.sh`), 并通过输入输出参数来反馈当前测试结果. 例如这样一个 `Django` 工程的测试脚本:


``` bash
# tests.sh
./manage.py test --settings=settings_test -v 2
```

那么最终的基本 `pre-commit` 文件可能是这个样子:


``` bash
# pre-commit.sh
git stash -q --keep-index
./tests.sh
test_result=$?
git stash pop -q
[ $test_result -ne 0 ] && exit 1
exit 0
```

### 如何在 Commit 时手动阻止 pre-commit 的执行?

Git大法好, 在 `git commit` 的时候添加 `--no-verify` 参数即可阻止 `pre-commit` 的执行.

当然为了偷懒, 可以写个 `aliases`.


``` bash
# ~/.bash_aliases
alias gc='git commit'
alias gcv='git commit --no-verify'
```

## **Create a global git commit hook**: 恶毒的在每个新建仓库中添加通用 hook.

人类懒惰的本性和不满足的本性是趋势科技发展的源泉...

1. 首先配置 `git templates` :

``` bash
git config --global init.templatedir '~/.git-templates'
```

通过配置 `git templates`, 在每次 `git init` 创建仓库的时候会把 `~/.git-templates` 下的文件拷贝到 `.git/` 目录下面.

2. 创建一个 hooks 文件夹:

``` bash
mkdir -p ~/.git-templates/hooks
```

3. 配置全局 hook 模板:
例如当前在 `~/.git-templates/hooks` 下面有个 `post-commit` hook 文件,可能是这样:

``` bash
#!/bin/sh
# Copy last commit hash to clipboard on commit
git log -1 --format=format:%h | pbcopy
# Add other post-commit hooks
```

4. 最后确保 hook 文件有可执行的文件属性:

``` bash
chmod a+x ~/.git-templates/hooks/post-commit
```

5. 在已存在的 repo 中使用全局 hook 仅仅重新 `git init` 即可.

  (Note: 如果当前 git repo 中已经存在一个同名的 hook 时, 默认是不会使用全局 hook 做覆盖的.)

