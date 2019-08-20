---
title: git 打标签的注意点
date: 2019-08-20 12:32:36
tags: "git"
categories: "技术"
---
### 问题
   tag的名字不要和分支一样... 我遇到的情况就是 不能正常merge,提示`refname 'xxx' is ambiguous.`和`branch is up to date with xxx`， 
### 如何排查
   `git show-ref` 查看命名情况,找到模糊定义的命名 删除或者修改即可 
