---
layout: title
title: Git标签管理
date: 2019-06-06 17:35:09
categories: 工具
tags: Git
---
发布一个版本时，我们通常先在版本库中打一个标签（tag），这样，就唯一确定了打标签时刻的版本。将来无论什么时候，取某个标签的版本，就是把那个打标签的时刻的历史版本取出来。所以，标签也是版本库的一个快照。

<!--more-->

> ** 快照 ** <br/>关于指定数据集合的一个完全可用拷贝，该拷贝包括相应数据在某个时间点（拷贝开始的时间点）的映像。快照可以是其所表示的数据的一个副本，也可以是数据的一个复制品。<br/>
快照的作用主要是能够进行在线数据备份与恢复。<br/>
1、当存储设备发生应用故障或者文件损坏时可以进行快速的数据恢复，将数据恢复某个可用的时间点的状态。<br/>
2、快照的另一个作用是为存储用户提供了另外一个数据访问通道，当原数据进行在线应用处理时，用户可以访问快照数据，还可以利用快照进行测试等工作。所有存储系统，不论高中低端，只要应用于在线系统，那么快照就成为一个不可或缺的功能。<br/>
好比玩RPG游戏时，每通过一关就会自动把游戏状态存盘，如果某一关没过去，你还可以选择读取前一关的状态。有些时候，在打Boss之前，你会手动存盘，以便万一打Boss失败了，可以从最近的地方重新开始。那个存盘就是快照，存储了那个时间点的状态。<br/>



标签可以针对某一时间点的版本做标记，常用于版本发布。
主要用来发布版本的。

标签就是跟某个commit关联起来，便于发布和查找。

步骤
1、切换到某个分支上。
```bash
$ git branch
* dev
  master
$ git checkout master
Switched to branch 'master'
```

2、然后，敲命令git tag <name>就可以打一个新标签：
打在了最近的那一次commit上
```bash
$ git tag v1.0
```
添加标签说明
```bash
$ git tag -a v0.1 -m "version 0.1 released" 
```

如果想打之前的commit，找历史然后再打就行了
```bash
$ git log --pretty=oneline --abbrev-commit
$ git tag v0.9 6224937
```

3、推送到远程
```bash
git push origin <tagname>
```
命令git push origin --tags可以推送全部未推送过的本地标签；

4、切换到标签
与切换分支命令相同，用git checkout [tagname]

5、删除标签
先从本地删除：
```bash
$ git tag -d v0.9
```
然后，从远程删除。删除命令也是push，但是格式如下：
```bash
$ git push origin :refs/tags/v0.9
```

6、查看标签
查看所有标签：
```bash
$ git tag
git show <tagname>查看标签信息
$ git tag -l 'v1.4.2.*'
```

没有不同分支下tag这个概念，也就是说tag不是属于某个分支的，而是全局的tag,是对于commit编号的一个别称。

标签是为了打上版本号信息，不能乱叫，通常用：
v1.0, v1.1, v2.0 ...
或者按发布日期：
build-20150702, build-20150910 ...

一般来说都是在master分支commit后打标签。

# TortoiseGit

{% asset_img 1.png %}