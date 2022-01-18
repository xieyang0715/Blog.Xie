---
title: git常用命令
date: 2022-01-18 10:30:46
categories: git
toc: true
---

git 添加远程仓库：

```
git remote add origin url
```

查看远程仓库库：

```
git remote
```

<!--more-->
删除远程仓库：

```
git remote rm origin //origin为仓库名
```

添加远程仓库：

```
git remote add origin var //origin为仓库名 var 仓库地址
```

重新命名分支：

```
git branch -M branchname
```

撤销上次暂存/撤销到上 n 次暂存：

```
git reset HEAD~
git reset HEAD~N
```


本地回到指定版本：

```
git revert -n 版本号
```

检出新分支到本地：

```
git checkout -b new_branch origin_branch
```

删除本地分支：

```
git branch -D local_branch
```

推送本地分支到新的远程分支：

```
git push --set-upstream origin new_branch
```

添加到上次提交过程中：

```
git commit --amend

--amend               amend previous commit
git commit --amend  # 会通过 core.editor 指定的编辑器进行编辑
git commit --amend --no-edit   # 不会进入编辑器，直接进行提交
```

合并多个 commit 为一个完整 commit（前开后闭）：

```
git rebase -i  [startpoint]  [endpoint]
```

将某一段 commit 粘贴到另一个分支上：

```
git rebase   [startpoint]   [endpoint]  --onto  [branchName]
```

日志显示在一行：

```
git log --pretty=oneline
```

还原删除分支（本地）

```
git reflog 已删除分支id
```

恢复分支 ddd94a4 名为 reback_remove_branch：

```
git checkout -b reback_remove_branch ddd94a4
```

将指定的提交（commit）应用于其他分支：

```
git cherry-pick <commitHash>
```

转移 HashA，HashB 两或多个个提交：

```
git cherry-pick <HashA> <HashB> ...
```

转移一系列的连续提交：

```
git cherry-pick A..B / git cherry-pick A^..B （包含A提交）
```

查看文件差异：

```
git diff 文件
```

**[git 文档](https://git-scm.com/book/zh/v2)**
