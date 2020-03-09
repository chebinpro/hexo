---
layout: title
title: 结合Laravel Facade看外观模式怎么用？
date: 2020-01-14 11:58:47
tags: php
---
思考并回答以下问题：
* 什么是外观模式？

<!--more-->

当独立子系统的开发完成后，如果两个系统是客户--供应商关系，也就是说其中一个子系统（客户）需要使用另一个子系统（供应商）提供的服务时，我们可以通过外观模式来对客户子系统隐藏调用供应商系统的复杂性，客户端只需通过facade调用相应的服务而无需涉及供应商具体的服务调用方法。下面通过laravel中的facade来说明:
　　
在laravel中，我们经常通过facade来实现全局调用某个方法而不需要实例化一个对象。通过查看源码可以知道，所有的Facade都是继承自同一个抽象父类Illuminate\Support\Facades\Facade。

当我们调用Auth::guard('customer')来返回一个guard实例的时候，只需要在调用这个方法的所在脚本写上use Illuminate\Support\Facades\Auth;就可以了，而当我们去查看Illuminate\Support\Facades\Auth的具体定义的时候，我们就会发现，这种调用都是基于php的魔术方法——\__callstatic()。

> \vendor\laravel\framework\src\Illuminate\Support\Facades.php

```php
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();

    if (! $instance) 
    {
        throw new RuntimeException('A facade root has not been set.');
    }
    return $instance->$method(...$args);
}
```

从代码中可以看出，最关键的一句就是$instance=static::getFacadeRoot();因为调用的方法和参数我们都能通过\__callstatic获取，关键是我们如何获取到一个能执行这个方法的正确的对象，让我们继续往下扒。

```php
public static function getFacadeRoot()
{
	return static::resolveFacadeInstance(static::getFacadeAccessor());
}
```
原来底层是你，resolveFacadeInstance(static::getFacadeAccessor())不管，继续扒。
```php
protected static function getFacadeAccessor()
{
	throw new RuntimeException('Facade does not implement getFacadeAccessor method.');
}
```
static::getFacadeAccessor()为什么直接抛出错误，搞错了？不对，刚才好像在哪见过？？没错，每个Facade都会对这个方法进行重写，如果没有重写就会抛出错误，例如在Illuminate\Support\Facades\Auth中的实现是：
```php
protected static function getFacadeAccessor()
{
	return 'auth';
}
```
好了，看来接下来这个才是最核心的resolveFacadeInstance($name)
```php
 protected static function resolveFacadeInstance($name)
{
    // 判断传入参数是否是对象，是则直接返回，我猜laravel框架告诉我们在定义自己的facade的时候可以直接在getFacadeAccessor返回一个对象
    if (is_object($name)) 
    {
        return $name;
    }
   	// 不是对象的话,判断是否已经实例化过这个对象，是的话就把之前实例化后保存的对象拿去使
    if (isset(static::$resolvedInstance[$name])) 
    {
 		return static::$resolvedInstance[$name];
    }
	// 没有?!那只能返回新的对象并保存到$resolvedInstance中方便下次用了
    return static::$resolvedInstance[$name] = static::$app[$name];
}
```
这里是$app是laravel的ioc容器。

通过上面的例子可以看出，laravel通过外观模式对外提供auth服务，向客户端隐藏了具体的操作，实际底层是调用了某个具体的执行者来提供该项服务，这样客户端就无需知道相关细节，就可以使用开箱即用的服务。