---
title: Git基础
date: 2021-10-04 22:48:38.402
updated: 2021-10-08 13:18:37.221
url: /archives/gitbasic
categories: Git
tags: Git | 基础
---

# 1. Git介绍
---
分布式版本控制
版本控制:记录文件内容变化,记录文件修改内容

版本控制实在本地进行的
不再存在代码服务器的单点故障问题
--- 
# 2. Git命令

命令名称 | 作用
---|---
git config --global user.name 用户名 | 设置用户签名
git config --global user.email 邮箱 | 设置用户签名
git init | 初始化本地库
git status| 查看本地库状态
vim 文件名.文件类型 | 新增文件(在vim中 yy复制 p粘贴)
git add | 将文件添加到添加到暂存区
git rm --chched 文件名..文件类型| 删除暂存区的文件
git commit -m "日志信息" 文件名| 提交到远程库
git reflog |查看引用信息(查看版本号)
git log |查看详细信息 (提交者+版本号)
vim 文件名.文件类型 | 修改文件内容
git reset --hard 版本号| 切换到指定版本号的版本
git branch 分支名 | 创建分支
git branch -v |查看分支
git checkout 分支名 |切换通知
git merge 分支名 |把指定的分支到合并当前分支上
git remote add 别名 远程库地址 |创建远程库别名
git push 别名 分支 | 将本地库指定分支代码推送到远程库
git pull 别名 分支 | 拉取远程库指定分支的代码到本地
git clone 远程地址 | 将指定的远程库代码克隆到本地库