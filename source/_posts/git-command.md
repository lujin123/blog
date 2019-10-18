---
title: Git常用的命令
date: 2019-05-05 10:52:53
tags: git
---

记录下`Git`中常用的命令，不用到处去找，提升下效率

## 批量删除

### 批量删除分支
```sh
# 删除本地分支
> git branch -a | grep -v -E 'master|develop' | xargs git branch -D

# 删除远程分支
> git branch -r| grep -v -E 'master|develop' | sed 's/origin\///g' | xargs -I {} git push origin :{}
```
如果某些分支不需要删除，就指定出来，不然就全部删了

**注意**：如果有些分支无法删除，是因为远程分支的缓存问题，可以使用下面的步骤

```sh
# 先查看下远程仓库和本地仓库的关系
> git remote show origin

# 有些分支会提示用下面这个命令跑一下就行了
> git remote prune
```

### 批量删除tag

```sh
# 批量删除本地tag
> git tag | xargs -I {} git tag -d {}

# 批量删除远程tag
> git tag | xargs -I {} git push origin :refs/tags/{}
```