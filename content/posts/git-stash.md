---
title: "git stash 的作用、流程与冲突处理思路"
date: 2026-02-11
tags: ["Git", "版本控制", "开发流程"]
slug: "git-stash-guide"
description: 介绍 git stash 以及与 git pull --no-rebase 的差异与适用场景。
keywords: ["git stash", "git pull --no-rebase", "git 冲突处理"]
---

这段时间跟同事一起做开发时，总是碰到分支起冲突，我去请教 AI 时，他给了我这样的一套命令，git stash,git pull,git stash pop,这个 pull 是了解的，但是剩下两个是什么呢？为什么能这样避免冲突呢？于是我就去搜索答案，发现 git stash 可以做三件事情，
- 保存当前修改（tracked 文件）
- 清空工作区
- 让你可以安全切分支
然后这个git stash pop命令是把最近一次 stash 的修改恢复到当前工作区，并且从 stash 列表里删除它。

## 我们来看看一个完整的使用流程
```bash
# 开发中
git status

# 临时保存
git stash push -m "WIP: payment module"

# 切分支
git checkout main

# 修 bug
git commit -am "hotfix"

# 切回来
git checkout dev

# 恢复修改
git stash pop
```

## 那其实有点好奇这个 stash 是存在哪里的，本质是什么？
- 存在 .git 目录里
- 实际是一个特殊的 commit
- 存在一个栈结构里

但是这种方法不是建议经常使用，很容易造成忘记这回事，最开始我用的时候没有执行最后一步 git stash pop 时吓了一跳，怎么我自己的修改全部不见了。后面才知道是少执行了一步，这也确实暴露了自己的不足。

此外，除了这个解决冲突的办法，还有一个就是git pull --no-rebase，这个 no-rebase 是拉下来以后用 merge 合并。

这两个方法的不同之处是 stash 是隐藏冲突，在冲突前隐藏起来，但是 no-rebase 是冲突发生时选择 merge 方式合并。冲突依然可能发生，但是在 merge 的过程中解决。且这个 no-rebase 也是需要手动去解决冲突的。
