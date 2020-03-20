---
title: 高性能千万级定时任务管理服务forsun laravel插件使用详解
date: 2018-03-10 16:58:48
tags: forsun
---

forsun高性能高精度定时服务，轻松管理千万级定时任务。

定时服务项目地址：https://github.com/snower/forsun

laravel插件项目地址： https://github.com/snower/forsun-laravel

* 轻松支持千万级定时任务调度。
* 定时任务触发推送到Queue，轻松支持跨机器和共性能分布式。
* 支持任务到期触发command、Job、Shell、Http和Event。
* 支持驱动原生Laravel Schedule运行。
* 支持创建延时任务和定时到期任务，和原生Laravel Schedule保持相同接口，轻松使用。

<!-- more -->

## 背景 ##

在实际项目中，存在大量需要定时或是延时触发的任务，比如电商中，延时需要检查订单是否支付成功，是否配送成功，定时给用户推送提醒等等，常规做法是用 crontab 每分钟扫码数据看是否到达时间，繁琐且扩展性伸缩性较差。

使用 forsun 服务，可以简单的针对每一个订单创建一个定时任务，配合异步队列，可以轻松实现扩展性伸缩性，Apache Thrift 的编程接口也可以很容易的和 celery、laravel 配合。 

其他场景下，比如失败延时重试，使用 forsun 定时服务也可以很简单就可以实现。

## 安装 ##

* 安装启动forsun服务，详情请看 [https://blog.snower.me/2018/03/05/forsun-introduction/](https://blog.snower.me/2018/03/05/forsun-introduction/)。
* composer安装forsun-laravel。

```bash
composer require "snower/forsun-laravel"
```

## 配置 ##

* 在 config/app.php 注册 ServiceProvider 和 Facade

```php
'providers' => [
    // ...
    Snower\LaravelForsun\ServiceProvider::class,
],
'aliases' => [
    // ...
    'Forsun' => Snower\LaravelForsun\Facade::class,
],
```

* 创建配置文件

```bash
php artisan vendor:publish --provider="Snower\LaravelForsun\ServiceProvider"
```

* 修改应用根目录下的 config/forsun.php 中对应的参数即可。

## 使用 ##

### 定义调度

* Artisan 命令调度。

```php

//不指定name是自动生成
Forsun::plan()->command('emails:send --force')->daily();

//指定name
Forsun::plan('email')->command(EmailsCommand::class, ['--force'])->daily();
```

* 队列任务调度

```php
Forsun::plan()->job(new Heartbeat)->everyFiveMinutes();
```

* Shell 命令调度

```php
Forsun::plan()->exec('node /home/forge/script.js')->daily();
```

* Event事件调度

```php
Forsun::plan()->fire('testevent', [])->everyMinute();
```

* Http事件调度

```php
Forsun::plan()->http('http://www.baidu.com')->everyMinute();
```

注意：

* 每个任务只能设置一次调度频率。
* 不支持任务输出、任务钩子及维护模式。
* Forsun::plan是不指定任务名时自动生成，每个任务名必须唯一，相同任务名重复定义将会自动覆盖。

### 移除调度

```php
$plan = Forsun::plan()->command('emails:send --force')->daily();
$plan->remove();

$plan = Forsun::plan()->command('emails:send --force')->daily();
$plan_name = $plan->getName();
Forsun::remove($plan_name);
```

### 调度频率设置

| 方法 | 描述 |
| ---------- | --- |
| ->hourly(); | 每小时运行 |
| ->hourlyAt(17); | 每小时的第 17 分钟执行一次任务 |
| ->daily(); | 每天午夜执行一次任务 |
| ->dailyAt('13:00'); | 每天的 13:00 执行一次任务 |
| ->monthly(); | 每月执行一次任务 |
| ->monthlyOn(4, '15:00'); | 在每个月的第四天的 15:00 执行一次任务 |
| ->everyMinute(); | 每分钟执行一次任务 |
| ->everyFiveMinutes(); | 每五分钟执行一次任务 |
| ->everyTenMinutes(); | 每十分钟执行一次任务 |
| ->everyFifteenMinutes(); | 每十五分钟执行一次任务 |
| ->everyThirtyMinutes(); | 每半小时执行一次任务 |
| ->at(strtoetime("2018-03-05 12:32:12")); | 在指定时间2018-03-05 12:32:12运行一次 |
| ->interval(10); | 从当前时间开始计算每10秒运行一次 |
| ->later(5); | 从当前时间开始计算稍后5秒运行一次 |
| ->delay(30); | 从当前时间开始计算稍后30秒运行一次 |

需要复杂定时控制建议生成多个定时任务或是在处理器中再次发起定时任务计划更简便同时也性能更高。

调度器应该尽可能使用Event或是Job通过Queue Work可以更高性能运行。

### 驱动原生Laravel Schedule运行

```bash
#注册
php artisan forsun:schedule:register

#取消注册
php artisan forsun:schedule:unregister
```
