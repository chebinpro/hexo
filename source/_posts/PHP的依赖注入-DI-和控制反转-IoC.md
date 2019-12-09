---
layout: title
title: PHP的依赖注入(DI)和控制反转(IoC)
date: 2018-07-17 15:01:10
categories: PHP
tags: 依赖注入
top: true
---
思考并回答以下问题：
* 这两种模式的目的是什么？
* 整个过程中参与者都有谁？
* 依赖：谁依赖于谁？为什么会产生依赖？
* 注入：谁注入于谁？到底注入了什么？
* 控制反转：谁控制谁？控制什么？为何叫反转（有反转就应该有正转了，正转是什么呢？）
* 依赖注入和控制反转是同一概念吗？
* A类不再主动去获取C，而是被动等待，等待IoC/DI的容器获取一个C的实例，然后反向的注入到A类中。怎么理解？
* 应用程序原本是老大，要获取什么资源都是主动出击，但是在IoC/DI思想中，应用程序就变成被动的了，被动的等待IoC/DI容器来创建并注入它所需要的资源了。怎么理解？

<!--more-->

# <span style="color:#039BE5;">简介</span>

* IoC - <span style="color:red">Inversion of Control  控制反转</span>
* DI  - <span style="color:red">Dependency Injection  依赖注入</span>

依赖注入和控制反转说的实际上是同一个东西，它们是一种设计模式，这种设计模式用来<span style="color:red">减少程序间的耦合</span>。

<font size=3>**优势（为什么使用）**</font>

使用依赖注入，最重要的一点好处就是有效的<span style="color:red">分离了对象和它所需要的外部资源</span>，使得它们松散耦合，有利于功能复用，更重要的是使得程序的整个体系结构变得非常灵活。

<font size=3>**概念**</font>

<span style="color:red">依赖注入和控制反转是对同一件事情的不同描述</span>，从某个方面讲，就是它们<span style="color:red">描述的角度不同</span>。

依赖注入是从应用程序的角度在描述，可以把依赖注入，即：应用程序依赖容器创建并注入它所需要的外部资源；

而控制反转是从容器的角度在描述，即：容器控制应用程序，由容器反向的向应用程序注入应用程序所需要的外部资源。

# <span style="color:#039BE5;">问答</span>

对于一个菜鸟，如果你看了上面的概念还是一头雾水的话，那么恭喜你，你和我一样不是天才，那么下面就让我们借助于几个问答来搞清楚这个概念的意思吧。

* 整个过程中参与者都有谁？

    * 一般有<span style="color:red">三方参与者</span>，一个是某个对象；一个是IoC/DI的容器；另一个是某个对象的外部资源。
    * 某个对象指的就是任意的、普通的PHP对象。
    * IoC/DI的容器简单点说就是指用来实现IoC/DI功能的一个框架程序。
    * 对象的外部资源指的就是对象需要的，但是是从对象外部获取的，都统称资源，比如：对象需要的其它对象、或者是对象需要的文件资源等等。


* 谁依赖于谁：

    * 当然是某个对象依赖于IoC/DI的容器。


* 为什么需要依赖：

    * 对象需要IoC/DI的容器来提供对象需要的外部资源。


* 谁注入于谁：

    * 是IoC/DI的容器注入某个对象。


* 到底注入什么：

    * 就是注入某个对象所需要的外部资源。


* 谁控制谁：

    * 当然是IoC/DI的容器来控制对象了。


* 控制什么：

    * 主要是<span style="color:red">控制对象实例的创建</span>。


* 为何叫反转：

    * 反转是相对于正向而言的，那么什么算是正向的呢？
    * 考虑一下常规情况下的应用程序，如果要在A里面使用C，你会怎么做呢？当然是直接去创建C的对象，也就是说，是在A类中主动去获取所需要的外部资源C（$c = new C();），这种情况被称为正向的。那么什么是反向呢？就是<span style="color:red">A类不再主动去获取C，而是被动等待，等待IoC/DI的容器获取一个C的实例，然后反向的注入到A类中</span>。

用图例来说明一下，先看没有IoC/DI的时候，常规的A类使用C类的示意图，如下图所示：

{% asset_img 1.png %}

代码示意：
```php
<?php
/**
 * 没有IoC/DI的时候，常规的A类使用C类的示例
 */

/**
 * Class c
 */
class C
{
    public function say()
    {
        echo 'hello';
    }
}

/**
 * Class A
 */
class A
{
    private $c;
    public function __construct()
    {
        $this->c = new C(); // 实例化创建C类
    }

    public function sayC()
    {
        echo $this->c->say(); // 调用C类中的方法
    }
}

$a = new A();
$a->sayC();
```
当有了IoC/DI的容器后，A类不再主动去创建C了，如下图所示：

{% asset_img 2.png %}

而是被动等待，<span style="color:red">等待IoC/DI的容器获取一个C的实例，然后反向的注入到A类中</span>，如下图所示：

{% asset_img 3.png %}

代码示意：
```php
<?php
/**
 * 当有了IoC/DI的容器后，A类依赖C类实例注入的示例
 */

/**
 * Class C
 */
class C
{
    public function say()
    {
        echo 'hello';
    }
}

/**
 * Class A
 */
class A
{
    private $c;

    public function setC(C $c)
    {
        $this->c = $c; // 实例化创建C类
    }

    public function sayC()
    {
        echo $this->c->say(); // 调用C类中的方法
    }
}

$c = new C();
$a = new A();
$a->setC($c); // 先set，$c作为C类的实例不是在A类里创建的。
$a->sayC();
```
* 什么是正转？

    * 正转就是按照普通的，在类中直接创建对象实例，如$c = new C();


* 依赖注入和控制反转是同一概念吗？

    * 根据上面的讲述，我们不难出来，“依赖注入”和“控制反转”确实是对同一件事情的不同描述，从某个方面讲，就是它们**描述的角度不同**。
  

# <span style="color:#039BE5;">总结</span>

其实IoC/DI对编程带来的最大改变不是从代码上，而是从思想上，发生了“主从换位”的变化。<span style="color:red">应用程序原本是老大，要获取什么资源都是主动出击，但是在IoC/DI思想中，应用程序就变成被动的了，被动的等待IoC/DI容器来创建并注入它所需要的资源了。</span>

> 注意
我们上面说了，这是一种“设计模式”，就像“工厂模式”和“单例模式”等是一样的，它是一种面向对象中的编程“思想”，自然它也不仅限于PHP，而是所有面向对象的语言基本都是可以适用的。