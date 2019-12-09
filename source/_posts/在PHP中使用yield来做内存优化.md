---
layout: title
title: 在PHP中使用yield来做内存优化
date: 2018-12-05 10:11:22
tags: PHP
---
* 什么是yield
* yield&return的区别
* yield有什么选项
* 结论

<!--more-->

# <span style="color:#339AFF;">什么是yield</span>

```txt
生成器函数看上去就像一个普通函数，除了不是返回一个值之外， 生成器会根据需求产生更多的值。
```

来看以下的例子：
```php
function getValues() {
    yield 'value';
}

// 输出字符串 "value"
echo getValues();
```
当然，这不是他生效的方式，前面的例子会给你一个致命的错误： 类生成器的对象不能被转换成字符串， 让我们清楚的说明：

# yield&return的区别

前面的错误意味着getValues()方法不会如预期返回一个字符串，让我们检查一下他的类型：
```php
function getValues() {
    return 'value';
}
var_dump(getValues()); // string(5) "value"

function getValues() {
    yield 'value';
}
var_dump(getValues()); // class Generator#1 (0) {}
```
生成器类实现了迭代器接口，这意味着你必须遍历getValue()方法来取值：
```php
foreach (getValues() as $value) {
   echo $value;
}

// 使用变量也是好的
$values = getValues();
foreach ($values as $value) {
   echo $value;
}
```
但这不是唯一的不同！

一个生成器允许你使用循环来迭代一组数据，而不需要在内存中创建是一个数组，这可能会导致你超出内存限制。

在下面的例子里我们创建一个有800,000元素的数字同时从getValues() 方法中返回他，同时在此期间，我们将使用函数 memory_get_usage() 来获取分配给次脚本的内存， 我们将会每增加 200,000 个元素来获取一下内存使用量，这意味着我们将会提出四个检查点：
```php
<?php
function getValues() {
   $valuesArray = [];
   // 获取初始内存使用量
   echo round(memory_get_usage() / 1024 / 1024, 2) . ' MB' . PHP_EOL;
   for ($i = 1; $i < 800000; $i++) {
      $valuesArray[] = $i;
      // 为了让我们能进行分析，所以我们测量一下内存使用量
      if (($i % 200000) == 0) {
         // 来 MB 为单位获取内存使用量
         echo round(memory_get_usage() / 1024 / 1024, 2) . ' MB'. PHP_EOL;
      }
   }
   return $valuesArray;
}
$myValues = getValues(); // 一旦我们调用函数将会在这里创建数组
foreach ($myValues as $value) {}
```
前面例子发生的情况是这个脚本的内存消耗和输出：
```txt
0.34 MB
8.35 MB
16.35 MB
32.35 MB
```
这意味着我们的几行脚本消耗了超过30MB的内存，每次你你添加一个元素到$valuesArray数组中，都会增加他在内存中的大小。

让我们使用yield同样的例子:
```php
<?php
function getValues() {
   // 获取内存使用数据
   echo round(memory_get_usage() / 1024 / 1024, 2) . ' MB' . PHP_EOL;
   for ($i = 1; $i < 800000; $i++) {
      yield $i;
      // 做性能分析，因此可测量内存使用率
      if (($i % 200000) == 0) {
         // 内存使用以 MB 为单位
         echo round(memory_get_usage() / 1024 / 1024, 2) . ' MB'. PHP_EOL;
      }
   }
}
$myValues = getValues(); // 在循环之前都不会有动作
foreach ($myValues as $value) {} // 开始生成数据
```
这个脚本的输出令人惊讶：
```txt
0.34 MB
0.34 MB
0.34 MB
0.34 MB
```
这不意味着你从return表达式迁移到yield，但如果你在应用中创建会导致服务器上内存出问题的巨大数组，则yield更加适合你的情况。

# 什么是"yield"选项

这里有很多yield的选项， 我将强调他们中的几个：

a. 使用yield， 你也可以使用return。
```php
function getValues() {
   yield 'value';
   return 'returnValue';
}
$values = getValues();
foreach ($values as $value) {}
echo $values->getReturn(); // 'returnValue'
```
b. 返回键值对：
```php
function getValues() {
   yield 'key' => 'value';
}
$values = getValues();
foreach ($values as $key => $value) {
   echo $key . ' => ' . $value;
}
```

# 结论

这个主题的主要原因是为了明确yield和return特别是在内存使用方面的区别，使用一些例子是因为我发现他对任何开发人员思考真的很重要。

# 具体情景

http://www.php100.com/9/20/19693.html

laravel ORM中的cursor游标cursor方法允许你使用游标遍历数据库，它只执行一次查询。处理大量的数据时,可以大大减少内存的使用量通过对比cursor与get方法，查找到了其中的一点不同，get方法是直接fetchAll返回所有数据，cursor方法是使用yield构建了一个生成器，逐步返回fetch的数据
PDO mysql中的查询
一般查询如下：
```php
$sql = "select * from `user` limit 100000000";
$stat = $pdo->query($sql);
$data = $stat->fetchAll();  //mysql buffered直接全部返回给php

var_dump($data);
```
使用yield如下：
```php
function get(){
    $sql = "select * from `user` limit 100000000";
    $stat = $pdo->query($sql);
    while ($row = $stat->fetch()) { //一个一个地返回给php
        yield $row;
    }
}

foreach (get() as $row) {
    var_dump($row);
}
```
**PDO参数**

这个PDO参数，侧面说明了，如果这个参数为ture,MySQL驱动程序将使用MySQL API的缓冲版本。

注意，这个参数只对Mysql起作用，如果要编写可移植的sql代码，请使用fetchAll
查看php手册MYSQL_ATTR_USE_BUFFERED_QUERY
在使用了yield的情况下，内存使用量的确减少了，但是在逐渐使用yield中，内存会逐渐增大，这样的情况与以下函数不一致，下面这个函数的内存使用量将会不变
```php
function getNum() {
    for($i=0; $i < 100000000; $i++) {
        yield $i;
    }
}
```

# 结论

在使用fetch、fetchAll之前，查找数据结果集还存在于mysql的缓冲区内，而每次yield和fetch配合，才取回一条结果存入内存，这就能解释，为什么内存使用量会逐步增大

而上面的这个getNum函数之所以内存使用量不变，是因为，每次只返回一个数字