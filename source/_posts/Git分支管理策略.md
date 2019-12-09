---
layout: title
title: Git分支管理策略
date: 2019-06-06 17:47:44
categories: 工具
tags: Git
---
通常，合并分支时，如果可能，Git会用Fast forward模式，但这种模式下，删除分支后，会丢掉分支信息。

<!--more-->

这句话的说明如下：此时master分支没有做任何代码的改动。切换到master分支合并有代码改动的dev，此时默认使用Fast forward模式，git log不会有合并的信息。
```bash
$ git checkout  -b cb_dev  （此时在master分支）
$ vim 1.txt  （此时在cb_dev分支）
$ git commit -am 'test'
$ git checkout master
$ git merge cb_dev
$ git log
 * e429856 test
* 121be0f 更新 xiaoxiaole.json
* 649f161 更新 README.md
* 3d02ba4 更新消消乐web版为手机预览版本
```

{% asset_img 1.png %}
如果再把分支删了，则分支信息一点都没有了

而如果master也有改动，则是会有分支的信息的。


如果要强制禁用Fast forward模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。

下面我们实战一下--no-ff方式的git merge：
首先，仍然创建并切换dev分支：
$ git checkout -b dev
Switched to a new branch 'dev'

修改readme.txt文件，并提交一个新的commit：
$ git add readme.txt 
$ git commit -m "add merge"
[dev f52c633] add merge
 1 file changed, 1 insertion(+)

现在，我们切换回master：
$ git checkout master
Switched to branch 'master'

准备合并dev分支，请注意--no-ff参数，表示禁用Fast forward：
$ git merge --no-ff -m "merge with no-ff" dev
Merge made by the 'recursive' strategy.
 readme.txt | 1 +
 1 file changed, 1 insertion(+)

因为本次合并要创建一个新的commit，所以加上-m参数，把commit描述写进去。
合并后，我们用git log看看分支历史：
$ git log --graph（图表） --pretty（pretty）=oneline --abbrev（缩写）-commit
*   e1e9c68 merge with no-ff
|\  
| * f52c633 add merge
|/  
*   cf810e4 conflict fixed
...

可以看到，不使用Fast forward模式，merge后就像这样：

{% asset_img 2.png %}

---------------------------------------------------------------------------------------------------------------------
在实际开发中，我们应该按照几个基本原则进行分支管理：

首先，master分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；

那在哪干活呢？干活都在dev分支上，也就是说，dev分支是不稳定的，到某个时候，比如1.0版本发布时，再把dev分支合并到master上，在master分支发布1.0版本；

你和你的小伙伴们每个人都在dev分支上干活，每个人都有自己的分支，时不时地往dev分支上合并就可以了。
所以，团队合作的分支看起来就像这样：


{% asset_img 3.png %}