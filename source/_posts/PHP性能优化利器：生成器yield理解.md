---
layout: title
title: PHP性能优化利器：生成器yield理解
date: 2020-01-14 11:15:56
tags: PHP
top: true
---
思考并回答以下问题：
* 使用yield还需要再循环一次，怎么理解？
* yield和两个foreach在一起。怎么理解？


<!--more-->

如果是做Python或者C#的小伙伴，对于生成器应该不陌生。但很多PHP开发者或许都不知道生成器这个功能，可能是因为生成器是PHP5.5.0才引入的功能，也可以是生成器作用不是很明显。但是，生成器功能的确非常有用。

# 优点

* 生成器会对PHP应用的性能有非常大的影响。
* PHP代码运行时节省大量的内存。
* 比较适合计算大量的数据。

那么，这些神奇的功能究竟是如何做到的？我们先来举个例子。

# 概念引入

首先，放下生成器概念的包袱，来看一个简单的PHP函数：

```php
function createRange($number)
{
    $data = [];

    for($i = 0; $i < $number; $i++)
    {
        $data[] = time();
    }

    return $data;
}
```
这是一个非常常见的PHP函数，我们在处理一些数组的时候经常会使用。这里的代码也非常简单：

* 1.我们创建一个函数。
* 2.函数内包含一个for循环，我们循环的把当前时间放到$data里面。
* 3.for循环执行完毕，把$data返回出去。

再写一个函数，把这个函数的返回值循环打印出来：

```php
$result = createRange(10); // 这里调用上面我们创建的函数

foreach($result as $value)
{
    sleep(1); // 这里停顿1秒，我们后续有用
    echo $value.'<br/>';
}
```
我们在浏览器里面看一下运行结果：

{% asset_img 1.png %}

这里非常完美，没有任何问题。（当然sleep(1)效果你们看不出来）

# 思考一个问题

我们注意到，在调用函数createRange的时候给$number的传值是10，一个很小的数字。假设，现在传递一个值<span style="color:red">10000000（1000万）</span>。

那么，在函数 createRange 里面，for循环就需要执行1000万次。且有1000万个值被放到 $data 里面，而$data数组在是被放在内存内。所以，在调用函数时候会占用大量内存。

这里，生成器就可以大显身手了。

# 创建生成器

我们直接修改代码，你们注意观察：

```php
function createRange($number)
{
    for($i = 0; $i < $number; $i++)
    {
        yield time();
    }
}
```
看下这段和刚刚很像的代码，我们删除了数组$data，而且也没有返回任何内容，而是在time()之前使用了一个关键字yield。

# 使用生成器

我们再运行一下第二段代码：

```php
$result = createRange(10); // 这里调用上面我们创建的函数

foreach($result as $value)
{
    sleep(1);
    echo $value.'<br />';
}
```
我们奇迹般的发现了，输出的值和第一次没有使用生成器的不一样。这里的值（时间戳）中间间隔了1秒。

这里的间隔一秒其实就是sleep(1)造成的后果。但是为什么第一次没有间隔？那是因为：

* 未使用生成器时：createRange函数内的for循环结果被很快放到$data中，并且立即返回。所以，foreach循环的是一个固定的数组。
* 使用生成器时：createRange的值不是一次性快速生成，而是依赖于foreach循环。foreach循环一次，for执行一次。

到这里，你应该对生成器有点儿头绪。

# 深入理解生成器

**代码剖析**

下面我们来对于刚刚的代码进行剖析。

```php
function createRange($number)
{
    for($i = 0; $i < $number; $i++)
    {
        yield time();
    }
}

$result = createRange(10); // 这里调用上面我们创建的函数

foreach($result as $value)
{
    sleep(1);
    echo $value.'<br />';
}
```
我们来还原一下代码执行过程。

* 1.首先调用createRange函数，传入参数10，但是for值执行了一次然后停止了，并且告诉foreach第一次循环可以用的值。
* 2.foreach开始对$result循环，进来首先sleep(1)，然后开始使用for给的一个值执行输出。
* 3.foreach准备第二次循环，开始第二次循环之前，它向for循环又请求了一次。
* 4.for循环于是又执行了一次，将生成的时间戳告诉foreach.
* 5.foreach拿到第二个值，并且输出。由于foreach中sleep(1)，所以，for循环延迟了1秒生成当前时间。

所以，整个代码执行中，始终只有一个记录值参与循环，内存中也只有一条信息。

无论开始传入的$number有多大，由于并不会立即生成所有结果集，所以内存始终是一条循环的值。

**概念理解**

到这里，你应该已经大概理解什么是生成器了。下面我们来说下生成器原理。

首先明确一个概念：**生成器yield关键字不是返回值，他的专业术语叫产出值，只是生成一个值**

那么代码中 foreach 循环的是什么？其实是PHP在使用生成器的时候，会返回一个 Generator 类的对象。 foreach 可以对该对象进行迭代，每一次迭代，PHP会通过 Generator 实例计算出下一次需要迭代的值。这样 foreach 就知道下一次需要迭代的值了。

而且，在运行中 for 循环执行后，会立即停止。等待 foreach 下次循环时候再次和  for  索要下次的值的时候，循环才会再执行一次，然后立即再次停止。直到不满足条件不执行结束。

# 实际开发应用

很多PHP开发者不了解生成器，其实主要是不了解应用领域。那么，生成器在实际开发中有哪些应用？

**读取超大文件**

PHP开发很多时候都要读取大文件，比如csv文件、text文件，或者一些日志文件。这些文件如果很大，比如5个G。这时，直接一次性把所有的内容读取到内存中计算不太现实。

这里生成器就可以派上用场啦。简单看个例子：读取text文件


我们创建一个text文本文档，并在其中输入几行文字，示范读取。

```php
<?php
header("content-type:text/html;charset=utf-8");
function readTxt()
{
    # code...
    $handle = fopen("./test.txt", 'rb');

    while (feof($handle)===false) {
        # code...
        yield fgets($handle);
    }

    fclose($handle);
}

foreach (readTxt() as $key => $value) {
    # code...
    echo $value.'<br />';
}
```

通过上图的输出结果我们可以看出代码完全正常。

但是，背后的代码执行规则却一点儿也不一样。使用生成器读取文件，第一次读取了第一行，第二次读取了第二行，以此类推，每次被加载到内存中的文字只有一行，大大的减小了内存的使用。

这样，即使读取上G的文本也不用担心，完全可以像读取很小文件一样编写代码。

 百万级别的访问量

yield生成器是php5.5之后出现的，yield提供了一种更容易的方法来实现简单的迭代对象，相比较定义类实现 Iterator 接口的方式，性能开销和复杂性大大降低。

yield生成器允许你 在 foreach 代码块中写代码来迭代一组数据而不需要在内存中创建一个数组。

使用示例：
```php
/** 
 * 计算平方数列 
 * @param $start 
 * @param $stop 
 * @return Generator 
 */  
function squares($start, $stop) 
{  
    if ($start < $stop) 
    {  
        for ($i = $start; $i <= $stop; $i++) 
        {  
            yield $i => $i * $i;  
        }  
    }  
    else 
    {  
        for ($i = $start; $i >= $stop; $i--) 
        {  
            yield $i => $i * $i; //迭代生成数组： 键=》值  
        }  
    }  
}  
foreach (squares(3, 15) as $n => $square) 
{  
    echo $n . ‘squared is‘ . $square . ‘<br>‘;  
}  
```
输出：  
```txt
    3 squared is 9  
    4 squared is 16  
    5 squared is 25  
    ...  
```
示例2：
 
```php
//对某一数组进行加权处理  
$numbers = array(‘nike‘ => 200, ‘jordan‘ => 500, ‘adiads‘ => 800);  
  
//通常方法，如果是百万级别的访问量，这种方法会占用极大内存  
function rand_weight($numbers)  
{  
    $total = 0;  
    foreach ($numbers as $number => $weight) 
    {  
        $total += $weight;  
        $distribution[$number] = $total;  
    }  
    $rand = mt_rand(0, $total-1);  
  
    foreach ($distribution as $num => $weight) 
    {  
        if ($rand < $weight) return $num;  
    }  
}  
  
//改用yield生成器  
function mt_rand_weight($numbers) 
{  
    $total = 0;  
    foreach ($numbers as $number => $weight) 
    {  
        $total += $weight;  
        yield $number => $total;  
    }  
}  
  
function mt_rand_generator($numbers)  
{  
    $total = array_sum($numbers);  
    $rand = mt_rand(0, $total -1);  

    foreach (mt_rand_weight($numbers) as $num => $weight) 
    {  
        if ($rand < $weight) return $num;  
    }  
}  
```

# Laravel + ElasticSearch

```php
/**
 * 迁移用户数据到es
 * @param $container
 * @param $logger
 * @param $type
 */
public static function syncUsersToEs($container, $logger, $type)
{
    list($builder, $currentTime) = [
        $container->get(ClientBuilderFactory::class)->create(),
        time(),
    ];

    $client = $builder->setHosts([env('ES_HOSTS', '')])->build();

    $esFun = function ($data, $client) use ($type) {
        $list = [];
        foreach ($data as $k => $va) 
        {
            if (count($list) >= 500) 
            {
                $client->bulk([
                    'index' => Users::ES_INDEX_NAME,
                    'type' => 'users',
                    'body' => $list
                ]);
                $list = [];
            } else 
            {
                switch ($type) 
                {
                    // 更新基础数据字段的值
                    case 'update':
                        array_push(
                            $list,
                            [
                                'update' =>
                                    [
                                        '_id' => (int)$va['id'],
                                        '_index' => Users::ES_INDEX_NAME,
                                        '_type' => '_doc',
                                    ]
                            ],
                            ['doc' => $va]
                        );
                        break;

                    // 把对应的新增用户注入es
                    case 'create':
                        array_push(
                            $list,
                            [
                                'create' =>
                                    [
                                        '_id' => (int)$va['id'],
                                        '_index' => Users::ES_INDEX_NAME,
                                        '_type' => '_doc',
                                    ]
                            ],
                            $va
                        );
                        break;
                }
            }
        }

        if (!empty($list)) 
        {
            $res = $client->bulk(['index' => Videos::ES_INDEX_NAME, 'type' => 'sv_videos', 'body' => $list]);
        }

        unset($list);
    };

    switch ($type) 
    {
        // 添加新用户到es中
        case 'create':
            Users::query()
                ->select(['id', 'name', 'avatar', 'gender', 'mobile'])
                ->where([
                    ['created_at', '>=', date('Y-m-d H:i:s', $currentTime - 3600)]
                ])
                ->chunkById(1000, function ($results) use ($client, $esFun) {
                    // 迭代器数据
                    $data = function () use ($results) {
                        foreach ($results->toArray() as $v) {
                            yield [
                                'id' => (int)$v['id'],
                                'name' => (string)$v['name'],
                                'avatar' => (string)$v['avatar'],
                                'gender' => (int)$v['gender'],
                                'mobile' => (string)$v['mobile'],
                            ];
                        }
                    };

                    $esFun($data(), $client);
                });
            break;

        // 去更新用户搜索的基础数据
        case 'update':
            Users::query()
                ->select(['id', 'name', 'avatar', 'gender', 'mobile'])
                ->where([
                    ['updated_at', '>=', date('Y-m-d H:i:s', $currentTime - 3600)]
                ])
                ->chunkById(1000, function ($results) use ($client, $esFun) {
                    // 迭代器数据
                    $data = function () use ($results) {
                        foreach ($results->toArray() as $v) {
                            yield [
                                'id' => (int)$v['id'],
                                'name' => (string)$v['name'],
                                'avatar' => (string)$v['avatar'],
                                'gender' => (int)$v['gender'],
                                'mobile' => (string)$v['mobile'],
                            ];
                        }
                    };

                    $esFun($data(), $client);
                });
            break;
    }
}

```