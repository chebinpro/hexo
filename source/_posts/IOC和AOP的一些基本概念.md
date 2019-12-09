---
layout: title
title: IOC和AOP的一些基本概念
date: 2019-09-11 16:22:26
tags:
---
思考并回答以下问题：

<!--more-->

# 什么是IOC

IoC就是Inversion of Control，控制反转。在Java开发中，IoC意味着将你设计好的类交给系统去控制，而不是在你的类内部控制。这称为控制反转。

下面我们以几个例子来说明什么是IoC

假设我们要设计一个Girl和一个Boy类，其中Girl有kiss方法，即Girl想要Kiss一个Boy。那么，我们的问题是，Girl如何能够认识这个Boy？

在我们中国，常见的ＭＭ与GG的认识方式有以下几种

１　青梅竹马； ２　亲友介绍； ３　父母包办

那么哪一种才是最好呢？

青梅竹马：Girl从小就知道自己的Boy。
```cs
public class Girl 
{  
    void kiss()
    {
        Boy boy = new Boy();
    }
}
```
然而从开始就创建的Boy缺点就是无法在更换。并且要负责Boy的整个生命周期。如果我们的Girl想要换一个怎么办？（笔者严重不支持Girl经常更换Boy）
亲友介绍：由中间人负责提供Boy来见面
```cs
public class Girl 
{
    void kiss()
    {
        Boy boy = BoyFactory.createBoy();       
    }
}
```
亲友介绍，固然是好。如果不满意，尽管另外换一个好了。但是，亲友BoyFactory经常是以Singleton的形式出现，不然就是，存在于Globals，无处不在，无处不能。实在是太繁琐了一点，不够灵活。我为什么一定要这个亲友掺和进来呢？为什么一定要付给她介绍费呢？万一最好的朋友爱上了我的男朋友呢？

父母包办：一切交给父母，自己不用费吹灰之力，只需要等着Kiss就好了。
```cs
public class Girl 
{
    void kiss(Boy boy)
    {
        // kiss boy  
        boy.kiss();
    }
}
```
Well，这是对Girl最好的方法，只要想办法贿赂了Girl的父母，并把Boy交给他。那么我们就可以轻松的和Girl来Kiss了。看来几千年传统的父母之命还真是有用哦。至少Boy和Girl不用自己瞎忙乎了。

这就是IOC，将对象的创建和获取提取到外部。由外部容器提供需要的组件。

我们知道好莱坞原则：“Do not call us, we will call you.” 意思就是，You, girlie, do not call the boy. We will feed you a boy。


我们还应该知道依赖倒转原则即 Dependence Inversion Princinple，DIP。

Eric Gamma说，要面向抽象编程。面向接口编程是面向对象的核心。

组件应该分为两部分，即

Service, 所提供功能的声明

Implementation, Service的实现

好处是：多实现可以任意切换，防止 “everything depends on everything” 问题．即具体依赖于具体。

所以，我们的Boy应该是实现Kissable接口。这样一旦Girl不想kiss可恶的Boy的话，还可以kiss可爱的kitten和慈祥的grandmother。

# IOC的type

IoC的Type指的是Girl得到Boy的几种不同方式。我们逐一来说明。

IOC type 0：不用IOC

```cs
public class Girl implements Servicable 
{

    private Kissable kissable;

    public Girl() 
    {

        kissable = new Boy();
    }

    public void kissYourKissable() 
    {
        kissable.kiss();
    }
}
```
Girl自己建立自己的Boy，很难更换，很难共享给别人，只能单独使用，并负责完全的生命周期。

IOC type 1，先看代码：

```cs
public class Girl implements Servicable 
{
    Kissable kissable;

    public void service(ServiceManager mgr) 
    {
        kissable = (Kissable) mgr.lookup(“kissable”);
    }

    public void kissYourKissable() 
    {
        kissable.kiss();
    }
}
```
这种情况出现于Avalon Framework。一个组件实现了Servicable接口，就必须实现service方法，并传入一个ServiceManager。其中会含有需要的其它组件。只需要在service方法中初始化需要的Boy。

另外，J2EE中从Context取得对象也属于type 1。

它依赖于配置文件
```xml
<Container>
      <component name=“kissable“ class=“Boy">               
         <configuration> … </configuration>
      </component>
      <component name=“girl" class=“Girl" />
</container>
```
      IOC type 2：
```cs
public class Girl 
{

    private Kissable kissable;

    public void setKissable(Kissable kissable) 
    {
      this.kissable = kissable;
    }

    public void kissYourKissable() 
    {
      kissable.kiss();
    }
}
```
Type 2出现于spring Framework，是通过JavaBean的set方法来将需要的Boy传递给Girl。它必须依赖于配置文件。
```xml
<beans>
      <bean id=“boy" class=“Boy"/>
      <bean id=“girl“ class=“Girl">
          <property name=“kissable">
             <ref bean=“boy"/>
          </property>
      </bean>
</beans>
```
IOC type 3
```cs
public class Girl 
{
    private Kissable kissable;

    public Girl(Kissable kissable) 
    {
        this.kissable = kissable;
    }

    public void kissYourKissable()
    {
        kissable.kiss();
    }
}
```
      这就是PicoContainer的组件 。通过构造函数传递Boy给Girl。
```cs
PicoContainer container = new DefaultPicoContainer();

container.registerComponentImplementation(Boy.class);

container.registerComponentImplementation(Girl.class);

Girl girl = (Girl) container.getComponentInstance(Girl.class);

girl.kissYourKissable();
```

 

================
AOP 关注与主要的东西，也可以说让你只关注与业务，其他的东西就让AOP帮你完成。

我们要吃武昌鱼：
```cs
public class Dinner 
{
    Customer yangyi;

    public void eatfish() 
    {
        yangyi.cookFish();
        yangyi.eatFish();
        yangyi.washDish();
    }
}
```
prog 1

现在我觉得这顿饭吃的太不爽了，还要自己做,洗碗。


我想这样
```cs
public class Dinner 
{
    ICustomer yangyi;

    public void eatfish() 
    {
        yangyi.eatFish();
    }
}
```
prog 2

就是说我现在只用吃就行了，但是这样可以么？
谁来帮我做鱼呢？这个我想就是所谓AOP吧，我只用关心我吃鱼的方面就行了，做鱼又食堂关注就行了。

现在看看谁来帮我cook fish呢，它怎么做到的呢？

我看到上面prog 1中的Customer已经变成了prog 2中的ICustomer了，就是说如果愿意，我可以用任意的ICustomer实现来替代yangyi了，这个就是一个Customer的代理，在调用 yangyi.eatfish的时候，之前代理会帮我调用cookfish()，之后代理会帮我调用washdish()
还是看看代码吧
```cs
class Customer implements ICustomer 
{
    void eatfish() 
    {
        // haha eat, delicious
    }
}

class CustomerProxy implements ICustomer 
{
    Customer yangyi = new Customer() //just for example. proxy is implement by dynamic proxy or other tech in real world
    void eatfish() 
    {
        cookfish();
        yangyi.eatfish();
        washdish();
    }
}
```
基本上就是说proxy会在调用我的Customer代码之前和之后做一些工作，ok, 这就是AOP，不知道我讲清楚了没有。在spring中，proxy是用java的动态代理做的，在运行的时候自动生成CustomerProxy这个东西，然后用这个Proxy替代我的Customer。

为什么老把IOC和AOP放在一起讲呢，想想我们的Customer是怎么来的吧

如果在某个时候我们说yangyi = new Customer()，那这样Proxy怎么替代这个Customer啊。
所以IoC说，你要用Customer么，你直接说我要用，声明他，告诉我怎么样能够设置这个Customer，用的时候我会来调用你的(don't call me, but I call you)

所以真正的代码应该是这样的
```cs
public class Dinner 
{
    ICustomer yangyi;

    public void eatfish() 
    {
        yangyi.eatFish();
    }

    public void setYangyi(ICustomer customer) 
    {
        this.yangyi = customer;
    }
}
```
我提供一个SetYangyi的方法，IoC会在运行时把yangyi设置成CustomerProxy，然后，我的代码就只用尽情的享受武昌鱼的美味了。其它的，留给ioc了。

BTW,烦死JDBC的connection.close()了吧，呵呵，IOC＋AOP应该最适合这方面了。

 

 

 

这两个概念基本上是一个设计层的概念，主要讲的就是怎么去分离关注，用面向对象的话说，就是怎么把职责进行分离。而这两个模式，我个人认为都有一个共同点，就是变以前的主动为被动，而我认为，这种改变可能也是将来面向对象发展的一个趋势。

首先说说什么叫主动。写过面向对象程序的人都知道，面向对象与面向过程的区别就是，面向对象是由一大堆对象组成的，对象通过协作完成面向过程中的任务。假设现在有对象A和B，那么当A需要使用B中的方法时，那么在A内部，就会有有一个对B方法的调用，这种调用就称为主动调用。代码大概会如下：
```cs
public class A
{
    B b;

    public void methodA()
    {
        b.methodB();
    }
}
```
这里为了下文解释方便，我增加了一个调用点的定义，调用点就是调用发生的地方。也就是上面

b.methodB()中的b。

理解了什么叫主动之后，我想就先介绍什么叫IoC。IoC的全称这里就不说了，他的字面意思就是控制反转。在上面的代码当中，由于A调用了B的方法，因此就形成了一个A对B的依赖，这本身并没有什么问题。但是OO的思想是希望我们基于接口编程，而不是基于实现编程。因此，系统设计将不止是原有的A，B，而需要变成IA，IB，A，B，其中IA，IB是接口，A，B是对应的实现类，然后为了使得A中现在对B的实现依赖变成对接口的依赖，代码应该变成这样。
```cs
public class A implements IA
{
    IB b;

    public void methodA()
    {
        b.methodB();
    }
}
```
这里虽然我们是基于接口编程了，但大家知道，在这中间，我们需要有一个步骤把b指向一个IB的实现类B，这个怎么做，就是IoC要做的事情，这里就不细说了。但简单来说，没有IoC，我们可能需要在A中通过某种方法去获取一个B的实例，但有了IoC，她就能在A不参与的情况下，给我们一个B的实例，所以，IoC要做的就是在调用点上从原来的主动生成一个调用点，变成被动的接受一个调用点。

接着就是AOP，全称也不说了，字面意思就是面向方面编程。举一个最普遍的例子，就是如果我们代码需要做日志的话，那么在没有AOP的时候，我们的代码可能就是这样：
```cs
public class A
{
    public void methodA()
    {
        do log;
        b.methodB();
    }
}
```
这里methodA()中的做日志并不是方法本身的逻辑功能，而是一个附属功能，因此，我们需要把它分离出去。怎么分离，就是AOP要做的事情，简单来说，就是系统在调用者不知情的情况下，为我们的类A增加了一个代理类，她把我们的类A包装了起来，像这样：
```cs
public class AP extends A
{
    A a;

    public void methodA()
    {
        do log;
        a.methodA():
    }
}

public class A
{

    public void methodA()
    {
        b.methodB();
    }
}
```
于是，当我们以为自己在调用A的methodA()方法时，实际调用的将是AP中的methodA()，于是就可以把做日志的功能从原有的methodA()中分离了出去。所以，AOP要做的就是在用户不知道的情况下，将我们的调用点包裹了起来，从而把原来的功能进行了分离。