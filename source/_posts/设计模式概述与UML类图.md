---
layout: title
title: 设计模式概述与UML类图
date: 2019-09-16 11:10:02
categories: 设计模式
tags: C#设计模式（第2版）
---
思考并回答以下问题：
* 学会设计模式不一定能让你搭建好框架，但搭建好框架的人一定熟练设计模式。
* UML图中的1..\*是什么意思？实心菱形是什么意思？
* 聚合和组合有什么关系和区别？

<!--more-->

# <span style="color:#339AFF;">设计模式要点</span>

1.使用桥接模式时系统中的类必须存在两个独立变化的维度，在使用组合模式时系统中必须存在整体和部分的层次结构。

2.设计模式根据范围分类，即模式主要是处理类之间的关系还是处理对象之间的关系，模式可分为类模式和对象模式两种类型。

(1)类模式：此类模式处理类和子类之间的关系，这些关系通过继承建立，在编译时就被确定下来，是一种静态模式。

(2)对象模式：此类模式处理对象间的关系，这些关系在运行时变化，更具动态性。

# <span style="color:#339AFF;">设计模式简介</span>


# <span style="color:#339AFF;">类和类的UML表示</span>

在UML2.0的13种图形中，类图是使用最广泛的图形之一，它用于描述系统中所包含的类以及它们之间的相互关系，每一个设计模式的结构都可以使用类图来表示。类图帮助人们简化对系统的理解，是系统分析和设计阶段的重要产物，也是系统编码的重要模型依据。

<font size=3>**1.类**</font>

类（Class）封装了数据和行为，是面向对象的重要组成部分，它是具有相同属性、操作、关系的对象集合的总称。在系统中，每个类都具有一定的职责，职责指的是类要完成什么样的功能，要承担什么样的义务。一个类可以有多种职责，设计得好的类通常有且仅有一种职责。在定义类的时候，将类的职责分解成为类的属性和操作（即方法）。类的属性即类的数据职责，类的操作即类的行为职责。设计类是面向对象设计中最重要的组成部分，也是最复杂和最耗时的部分。 

在软件系统运行时，类将被实例化成对象（Object），对象对应于某个具体的事物，是类的实例（Instance）。

类图（Class Diagram）使用出现在系统中的不同类来描述系统的静态结构，它用来描述不同的类以及它们之间的关系。  

<font size=3>**2.类的UML图示**</font>  

在UML中，类使用包含类名、属性和操作且带有分隔线的长方形来表示，例如定义一个Employee类，它包含属性name、age和name和email，以及操作ModifyInfo()，在UML类图中该类如图1所示。

> 图1 类的UML图示

{% asset_img 1.png %}

图1对应的C#代码片段如下：
```cs
public class Employee 
{
    private string name;
    private int age;
    private string email;

    public void ModifyInfo()
    {
        ...
    }
}
```
在UML类图中，类一般由三部分组成：

(1)第一部分是类名，每个类都必须有一个名字，类名是一个字符串。

(2)第二部分是类的属性（Attributes），属性是指类的性质，即类的成员变量。一个类可以有任意多个属性，也可以没有属性。

UML规定属性的表示方式如下：
```txt
可见性 名称:类型[ = 默认值]
```
其中： 

①“可见性”表示该属性对于类外的元素而言是否可见，包括公有（public）、私有（private）和受保护（protected）3种，在类图中分别用符号“+"、“-”和“#”表示。在C#语言中还新增了internal和protected internal两种可见性，其中，internal表示程序集内可见，protected internal表示程序集内可见或者子类可见，分别用符号“i”和“r”表示。为了保证数据的封装性，属性的可见性通常为private，它们通过公有的Getter方法和Setter方法供外界使用。

②“名称”表示属性名，用一个字符串表示，按照C#语言的命名规范，属性命名采用驼峰命名法（Camel Case），即属性名中的第一个单词全小写，之后每个单词的首字母大写。

③“类型”表示属性的数据类型，可以是基本数据类型，也可以是用户自定义类型。

④“默认值”是一个可选项，即属性的初始值。

(3)第三部分是类的操作（Operations），操作是类的任意一个实例对象都拥有的行为，是类的成员方法。

UML规定操作的表示方式如下：
```txt
可见性 名称(参数列表) [ : 返回类型]
```
其中：
①“可见性”的定义与属性的可见性定义相同。  

②“名称”即方法名或操作名，用一个字符串表示，按照C#语言的命名规范，方法命名采用帕斯卡命名法（Pascal Case），即方法名中的每个单词首字母都大写。 

③“参数列表”表示方法的参数，其语法与属性的定义相似，参数个数是任意的，多个参数之间用逗号“,”隔开。

④“返回类型”是一个可选项，表示方法的返回值类型，依赖于具体的编程语言，可以是基本数据类型，也可以是用户自定义类型，还可以是空类型（void），如果是构造方法，则无返回类型。

# <span style="color:#339AFF;">类之间的关系</span>

在软件系统中，类并不是孤立存在的，类与类之间存在着各种关系，对于不同类型的关系， UML提供了不同的表示方式。  

<font size=3>**1.关联关系**</font>  

关联（Association）关系是类与类之间最常用的一种关系，它是一种结构化关系，用于表示一类对象与另一类对象之间有联系，如汽车和轮胎、师傅和徒弟、班级和学生等。在UML类图中，用实线连接有关联关系的对象所对应的类，在使用C、C++和Java等编程语1言实现关联关系时，通常将一个类的对象作为另一个类的成员变量。在使用类图表示关联关系时可以在关联线上标注角色名，一般使用一个表示两者之间关系的动词或者名词表示角色名（有时该名词为实例对象名），关系的两端代表两种不同的角色，因此在一个关联关系中可以包含两个角色名，角色名不是必需的，可以根据需要增加，其目的是使类之间的关系更加明确。  

例如在一个登录界面类LoginForm中包含一个Button类型的注册按钮loginButton，它们之间可以表示为关联关系，代码实现时可以在LoginForm中定义一个名为loginButton的属性对象，其类型为Button，如图2所示。

> 图2 关联关系实例

{% asset_img 2.png %}

图2对应的C#代码片段如下：

```cs
public class LoginForm
{
    private Button loginButton;
    ...
}

public class Button
{
    ...
}
```
在UML中，关联关系通常又包含以下几种形式。  

(1)双向关联：默认情况下，关联是双向的。例如顾客（Customer）购买商品（Product）并拥有商品，反之，卖出的商品总有某个顾客与之相关联。因此，Customer类和Product类之间具有双向关联关系，如图3所示。

> 图3 双向关联实例

{% asset_img 3.png %}

图3对应的C#代码片段如下：
```cs
public class Customer
{
    private Product[] products;
    ...
}

public class Product
{
    private Customer customer;
    ...
}
```
(2)单向关联：类的关联关系也可以是单向的，单向关联用带箭头的实线表示。例如顾客（Customer）拥有地址（Address），则Customer类与Address类具有单向关系，如图4所示。

> 图4 单向关联实例

{% asset_img 4.png %}

图4对应的C#代码片段如下：
```cs
public class Customer
{
    private Address address;
    ...
}

public class Address
{
    ...
}
```
(3)自关联：在系统中可能会存在一些类的属性对象类型为该类本身，这种特殊的关联关系称为自关联。例如，一个结点类（Node）的成员又是结点Node类型的对象，如图5所示。

> 图5 自关联实例

{% asset_img 5.png %}

图5对应的C#代码片段如下：
```cs
public class Node
{
    private Node subNode;
    ...
}
```

(4)多重性关联：多重性关联关系又称为重数性（Multiplicity）关联关系，表示两个关联对象在数量上的对应关系。在UML中，对象之间的多重性可以直接在关联直线上用一个数字或一个数字范围表示。  

对象之间可以存在多种多重性关联关系，常见的多重性表示方式如表1所示。

> 表1 多重性表示方式列表

| <center>**表示方式** </center>  | <center>**多重性说明** </center>  |
| :-| :- |
| 1..1  | 表示另一个类的一个对象只与该类的一个对象有关系  |
| 0..*  | 表示另一个类的一个对象与该类的零个或多个对象有关系  |
| 1..*  | 表示另一个类的一个对象与该类的一个或多个对象有关系  |
| 0..1  | 表示另一个类的一个对象没有或只与该类的一个对象有关系  |
| m..n  |  表示另一个类的一个对象与该类最少m个最多n个对象有关系 |

例如一个界面（Form）可以拥有零个或多个按钮（Button），但是一个按钮只能属于一个界面，因此，一个Form类的对象可以与零个或多个Button类的对象相关联，但一个Button类的对象只能与一个Form类的对象关联，如图6所示。

> 图6 多重性关联实例

{% asset_img 6.png %}

图5对应的C#代码片段如下：
```cs
public class Form
{
    private Button[] buttons; // 定义一个集合对象
    ...
}

public class Button
{
    ...
}
```

(5)聚合关系：聚合（Aggregation）关系表示整体与部分的关系。在聚合关系中，成员对象是整体对象的一部分，但是成员对象可以脱离整体对象独立存在。在UML中，聚合关系用带空心菱形的直线表示。例如汽车发动机（Engine）是汽车（Car）的组成部分，但是汽车发动机可以独立存在，因此，汽车和发动机是聚合关系，如图7所示。  

> 图7 聚合关系实例

{% asset_img 7.png %}

在用代码实现聚合关系时，成员对象通常作为构造方法、Setter方法或业务方法的参数注入整体对象中。图7对应的C#代码片段如下：

```cs
public class Car
{
    private Engine engine;

    // 构造注入
    public Car(Engine engine)
    {
        this.engine = engine;
    }

    public Engine Engine
    {
        get { return engine; }
        set { engine = value; } // 设值注入
    }
    ...
}

public class Engine
{
    ...
}
```

(6)组合关系：组合（Composition）关系也表示类之间整体和部分的关系，但是在组合关系中整体对象可以控制成员对象的生命周期，一旦整体对象不存在，成员对象也将不存在，成员对象与整体对象之间具有同生共死的关系。在UML中，组合关系用带实心菱形的直线表示。例如人的头（Head）与嘴巴（Mouth），嘴巴是头的组成部分之一，如果头没了，嘴巴也就没了，因此头和嘴巴是组合关系，如图8所示。

> 图8 组合关系实例

{% asset_img 8.png %}

在用代码实现组合关系时，通常在整体类的构造方法中直接实例化成员类。图8对应的C#代码片段如下：
```cs
public class Head
{
    private Mouth mouth;

    public Head()
    {
        mouth = new Mouth(); // 实例化成员类
    }
    ...
}

public class Mouth
{
    ...
}
```

<font size=3>**2.依赖关系**</font>

依赖（Dependency）关系是一种使用关系，特定事物的改变有可能会影响到使用该事物的其他事物，在需要表示一个事物使用另一个事物时使用依赖关系。大多数情况下，依赖关系体现在某个类的方法使用另一个类的对象作为参数。在UML中，依赖关系用带箭头的  虚线表示，由依赖的一方指向被依赖的一方。例如驾驶员开车，在Driver类的Drive()方法中将Car类型的对象car作为一个参数传递，以便在Drive()方法中能够调用car的Move()方法，驾驶员的Drive()方法依赖车的Move()方法，因此类Driver依赖类Car，如图9所示。

> 图9 依赖关系实例

{% asset_img 9.png %}

在系统实施阶段，依赖关系通常通过3种方式来实现，第一种也是最常用的一种方式，如图9所示，将一个类的对象作为另一个类中方法的参数，第二种方式是在一个类的方法中将另一个类的对象作为其局部变量，第三种方式是在一个类的方法中调用另一个类的静态方法。图9对应的C#代码片段如下：
```cs
public class Driver
{
    public void Drive(Car car)
    {
        car. Move();   
    }
    ...
}

public class Car
{
    public void Move()
    {
        ...
    }
    ...
}
```

<font size=3>**3.泛化关系**</font>

泛化（Generaliation）关系也就是继承关系，用于描述父类与子类之间的关系，父类又称为基类或超类，子类又称为派生类。在UML中，泛化关系用带空心三角形的直线来表示。在用代码实现时，使用面向对象的继承机制来实现泛化关系，在C#中使用冒号“：”来实现。例如Student类和Teacher类都是Person类的子类，Student类和Teacher类继承了Person类的属性和方法，Person类的属性包含姓名（name）和年龄（age），每一个Student和Teaeher也都具有这两个属性，另外，Student类增加了属性学号（studentNo），Teacher类增加了属性教师编号（teacherNo），Person类的方法包括行走Move()和说话Say()，Studen类和Teaeher类继承了这两个方法，而且Student类还新增了方法Study()，Teaeher类还新增了方法Teach()，如图10所示。

> 图10 泛化关系实例

{% asset_img 10.png %}

图10对应的C#代码片段如下：
```cs
// 父类
public class Person
{
    protected string name;
    protected int age;

    public void Move()
    {
        ...
    }

    public void Say()
    {
        ...
    }
}

// 子类
public class Student : Person
{
    private string studentNo;

    public void Study()
    {
        ...
    }
}

// 子类
public class Teacher : Person
{
    private string teacherNo;

    public void Teach()
    {
        ...
    }
}
```

<font size=3>**4.接口与实现关系**</font>  

在很多面向对象语言中都引入了接口的概念，例如C#、Java等，在接口中通常没有属性，而且所有的操作都是抽象的，只有操作的声明，没有操作的实现。在C#中，接口中的方法默认可见性均为public，无须再使用任何可见性关键字。在UML中用与类的表示法类似的方式表示接口，如图11所示。  

> 图11 接口的UML图示

{% asset_img 11.png %}

接口之间也可以有与类之间关系类似的继承关系和依赖关系，但是接口和类之间还存在一种实现（Realization）关系，在这种关系中，类实现了接口，类中的操作实现了接口中所声明的操作。在UML中，类与接口之间的实现关系用带空心三角形的虚线来表示。例如定义了一个交通工具接口Vehicle，包含一个抽象操作Move()，在类Ship和类Car中都实现了该Move()操作，不过具体的实现细节将会不一样，如图12所示。

> 图12 实现关系实例

{% asset_img 12.png %}

实现关系在用代码实现时，不同的面向对象语言也提供了不同的语法，在C#中使用冒号“：”来实现。图12对应的C#代码片段如下：
```cs
public interface Vehicle
{
    void Move();
}

public class Ship : Vehicle
{
    public void Move()
    {
        ...
    }
}

public class Car : Vehicle
{
    public void Move
    {
        ...
    }
}
```