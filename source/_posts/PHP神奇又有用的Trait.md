---
layout: title
title: PHP神奇又有用的Trait
date: 2018-12-23 10:11:58
tags:
---
PHP和JAVA，C++一样都是单继承模式。但是像Python，是支持多继承（即Mixin模式）。那么如何在PHP中实现多继承模式？这就需要使用trait。

<!--more-->

# <span style="color:#339AFF;">Trait使用方式</span>

```php
trait Arrayabletrait
{
    public function toArray()
    {
    }
}
class Model
{
    use Arrayabletrait;
}

$model = new Model();
$model->toArray();
```

# <span style="color:#339AFF;">Trait使用场景</span>

1、有些功能不需要类的方法属性，但是在不同的类都有使用需求。例如上面的对象转数组方法。

这种情况可以使用一个基类定义toArray方法，则需要将这类基础方法定义在尽可能顶层的基类当中，保证所有的类都能够调用这个方法。

2、类因为某些需求，已经继承了第三方类对象。例如第三方ORM模型类。这种情况如果要给类附加一些公共的功能，除了创建一个继承于ORM模型的基类，复制一套公共功能的代码之外，就可以使用trait。

# <span style="color:#339AFF;">trait使用注意</span>

**方法优先级**

```php
trait Arrayabletrait
{
    public function logname()
    {
        return 'trait:'.$this->name;
    }

    public static function staticlog()
    {
        return 'trait:'.self::$staticname;
    }
}

class Obj
{
    protected $name = 'obj';
    public static $staticname = 'Obj';

    public function logname()
    {
        return 'obj:'.$this->name;
    }
}

class Model extends Obj
{
    protected $name = 'model';
    public static $staticname = 'model';

    use Arrayabletrait;

    public function logname()
    {
        return 'model:'.$this->name;
    }

    public static function staticlog()
    {
        return 'model:'.self::$staticname;
    }
}

class Model2 extends Obj
{
    protected $name = 'model2';
    public static $staticname = 'Model2';
    use Arrayabletrait;
}

$model = new Model();
$model2 = new Model2();

echo $model->logname()."\n";
echo $model2->logname()."\n";
```
上面输出内容分别为model:model，trait:model2，model:model，trait:model2。可以看出，trait方法优先级为 **当前对象>trait>父类**，以上规则同样使用于静态调用。

**属性定义要特别小心**！！trait中可以定义属性。但是不能和use trait当前类定义的属性相同，否则会报错:define the same property。但是，如果父类使用了trait，子类定义trait中存在的属性，则没有问题。
```php
trait Arrayabletrait
{
    public $logger = 'file';
    public function log()
    {
        return 'trait:'.$this->logger.$this->name;
    }
}

class Obj
{
    use Arrayabletrait;
    protected $name = 'Obj';
}

class Model extends Obj
{
    protected $logger = 'redis';
}

$model = new Model();
echo $model->log()."\n";
```

**私有属性私有方法**。triat中可以方位use类的私有属性私有方法！！

从以上可以看出，trait本身是对类的一个扩展，在trait中使用$this，self，static，parent都与当前类一样，zend底层将trait代码嵌入到类当中，相当于底层帮我们实现了代码复制功能。
```php
trait Arrayabletrait1
{
    public function log()
    {
        return 'trait1:'.$this->logger.$this->name;
    }

    public function logname()
    {
        return 'trait1:'.$this->name;
    }
}

trait Arrayabletrait2
{
    public function log()
    {
        return 'trait2:'.$this->logger.$this->name;
    }

    public function logname()
    {
        return 'trait1:'.$this->name;
    }
}

class Model
{
    public $name = 'model';
    use Arrayabletrait1,Arrayabletrait2
    {
        Arrayabletrait1::log insteadof Arrayabletrait2;
        Arrayabletrait2::logname insteadof Arrayabletrait1;
        Arrayabletrait2::logname as logname1;
    }
    protected $logger = 'redis';
}

$model = new Model();
echo $model->log()."\n";
echo $model->logname1()."\n";
```

**多个trait相同方法。**

多trait相同的方法，需要使用instanceof指定使用哪个trait的方法。instanceof后面的使用的trait。可以使用as设置添加方法别名（添加，原有方法还是能调用！！）。as还可以改变方法的访问控制

Arrayabletrait2::logname as private改为私有方法。