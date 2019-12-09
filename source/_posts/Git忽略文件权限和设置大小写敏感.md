---
layout: title
title: Git忽略文件权限和设置大小写敏感
date: 2019-06-05 11:58:07
categories: 工具
tags: Git
---
在发布项目到线上时，很多时候需要修改文件的权限，如果是使用git来发布的话，那么下次更新线上文件的时候就会提示文件冲突。明明文件没有修改，为什么会冲突呢？原来git把文件权限也算作文件差异的一部分。下面笔者自己做了个简单的例子来演示这种情况。

<!--more-->

# 忽略文件权限

修改版本库的文件的权限，然后使用diff查看下改变。
```bash
$ chmod 777 pack.php
$ git diff pack.php
```

可以看到git把文件权限也列入了版本管理。


git中可以加入忽略文件权限的配置，具体如下：
```bash
$ git config core.filemode false
```
这样就设置了忽略文件权限。查看下配置：
```bash
$ cat .git/config
```
git忽略文件权限的配置
这时候再更新代码就OK了。

文件所有者和所有组的修改不列入版本管理。

# 对文件名大小写敏感

当你创建一个文件后,叫 readme.md 写入内容后 提交到线上代码仓库.
然后你在本地修改文件名为 Readme.md 接着你去提交,发现代码没有变化.
控制台输入git status 也不显示任何信息

那么就配置git 使其对文件名大小写敏感
git config core.ignorecase false